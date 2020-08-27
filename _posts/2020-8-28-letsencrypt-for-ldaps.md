---
layout: post
title: LetsEncrypt for LDAPS
---

LDAPS is the secure form of the LDAP protocol and it relies on signed certificates for validation. This is a brain dump on an example of how to automatically deploy and renew signed certificates using ACME and use it for LDAPS.

# Certificate Authority (CA) vs Self-Signed
People are divided as to whether to issue a self-signed certificate and manage that trust to your internal network versus signing LDAPS with a valid CA.

For most HTTPS traffic people can agree to use CA signed certificates because its often used using end-user web browsers which have a good source of trust. When it comes to LDAPS many people opt for using self signed because usually the only things connecting over LDAPS is servers they have full control of so it is easy to manage the chain of trust.

Reasons why people may want to avoid CA signed certificates is cost and expiry dates. LetsEncrypt offers a free alternative to other Certificate Authorities however it enforces a strict 90 day lifetime of its certificates.

I personally prefer a CA signed certificate because servers that are kept up to date should trust it implicitly. If your entire server network all have up to date root CA trust then you don't have to worry about managing the chain of trust.

# Dealing with shorter certificate durations in LDAP
## Automated Certificate Management Environment (ACME)
ACME is a way we can automate certificate renewals. Certbot is a ACME implementation we can run on Linux. Out of the box it will use HTTP validation. This probably won't work for us because the ACME server would need to route HTTP to our LDAP server which is probably on a private address. Instead we will use DNS based validation.

### DNS Validation
In DNS based validation the ACME server generates a random passphrase that we then need to set as a DNS text record to prove we have ownership of the domain.

- The name of this DNS record will always be `_acme-challenge.<YOUR_DOMAIN>`.
- This record will need to be re-set every 2 months when we renew the certificate.
- The validation DOES follow CNAME records. It is a good idea to CNAME to a seperate DNS zone you own that supports dynamic updates.

Certbot has a bunch of different plugins for different DNS providers including Dynamic DNS if you self-host.

# Staggering Updates
For example, if you have two multi-master LDAP servers (ldap1 and ldap2) you may want to have two seperate certificates for the purposes of staggering renewals.

The first certificate would still have ldap.example.com as your common name and you wan add on ldap1.example.com for LDAPS replication. If you issue this certificate first and wait one month before issuing a certificate for ldap2 then each certificate will have a different expiry date.

This is an example of issuing one of these certificates using AWS Route53 as domain based validation.
```
/usr/local/bin/certbot certonly --dns-route53 -d ldap.example.com -d ldap1.example.com
```

If anything goes wrong when ldap1 automatically renews its certificate then you still have the second ldap server to rely on until you fix the issue.

## Intermediate Certificates
There may be scenarios where your LDAPS certificate is still not trusted even by a public CA. In this case you will need to update the CA certificates on the server with the intermediate certificate from the CA that was used to sign your certificate.

Intermediate certificates from the CA do not change as frequently as your ACME certificates. LetsEncrypt appear to have intermediate certificates for five years. Whenever intermediate certificates update you will still need to properly check your environment trusts the new certificate.

# HAProxy
389 Directory Service uses certutil to configure certificates for LDAPS. This has its benefits however its more cumbersome and uncommon which makes automatic certificate renewal more difficult. HAProxy give us a easy way to terminate the TLS for LDAPS which also lets us swap out certificates without restarting the whole LDAP service.

HAProxy requires our certificate file to have both the private and public keys in one.
```
cat /etc/letsencrypt/live/ldap.example.com/fullchain.pem /etc/letsencrypt/live/ldap.example.com/privkey.pem >> /etc/letsencrypt/live/ldap.example.com/combo.pem
```

Here is a simple HAProxy config to terminate TLS for LDAPS. The `ldap_front` is the TLS termination and the `ldap_back` connects to your LDAP service listinging on localhost.
```
frontend ldap_front
    bind 0.0.0.0:636 ssl crt /etc/letsencrypt/live/ldap.example.com/combo.pem
    mode tcp
    option tcplog
    default_backend ldap_back

backend ldap_back
    mode tcp
    balance roundrobin
    option ldap-check
    server ldap1 127.0.0.1:389
```

We still need to be able to renew the certificate automatically. This is a cron that runs at a random point past the hour every 12 hours to check for renewal. When renewing it calls a post hook to merge the new certificates into the new combo.pem and restarts HAProxy.
```
0 */12 * * * root perl -e 'sleep int(rand(43200))' && /usr/local/bin/certbot renew --post-hook "cat /etc/letsencrypt/live/ldap.example.com/privkey.pem /etc/letsencrypt/live/ldap.example.com/fullchain.pem > /etc/letsencrypt/live/ldap.example.com/combo.pem && systemctl restart haproxy"
```

---
layout: post
title: EduRoam Chargeable User Identity
---

Last year I did a presentation on EduRoam Analytics. Here is a brief summary of the presentation slides.

# Overview
EduRoam is an enormous network which can be difficult to track analytics and respond to security incidents in a timely manner.

There are things we can do today to give better visibility over the devices connected to EduRoam but also things we can work towards for a collaborative effort to increase the security posture for EduRoam across Australia.

# Problem
How can we identify when an EduRoam account is compromised?
- The outer identity of a RADIUS packet is usually anonymous.
- The source IP of the RADIUS server is lost when going through the EduRoam RADIUS proxy.

# How can we be a friendly roaming location?
We can't fix these problems easily because of how EduRoam is distributed. We can make our own infrastructure friendly where it returns some metadata.

## Operator-Name
The Operator-Name attribute is defined in [RFC5580] as a means of unique identification of the access site. It can help with incident response when a user of your organisation is authenticating from another site.
https://tools.ietf.org/html/rfc7593#section-5.2

The RADIUS Operator-Name is an attribute that is a fully qualified domain name added to the access and accounting request messages. The domain name is the realm of the owner of the EduRoam infrastructure at that location.

Now when an organisations RADIUS server receives a proxied message it will have the realm of the location the user is roaming from.

## Chargeable User Identity
The CUI attribute serves as an alias to the user's real identity, representing a chargeable identity as defined and provided by the home network as a supplemental or alternative information to User-Name. This is not necessarily the User-Name but is some kind of identifier that can be mapped to a single user. This helps with organisations incident response if they have a malicious actor on their EduRoam infrastructure tha
t is using another IDP to authenticate to EduRoam.
https://tools.ietf.org/html/rfc4372#section-2.1

The Chargeable User Identity is mapped with the Operator-Name so that different organisations can collaborate during security incidents.

## Can we alert on this?
Can we alert if a single user is in two locations at the same time? We have some problems:
- Not everyone has Operator-Name set
- The Operator-Name is a realm. Not anything as accurate as longitude or latitude

If we were to collaborate with our nearest neighbours then we could identify the local realms that a user may realistically roam to in a given day. Anything other then those realms would be considered "Non-Local". We could report back to users if they are using EduRoam from a "Local" spot or somewhere else.

# How we can give our users security using the Calling station Id
The Calling station Id is the users MAC address. The formatting of the Calling station Id is different with different RADIUS servers. This is far from a perfect measure but could be an indicator for users to identify untrusted devices in real time.

## Trusted devices can look rogue
- Giving a laptop to a friend
- Changing your password but not updating the WiFi config on your phone

# What is on our outer tunnel that can be sniffed
RADIUS is designed to operate over UDP. The tunneled TLS is there for a good reason. We need to be aware of what an attacker could get from the network if this isn't using TLS.
- Calling-station-id (MAC Address)
- Outer identity, realm, users not setting anonymous identities

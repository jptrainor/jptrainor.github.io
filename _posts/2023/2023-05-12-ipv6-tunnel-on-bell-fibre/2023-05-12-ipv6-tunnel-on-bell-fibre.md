---
layout: post
title:  "OpenWrt IPV6 Tunnel Configuration"
date:   2023-05-11
categories: IPV6
---

Bell Canada's residential fiber internet service doesn't support
IPV6. An IPV6-over-IPV4 tunnel is a solution, and [Hurricane
Electric](https://www.he.net/) provide a free [IPV6 over IPV4 tunnel
service](https://tunnelbroker.net).

# Not so simple

Should be simple, but all my attempts failed and left me
frustrated. The tunnel would send but no data was ever received, and
it behaved the same regardless of what platform I tested with. Even
pinging my router's public IP address proved unreliable. Others
[reported](https://forum.bell.ca/t5/Internet/Is-ipv6-tunneling-possible-with-Bell/m-p/12415)
similar problems, and are similarly
[frustrated](https://forum.bell.ca/t5/Internet/Gigahub-and-IPv6/m-p/10489)
by this.

IPV6-over-IPV4 is implemented using [IP protocol
41](https://simple.wikipedia.org/wiki/Protocol_41). A call to Bell's
tech support confirmed that they don't block IP protocol 41 on their
network, indeed they said they don't block anything. In theory, then,
it should work. And, in theory at least, if their network isn't
blocking anything then what must be blocking the traffic is the [Home
Hub 4000
router](https://support.bell.ca/internet/products/home-hub-4000-modem),
a.k.a. the "Gigahub", that Bell's fiber customer have no choice but to
use.

So how to get the IPV6 tunnel traffic through the Gigahub? It has
[DMZ](https://en.wikipedia.org/wiki/DMZ_(computing)) support, even an
"Advanced DMZ" mode that claims to completely bypass the router's
firewall. None of that helped. Indeed, it seemed even more
unreliable. There are many reports of Gigahub DMZ being
[buggy](https://forum.bell.ca/t5/Internet/Advanced-DMZ/m-p/10971) on
the Gighub and that was my experience.

# The solution

What I found works to to pass IPV6 tunnel traffic on both Bell's
network, and with the Gigahub router, is [PPPoE
passthrough](https://forum.bell.ca/t5/Internet/Using-PPPoE-and-DMZ-Advanced-DMZ-for-Bridge-Mode-use-of-3rd/m-p/705).

The solution I settled on was a dedicated OpenWrt router that
configures a PPPoE connection to transit the IPV4 traffic (the only
traffic that Bell supports), and a Hurricane Electric IPV6 tunnel,
also running on the OpenWrt router, to support the IPV6
traffic. Clients of this router all get both IPV4 and IPV6 internet
access.

The PPPoE connection gets a new public IP address, different from the
GigaHub's own IP address. This new public address is pingable, and
IPV6 Hurricane Electric tunnel traffic passes through the Gigahub
without problem.

Note, this additional, dedicated, IPV6 router doesn't prevent the rest
of the users on my home network from routing their normal IPV4 traffic
directly thorugh the GigaHub.

# OpenWrt router configuration

Starting from a fresh install, with SSH access, run the following
configuration script on your OpenWrt router:




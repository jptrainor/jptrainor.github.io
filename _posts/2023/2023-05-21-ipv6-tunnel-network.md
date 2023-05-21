---
layout: post
title:  "IPV6 Tunnel Local Network Configuration"
date:   2023-05-16
categories: IPV6
---

This is a followup to my post about configuring a [Hurricane Electric
IPv6 tunnel](https://tunnelbroker.net/) On an [OpenWrt IPv6
Tunnel](./ipv6-tunnel-on-bell-fibre.html) to work around lack of IPv6
support by my Bell Fiber internet service. It describes how I
integrated this tunnel router into my local home network.

There are three important constraints that determine how and why I did this the way I did it:

1. Hurricane Electric's tunnel service is, in their
[words](https://tunnelbroker.net): "... oriented towards developers
and experimenters that want a stable tunnel platform." And that is
what I'm using it for - development. I don't use full time. I connect
to it ass needed when I want specifically want to test or access
something that requires IPv6.

2. My internet service is very fast 1.5 Gbps fiber. It don't want
routine network traffic transitting the IPv6 tunnel needlessly because
it just get in the way and slow down everything for no advantage. It
should be totally out of the loop for the rest of the users of my home
network.

3. I have no interest in getting a new (expensive) primary router to
take over all the routing on my network. Most of the traffic has to be
routed normally by the [Bell
GigaHub](https://support.bell.ca/internet/products/home-hub-4000-modem). The
IPv6 doesn't have to be fast to meet my needs, it just has to be easy
to use and reliable.

With that in mind, here, unsurprisingly, is the network topology:

![Local network topology](/assets/images/2023/2023-05-21-ipv6-tunnel-network/localNetworkTopology.png)

# Dedicated DHCP/DNS on primary LAN 

All the normal (IPv4 only) network traffic uses the 192.168.1.0/24 LAN
that is routed by the Bell GigaHub.  There's nothing unusual about
anything other than that I don't use the DHCP service provided by the
GigaHub.

For DHCP, I already had dedicated DHCP / DNS server running on the
network. It is a modest DHCP server (but nonetheless much more capable
that that provided by the GigaHub). It's nothing more than OpenWRT
running on an old router and configured to to only DHCP. It provides
routing information to clients that configures the GitHub as the
default route (just as the GitHub itself would do if it was acting as
the DHCP server). This is optional, but you'll see later that is
useful for configuring routing on clients of the 192.1681.0/24
network.

Note that this DHCP server only serves clients on the IPV4-only
192.168.1.0/24 local area network. It has no role to play in providing
IPv6 support. Clients on this LAN only have IPv4 internet access.

# IPv6 support on sub-LAN.

The IPv6 tunnel router has a PPPoE WAN connection with a public IPv4
address that is different than the GigaHub WAN's IPv4 address. It has a LAN sub-network that I assign IPv4 172.16.1.0/24 to distinguish
it from the prmary LAN.

This router runs the Hurricane Electric IPv6 tunnel, and it runs its
own DHCP and DHCPv6 service for clients of its LAN (normal OpenWrt
router configuration). LAN clients get an IPv4 172.16.1.0/24 address
and an IPv6 address derived from the tunnel's IPv6 prefix. All clients
of the tunnel router's LAN have both IPV4 and IPV6 access via the 6in4
tunnel.


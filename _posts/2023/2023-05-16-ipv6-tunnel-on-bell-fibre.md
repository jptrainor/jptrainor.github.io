---
layout: post
title:  "OpenWrt IPV6 Tunnel Configuration"
date:   2023-05-16
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

The configuration steps below are all performed using the router's
command line interface. You have to ssh into the router.

Connect your client computer directly to the router's ethernet LAN
interface, configure your client computer's ethernet interface to use
DHCP. The router will provide DHCP configuration to your computer, and
you should then be able access the router using:

```
ssh root@openwrt.lan
```

All the steps below can be performed using the router's
[LuCi](https://openwrt.org/docs/guide-user/luci/start) web interface
as well, but that's not documented here.

# Configure the wan (IPV4) network interface

The router's ethernet WAN port should be connected to the same network
as the PPPoE server (i.e. the Bell Gigahub, for the purposes of this
blog).

Starting from a fresh install, log into your router using ssh and run
the following configuration script on your OpenWrt router.

```
uci set network.wan.proto=pppoe
uci set network.wan.username="your-isp-username"
uci set network.wan.password="your-isp-password"
uci commit network
ifup wan
```

Wait for the network interface to come up and verify that you have an
internet connection. You'll need your public IP4 address later to
create the [Hurricane Electric IPV6 tunnel](https://tunnelbroker.net).

Get your WAN IP4 address from an external service:

```
$ wget -4qO- api.ipify.org; echo
123.456.789.123
```

Get your WAN IPV4 address from OpenWrt's network status:

```
$ ifstatus wan |  jsonfilter -e '@["ipv4-address"][0].address'
123.456.789.123
```

The two should match. If they don't, or if one doesn't work, then
something is wrong.

# Install the tunnel software

Install the OpenWRT IPV6 tunnel software, and reboot the router.

```
opkg update
opkg install 6in4
reboot
```

# Create Hurricane Electric IPV6 tunnel.

Visit [tunnelbroker.net](https://tunnelbroker.net) and create a new
IPV6 tunnel. You'll need your router's public IPV4 address (the WAN
address from above) to setup the tunnel.

After it's setup, you'll need the need the Hurricane Electric tunnel
information below to configure the OpenWrt IPV6 wan interface.

![Hurricane Electric UI tunnel details](/assets/images/2023/2023-05-16-ipv6-tunnel-on-bell-fibre/TunnelDetail-IPv6Tunnel.png)
![Hurrican Electric UI tunnel advanced](/assets/images/2023/2023-05-16-ipv6-tunnel-on-bell-fibre/TunnelDetail-IPv6Advanced.png)

# Configure the wan6 (IPV6) network interface

Finally, configure the IPV6 wan interface. Substitute the "henet-...."
place holders using the matching values from the Hurricane Electric
"Tunnel Details" page.

```
uci -q delete network.wan6
uci set network.wan6="interface"
uci set network.wan6.proto="6in4"
uci set network.wan6.peeraddr="henet-endpoint-server-ipv4-address"
uci set network.wan6.ip6addr="henet-endpoint-client-ipv6-addr"
uci set network.wan6.mtu="1480"
uci set network.wan6.tunnelid="henet-tunnelid"
uci set network.wan6.username="henet-username"
uci set network.wan6.password="henet-update-key"
uci add_list network.wan6.ip6prefix="henet-routed-prefix-64"
uci commit network
ifup wan6
```

# Check IPV6 Connectivity

Check on the router first.
```
ping6 ipv6.google.com
PING ipv6.google.com (2607:f8b0:400b:803::200e): 56 data bytes
64 bytes from 2607:f8b0:400b:803::200e: seq=0 ttl=121 time=11.088 ms
```

The router should be serving correctly configured IPV6 configuration
to DHCP LAN clients. In the example below is the enthernet interface
configuration of a Mac that is connected to the router's LAN
interface. Expect to see multiple inet6 addresses. At least one of
them should begin with the `uci add_list
network.wan6.ip6prefix="henet-routed-prefix-64` IPV6 address prefix
configured on the router's wan6 interface.

If the IPV6 router/64 prefix value is `1234:567:8a:9b1/64` then you
would expect to see something similar to the following network
configuration on the client ethernet interface:

```
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=27<RXCSUM,TXCSUM,VLAN_MTU,TSO4>
	ether 00:23:32:d4:72:80 
	inet6 fe80::223:32ff:fed4:7280%en0 prefixlen 64 scopeid 0x4 
	inet 192.168.1.111 netmask 0xffffff00 broadcast 192.168.1.255
	inet6 1234:567:8a:9b1:223:32ff:fed4:7280 prefixlen 64 autoconf 
	inet6 1234:567:8a:9b1:b104:1994:896:aa46 prefixlen 64 autoconf temporary 
	inet6 fd65:972f:1bf2::223:32ff:fed4:7280 prefixlen 64 autoconf 
	inet6 fd65:972f:1bf2::19d6:5448:c14b:e14 prefixlen 64 autoconf temporary 
	inet6 1234:567:8a:9b1::fdc prefixlen 64 dynamic 
	nd6 options=1<PERFORMNUD>
	media: autoselect (1000baseT <full-duplex,flow-control>)
	status: active

```

The client should have IPV6 connectivity:

```
$ ping6 ipv6.google.com
PING6(56=40+8+8 bytes) 1234:567:8a:9b1:b104:1994:896:aa46 --> 2607:f8b0:400b:803::200e
16 bytes from 2607:f8b0:400b:803::200e, icmp_seq=1 hlim=120 time=9.889 ms
```

If this much works, then every client that connects to the router
should, in theory, have IPV6 internet access. Including wireless
clients, if you enable wifi on the router.

# WAN IP4 Address Update

If the WAN's IP4 public address changes then the Hurricane Electric
tunnel must updated. The 6in4 tunnel interface should take care of
this automatically. If it doesn't update correctly the IPV6 wan
interface you will lose connectivity.

Debug this by logging into
[tunnelbroker.net](https://tunnelbroker.net) and checking the "Client
IPV4 Address" (see the Tunnel Details image above). Compare it to the
router's WAN interface IP4 address. They should be identical.

You can also check the router logs to see if the update happened as
expected:

```
$ logread | grep wan6
Tue May 16 16:50:47 2023 daemon.notice netifd: Interface 'wan6' is now up
Tue May 16 16:50:47 2023 daemon.notice netifd: tunnel '6in4-wan6' link is up
Tue May 16 16:50:49 2023 user.notice 6in4-wan6: update 1/3: nochg 123.456.789.123
Tue May 16 16:50:49 2023 user.notice 6in4-wan6: updated
```

If you need to force a tunnelbroker.net Client IPV4 Address update
then try restarting the wan interface (not wan6), or execute an http
get on the "ExampleUpdate URL" (seen in the Tunnel Details image
above). You can execute this on the router command line as follows:

```
$ wget -4qO- $ https://henet-username:henet-update-key@ipv4.tunnelbroker.net/nic/update?hostname=henet-tunnelid
nochg 123.456.789.123
```

If the reply is "nochg ..." then the Client IPV4 Address was up to
date.

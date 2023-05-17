---
layout: post
title:  "OpenWrt IPV6 Tunnel Configuration (for Bell Fibre Internet)"
date:   2023-05-16
categories: IPV6
---

The is an account of my experience getting an IPV6 tunnel working over
my [Bell Canada fibre residential internet
service](https://www.bell.ca/Bell_Internet). It was a headache, so I
thought I would document it. If you have Bell fibre residential
internet service, and you have Bell's mandatory "GigaHub" router, and
you want IPV6 connectivity, then this is an exact recipe. However it's
mostly generic, and is still applicable if you have a different ISP.

[TLDR: jump to the solution](#the-solution)

Bell Canada's residential fiber internet service doesn't support
IPV6. An IPV6-over-IPV4 tunnel is a solution, and [Hurricane
Electric](https://www.he.net/) provide a free [IPV6 over IPV4 tunnel
service](https://tunnelbroker.net). Should be simple, right?

# Not so simple

Should be simple, but all my initial attempts failed and left me
frustrated. The tunnel would send data but no data was ever received,
and it behaved the same regardless of what platform I tested
with. Even pinging my router's public IP4 address proved unreliable,
and I needed that to work reliably to setup the tunnel. Others
[reported](https://forum.bell.ca/t5/Internet/Is-ipv6-tunneling-possible-with-Bell/m-p/12415)
similar problems, and have been similarly
[frustrated](https://forum.bell.ca/t5/Internet/Gigahub-and-IPv6/m-p/10489)
by this.

IPV6-over-IPV4 is implemented using [IP protocol
41](https://simple.wikipedia.org/wiki/Protocol_41). A call to Bell's
tech support confirmed that they don't block IP protocol 41 on their
network, indeed they said they don't block anything. In theory, then,
it should work. And, in theory at least, if their network isn't
blocking anything then what must be blocking the tunnel traffic is the
Bell [Home Hub 4000
router](https://support.bell.ca/internet/products/home-hub-4000-modem),
a.k.a. the "GigaHub", that Bell's fiber customer have no choice but to
use.

So how to get the IPV6 tunnel traffic through the Gigahub? It has
[DMZ](https://en.wikipedia.org/wiki/DMZ_(computing)) support, even an
"Advanced DMZ" mode that claims to completely bypass the router's
firewall. None of that helped. Indeed, it seemed even more unreliable
and frustrating. There are many reports of GigaHub DMZ acting
[buggy](https://forum.bell.ca/t5/Internet/Advanced-DMZ/m-p/10971), and
that was my experience.

# The solution

What I found does work to pass IPV6 tunnel traffic through the GigaHub
router, and through Bell's network, is [PPPoE
passthrough](https://forum.bell.ca/t5/Internet/Using-PPPoE-and-DMZ-Advanced-DMZ-for-Bridge-Mode-use-of-3rd/m-p/705).

The solution I settled on was a dedicated
[OpenWrt](https://openwrt.org/) router that configures a PPPoE
connection to transit the IPV4 traffic (the only traffic that Bell
supports), and a Hurricane Electric IPV6 tunnel, also running on the
OpenWrt router, to support the IPV6 traffic. Clients of this router
get both IPV4 and IPV6 internet access.

The PPPoE connection gets a new public IP address, different from the
GigaHub's own public IP address. This new public address is pingable,
and IPV6 Hurricane Electric tunnel traffic passes through the GigaHub
without problem.

Note, this additional, dedicated, IPV6 router doesn't prevent the rest
of the users on my home network (who don't care about IPV6
connectivity) from routing their normal IPV4 traffic directly through
the GigaHub.

# Get your hands on an OpenWrt router

For those who have never used OpenWrt: You need an router that is
[supported by OpenWrt](https://openwrt.org/toh/start), and of course
you need to
[install](https://openwrt.org/docs/guide-user/installation/generic.flashing)
the latest version OpenWrt on that router. I used OpenWrt version
22.03.5, which was the [current stable
release](https://openwrt.org/#current_stable_seriesopenwrt_2203) at
the time.

Any old router will work as long it's supported by the latest version
of OpenWrt. I happened to use an old [NetGear
WNDR3700](https://openwrt.org/toh/netgear/wndr3700) that I had lying
around unused. Old [used
routers](https://www.kijiji.ca/b-computer-accessories/canada/router/k0c128l0?sort=priceAsc)
are really cheap if you need one. Just be sure to check its OpenWrt
support. Try to avoid one that requires a [serial
console](https://openwrt.org/docs/techref/hardware/port.serial) to
perform the install if you don't have much experience with such
things, and wish to avoid added frustration.


# OpenWrt router configuration

The configuration steps below are all performed using the OpenWrt
router's command line interface and assume that you're are starting
with a fresh install. You have to ssh into the router to get to the
command line interface.

Connect your client computer directly to the router's ethernet LAN
interface and configure your client computer's ethernet interface to
use DHCP. The router will provide DHCP configuration to your computer,
and you should then be able access the router using:

```
ssh root@openwrt.lan
```

All the steps below can be performed using the router's
[LuCi](https://openwrt.org/docs/guide-user/luci/start) web interface
as well, but that's not documented here.

# Configure the wan (IPV4) network interface

The router's ethernet WAN port should be connected to the same network
as the PPPoE server (i.e. the Bell GigaHub).

Starting from a fresh install, log into your router using ssh and run
the following configuration script on your router to get an internet
connection.

You need the same username and password that your Bell GigaHub uses to
connect to Bell's network. You can find the username on your GigaHub's
web interface, but not the password. If you don't have the password
saved somewhere then you'll have to contact Bell to get it.

```
uci set network.wan.proto=pppoe
uci set network.wan.username="your-isp-username"
uci set network.wan.password="your-isp-password"
uci commit network
ifup wan
```

Wait a few seconds for the wan interface to come up and then verify
that you have an internet connection.

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

You'll need this public IP4 address later to create the [Hurricane
Electric IPV6 tunnel](https://tunnelbroker.net).

# Install the tunnel software

Install the [OpenWRT IPV6 tunnel
software](https://openwrt.org/docs/guide-user/network/ipv6_ipv4_transitioning),
and reboot the router.

```
opkg update
opkg install 6in4
reboot
```

# Create the Hurricane Electric IPV6 tunnel.

Visit [tunnelbroker.net](https://tunnelbroker.net) and create a new
IPV6 tunnel. You'll need your router's public IPV4 address (the WAN
address from above) to setup the new tunnel.

After it's setup, you'll need the need the Hurricane Electric tunnel
information below to configure the OpenWrt IPV6 wan6 interface.

![Hurricane Electric UI tunnel details](/assets/images/2023/2023-05-16-ipv6-tunnel-on-bell-fibre/TunnelDetail-IPv6Tunnel.png)
![Hurrican Electric UI tunnel advanced](/assets/images/2023/2023-05-16-ipv6-tunnel-on-bell-fibre/TunnelDetail-IPv6Advanced.png)

# Configure the wan6 (IPV6) network interface

Finally, configure the IPV6 wan6 interface. Substitute the
"henet-...."  place holders below using the matching values from the
Hurricane Electric "Tunnel Details" page shown above.

```
uci -q delete network.wan6
uci set network.wan6="interface"
uci set network.wan6.proto="6in4"
uci set network.wan6.peeraddr="henet-endpoint-server-ipv4-address" # you provide
uci set network.wan6.ip6addr="henet-endpoint-client-ipv6-addr"     # you provide
uci set network.wan6.mtu="1480"
uci set network.wan6.tunnelid="henet-tunnelid"                     # you provide
uci set network.wan6.username="henet-username"                     # you provide
uci set network.wan6.password="henet-update-key"                   # you provide
uci add_list network.wan6.ip6prefix="henet-routed-prefix-64"       # you provide
uci commit network
ifup wan6
```

# Check IPV6 Connectivity

You should now have IPV6 connectivity via the tunnel.

Check on the router first.
```
$ ping6 ipv6.google.com
PING ipv6.google.com (2607:f8b0:400b:803::200e): 56 data bytes
64 bytes from 2607:f8b0:400b:803::200e: seq=0 ttl=121 time=11.088 ms
```

The router should be serving correctly configured IPV6 configuration
to DHCP LAN clients. The example below is the enthernet interface
configuration of a Mac that is connected to the router's LAN
interface.

Expect to see multiple inet6 addresses. At least one of them should
begin with the "henet-routed-prefix-64" value that you provided
above. For example, If the IPV6 router/64 prefix value is
`1234:567:8a:9b1/64` then you would expect to see something similar to
the following network configuration on the client ethernet interface:

```
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	options=27<RXCSUM,TXCSUM,VLAN_MTU,TSO4>
	ether 00:23:32:d4:72:80 
	inet6 fe80::223:32ff:fed4:7280%en0 prefixlen 64 scopeid 0x4 
	inet 192.168.1.111 netmask 0xffffff00 broadcast 192.168.1.255
*	inet6 1234:567:8a:9b1:223:32ff:fed4:7280 prefixlen 64 autoconf 
*	inet6 1234:567:8a:9b1:b104:1994:896:aa46 prefixlen 64 autoconf temporary 
	inet6 fd65:972f:1bf2::223:32ff:fed4:7280 prefixlen 64 autoconf 
	inet6 fd65:972f:1bf2::19d6:5448:c14b:e14 prefixlen 64 autoconf temporary 
*	inet6 1234:567:8a:9b1::fdc prefixlen 64 dynamic 
	nd6 options=1<PERFORMNUD>
	media: autoselect (1000baseT <full-duplex,flow-control>)
	status: active

* - this is the IPV6 tunnel prefix provided by the router via DHCP
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

If the WAN's IP4 public address changes (which it will regularly on
Bell's network) then the Hurricane Electric tunnel must be
updated. The OpenWrt 6in4 tunnel interface should take care of this
automatically. If it doesn't succesfully update your IPV4 endpoint
address at [tunnelbroker.net](https://tunnelbroker.net), then you will
lose IPV6 connectivity.

Note that I have experienced occassions when it didn't update
correctly (for reasons that I do not yet understand). However, usually
it does update correctly.

Debug this by logging into
[tunnelbroker.net](https://tunnelbroker.net) and checking the "Client
IPV4 Address" (see the Tunnel Details image above). Compare it to the
router's WAN interface IP4 address. They should be identical.

You can also check the router logs to see if the update happened as
expected:

```
$ logread | grep 6in4-wan6
Tue May 16 16:50:49 2023 user.notice 6in4-wan6: update 1/3: nochg 123.456.789.123
Tue May 16 16:50:49 2023 user.notice 6in4-wan6: updated
```

If the update failed, it will be obvious from the log messages. If
there are no 6in4-wan6 log messages then the log messages from the
last update may have been lost do to the limited size of the log file
and you should consider the update status to be unknown.

If you need to force a tunnelbroker.net Client IPV4 Address update
then try restarting the IPV4 wan interface ("wan", not "wan6"), or
execute an http get on the "ExampleUpdate URL" seen in the Tunnel
Details image above. You can execute this on the router command line
as follows:

```
$ wget -4qO- $ https://henet-username:henet-update-key@ipv4.tunnelbroker.net/nic/update?hostname=henet-tunnelid
nochg 123.456.789.123
```

If the reply is "nochg ..." then the tunnel's Client IPV4 Address was
up to date. If it wasn't up to date, and was updated, then the wget
reply will reflect that.

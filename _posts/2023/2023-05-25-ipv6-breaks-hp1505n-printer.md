---
layout: post
title:  "IPv6 Breaks HP1505n printer"
date:   2023-05-21
categories: IPv6
---

# Printer shuts down if there is IPv6 network activity.

My venerable
[HP1505n](https://support.hp.com/us-en/drivers/selfservice/hp-laserjet-p1500-printer-series/3435666/model/3435670)
stopped working recently, would not even start, appeared to have
died. It turned out to be related to a home network upgrade and was
easily resolved.

Upon turning on, it would begin making the usual noises, status lights
would flash as usual, then it would unexpectly simply turn itself
off. Sometimes it would get as far as printing out a partial self-
test/configuration page before shutting down (if I pressed the "GO"
button before it shut itself off). It appeared like it was the end of
the road for the old printer. I initially thought the power supply was
failing, until I discovered that if I unplugged it from the network,
it would power up fine, and would print a complete
self-test/configuration page. If I connected to it via USB, and
disconnected it from the network, then it would print fine. If I
connected it to a standalone computer via ethernet, then it would also
print fine, but when I put it back on my home network, it would die.

I watched the network activity with
[Wireshark](https://www.wireshark.org/) and I could see the printer
wake up, send a few network discovery packets, then there was a
flourish of IPv6 activity on the network, and then the printer's
network activity went silent and the printer abruptly shut down. IPv6
traffic on my home network was new. Could it be that the old printer
was crashing as a result of new IPv6 activity on my home network?  It
turned out, yes, and the fix was to simply disable IPv6 on the
printer.

# Disable IPv6 

1. Connect the printer to a stand alone computer using ethernet and
turn it on. Don't worry about the computer's network configuration
just yet. It's important to do this first because if the printer's
ethernet port is not connected then it will not initialize its network
interface.

2. If the printer was formerly configured to use DHCP then it will
time out (because there is no DHCP server on your stand alone
printer-computer network), and the network interface will get a
default configuration. That will take about 30 seconds or so. Then
hold the "GO" button to print a status page. You should see that it
has defaulted to a network address of 169.254.241.75. Unless the
printer had a static IPv4 address configured, then it will show the
configured static address.

3. Configure your computer to use an IPv4 address on the same subnet
as the printer's IP address. For example, configure your computer's
address to be 169.254.241.99, and connect to the printer's web
interface at http://169.254.241.75. Adjust the computer's IP address
as necessary if the printer has a static IP address configured.

4. Disable IPV6 on the printer:

![HP1505n IPv6 config](/assets/images/2023/2023-05-25-ipv6-breaks-hp1505n-printer/HP1505n-IPV6-config.png)

Finally, reboot the printer. It should now ignore the IPv6 activity on
your network, and return to working... if IPv6 was the indeed the
problem.

# Factory Reset

If for some reason you need to perform a [factory
reset](https://h30434.www3.hp.com/t5/Printers-Archive-Read-Only/HP-LaserJet-P1505n/m-p/29244/highlight/true#M3367995),
here is how:

1. Turn off the printer
2. Hold the "GO" and "Cancel" buttons at the same time.
3. Press and release the power button while still holding the "Go" and
"Cancel" buttons.
4. Release the "Go" and "Cancel" buttons after about 6 seconds after
you press the power button.

Note, the factory default is IPv6 enabled, and DHCP on.




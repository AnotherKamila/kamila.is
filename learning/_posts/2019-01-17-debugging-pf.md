---
title: Debugging PF
excerpt: |
  PF, the firewall available in OpenBSD and FreeBSD, is much easier to configure than other alternatives. Still, firewalls are difficult and it's nearly impossible to get it right on the first try. The good thing is that PF makes it very easy to debug things. Here is a quick reference for debugging PF.
---

PF, the firewall available in OpenBSD (which is where PF was developed) and FreeBSD (which is what I use), is much easier to configure than other alternatives. Still, firewalls are difficult and it's nearly impossible to get it right on the first try. The good thing is that PF makes it very easy to debug things. Here is a quick reference for debugging PF.

# Syntax check

`service pf reload` will print syntax errors, if any.

If you just want to check the syntax of your config file without actually loading it, you can use `service pf check`.

# Show the firewall's state

The "you could have guessed it" way (because guessing works on FreeBSD): `service pf status` :-)

To see all sorts of other details about PF's state, you can use the `pfctl(8)` utility. As usual in FreeBSD, `man pfctl` is very useful. For quick reference, these are the things I use most often:

* `pfctl -s rules` (show rules; can be shortened to `pfctl -sr`): shows how it has interpreted the filtering rules in your config file, including substitutions and defaults
* `pfctl -s nat` (or `pfctl -sn`): shows the NAT rules
* `pfctl -s all` (or `pfctl -sa`): shows all the things

# pflog: readable logs of what is happening with packets

When it comes to firewalls, `pflog` is just the best thing ever. Seriously.

## How it works (some background info)

`pflog` creates a virtual network interface, and as packets flow through the firewall, the ones marked for logging also appear on this interface. They are the exact same packets, so you can inspect what exactly is in them. In addition, they have a special header which contains information about which PF rule was matched.

This is really really really powerful. Besides debugging, you could use it for lots of other things: for example, you could filter out the packets that interest you using the rich and wonderful filtering capabilities of PF, and use `tcpdump -w` (or `pflogd`) on the `pflog0` interface to capture them for later analysis. The fact that this creates a network interface with the actual packets makes it possible to do all sorts of things. Here, I will just focus on debugging.

On FreeBSD, `tcpdump` is augmented to show the information from PF on the packets. Here is what it looks like in practice:

```
root@gasfaser:~ # tcpdump -eni pflog0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on pflog0, link-type PFLOG (OpenBSD pflog file), capture size 262144 bytes
18:16:35.674483 rule 0/0(match): nat out on tun0: 100.98.44.48 > 172.217.168.35: ICMP echo request, id 29367, seq 1, length 64
18:16:36.506939 rule 1/0(match): pass in on wlan0: 192.168.0.100.60084 > 185.203.114.130.443: Flags [S], seq 110274663, win 29200, options [mss 1460,sackOK,TS val 1615450250 ecr 0,nop,wscale 7], length 0
18:16:36.507034 rule 2/0(match): pass out on tun0: 100.98.44.48.53245 > 185.203.114.130.443: Flags [S], seq 110274663, win 29200, options [mss 1460,sackOK,TS val 1615450250 ecr 0,nop,wscale 7], length 0
18:16:40.641317 rule 2/0(match): pass out on tun0: 100.98.44.48.123 > 130.60.204.10.123: NTPv4, Client, length 48
18:16:53.940265 rule 1/0(match): pass in on wlan0: 192.168.0.100.58254 > 172.217.168.78.443: UDP, length 1350
```

(Note that by default only the first packet of each connection is logged, as PF keeps state and therefore further packets of the same connection take the quick path. If you want to log all packets of a connection, use `log all`. You could have guessed it! :D)

## How to enable it

1. Add logging into the PF rules that interest you. Examples:
    * `nat on ...` -> `nat log on ...`
    * `block all` -> `block log all`
    * `pass in quick ...` -> `pass in log quick ...`
    
    Then reload pf with `service pf reload`.
2. Start the `pflog` service, which creates the `pflog` interface:
    `service pflog onestart`
3. Look at what is going through the interface: `tcpdump -eni pflog0`

This makes it obvious when your packets are matching different rules than intended, and that's why debugging PF doesn't suck :-)
    
Note: Logging decreases performance. There is no performance hit if you just have the `log` statements in your PF config file, but having `pflogd` log all the packets will have a noticeable effect. Therefore, on my systems I tend to not start `pflog` on boot and only `onestart` it when I'm actually going to use it.

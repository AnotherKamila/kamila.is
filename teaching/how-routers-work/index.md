---
title: How do Routers Work, Really?
---

**Work in progress**: I still need to clean this up & add the complete source code. The snippets here are conceptually correct, but some minor details need to be updated too. ETA for a more or less done version 2019-01-10 or earlier.

This is the inside view of how exactly a router operates. You only need to know this if you are poking inside a router implementation. If that is the case, my condolences.

At the end of this exposition, I will give you the complete source code to a functional router (written in P4 (the new & shiny software-defined networking thing)). My aim is that you will understand every line of that.

I accompany my explanations below with some P4 code. I think it is useful to read it even if you've never seen P4, because it shows a bit more detail than the text and I believe that it is sufficiently pseudocode-ish. Here is all you need to know to read it:<sup>[1](#fn1)</sup>
* everything happens per packet
* `hdr` are the packet's parsed headers
* `standard_metadata` is how you tell the switch to do things with the packet (like send it on a specific port)
* `meta` are user-defined in-memory variables which can be used e.g. for matching in tables

# 0. Some Terminology

* Figuring out what should be done with packets is done by _the control plane/in the slow path/on the CPU/by the controller_ or similar phrases. I will refer to all of this as "the control plane".
* Actually forwarding the packets  is done by _the data plane/in the fast path/in the hardware/in the switch_ and such. I will refer to this as "the data plane".

Software-defined networking makes these distinctions not always entirely obvious, but I will mention cases like that.

# 1. High-level Overview

A _switch_ (or an L2 switch :-) ) is an L2-only thing. It knows about L2 stuff such as MAC addresses and ports. It does **not** know about anything like IP addresses. It has a **MAC table**: it maps MAC addresses to ports.

A _router_ (or an L3 switch by some people's vocabulary) operates on L3 only. It knows about L3 stuff such as IP addresses and interfaces and hosts. It does **not** know about L2 stuff such as MAC addresses or ports.<sup>[2](#fn2)</sup> In fact, the routing parts of the router would not have to be changed at all if you decided to use something other than ethernet on L2. It has a **routing table** (details later): a table of subnets/prefixes and how to reach them.

What you normally call a router (that box sitting over there) is actually a router (for handling L3) and one or more switches (for handling L2), and some glue in between. They may in fact be separate chips in hardware.

You need glue to put together the L2 and the L3. This "L2.5" glue is ARP (or NDP for IPv6). It usually lives in the router, but it is glue, not routing, and it can be thought of separately.

## The Data Plane: Life of a Packet

When a packet arrives and needs to be sent further, these things have to happen to it:

1. It needs to be _routed_: the **router**, based on L3 information, decides where it needs to go ,in L3 speak -- it will decide which _host_ to send it to, but not how. This corresponds to the _routing table_ (or FIB).
2. It needs to be passed down to L2: this is where the L2.5 ARP/NDP **glue** translates the L3-speak IP address to L2-speak MAC address. This is the _ARP table_.
3. It needs to be _forwarded_ on the correct port: the **switch** puts the packet on the correct port. This is the _MAC table_.

# The details: What *exactly* is going on?

## Life of a Packet, Now Properly

### 1. "It needs to be routed": L3/router

The packet has a destination IP address. This is matched in the _routing table_, using a longest-prefix match (LPM), i.e. it matches prefixes. It may either be for a host the router is directly connected to (on some interface), or it may need to be sent further, through a _gateway_ (through some interface). Therefore: **The routing table maps a _prefix_ to either _a next hop through a gateway and an interface_, or _a direct connection through an interface_**.

```
routing_table : Prefix -> NextHop (GatewayIP, Interface) | Direct Interface
```

Note that the next hop's IP address is in the router's memory only: it does not appear in the packet at any time.

The P4 code defining the IPv4 routing table is:

```
action ipv4_through_gateway(ipv4_addr_t gateway, interface_t iface) {
    meta.out_interface = iface;
    meta.ipv4_next_hop = gateway;  // send through the gateway
}

action ipv4_direct(interface_t iface) {
    meta.out_interface = iface;
    meta.ipv4_next_hop = hdr.ipv4.dst_addr;  // send directly to the destination
}

table ipv4_routing {
    key = {
        hdr.ipv4.dst_addr: lpm;  // match prefixes
    }
    actions = {
        ipv4_through_gateway;    // ipv4_through_gateway(gateway, iface)
        ipv4_direct;             // ipv4_direct(iface)
        drop;
    }
    default_action = drop();     // If there is no route, drop it -- in reality, we might want to
                                 // send an ICMP "No route to host" packet.
                                 // Note that this is the default route, so control plane might
                                 // want to set a default gateway here instead of dropping.
    size = ROUTING_TABLE_SIZE;
}
```
(and the exact same thing for IPv6)

### 2. "It needs to be passed down": L2.5/ARP glue

If we did not drop the packet because there was no route, we now know the IP address and interface of the next hop. (Note that this is a host that is connected to us directly -- it is sitting on the same wire.)
We need to translate this into an L2 MAC address in order to pass it to the switch. We do it via the ARP table:

```
arp_table : (IPv4Address, Interface) -> MACAddress
```

Note: `Interface` conceptually belongs there, but `IPv4Address` should be unique. We need to store the interface in the control plane anyway, because we want to pre-emptively re-send ARP requests when an entry is about to expire, but in the data plane it is not strictly necessary.

An interesting question arises here: What do we do if there is no match, i.e. when we don't know the MAC address for the IP? First, we send an ARP request. Then, most routers drop the packet (relying on either retransmissions or "nobody will miss it"). Storing the packet until the ARP reply comes back (or until it expires) also works.
Sending ARP requests is normally done in the control plane, because the ARP requests need to be throttled and expired and such.

P4 code:
```
action set_dst_mac(mac_addr_t dst_addr) {
    hdr.ethernet.dst_addr = dst_addr;
}

table ipv4_arp {
    key = {
        meta.ipv4_next_hop: exact;  // next_hop is the host we found in the routing step; we want to send to that
        meta.out_interface: exact;  // conceptually this belongs here, but actually next_hop should be unique
    }
    actions = {
        set_dst_mac;                // set_dst_mac(mac)
        drop;
    }
    default_action = drop();
    size = ARP_TABLE_SIZE;
}
```

IPv6 uses NDP instead of ARP, which is different but the same ;-)

### 3. "It needs to be forwarded": L2/switch

This is L2 / the switch. It works on each interface separately (it could be multiple chips in hardware (TODO is it?)). It gets a packet with some destination MAC address, and it decides on which port it should put it. It uses a _MAC table_ to do it:

```
mac_table : MACAddress -> Port
```

P4 code:

```
// note: we're operating on metadata.out_interface

action set_out_port(port_t ports) {
    standard_metadata.egress_spec = port;
}

action broadcast() {
    // Implementation depends on the switch.
    // In v1model, use a multicast group corresponding to all ports on metadata.out_interface.
}

// we call it dmac -- see below why
table dmac {
    key = {
        hdr.ethernet.dst_addr: exact;
    }
    actions = {
        set_out_port;  // set_out_port(port)
        broadcast;     // no params, uses metadata.out_interface
                       // remember to set broadcast for 0xffffffffffff in the control plane
        drop;
    }
    default_action = drop();
    size = ARP_TABLE_SIZE;  // we can have at most as many ports as MAC addresses
}

```

Note: Real switches are a bit more complicated than that: for example, redundant links mean that a MAC address may be on more than one port. However, you will notice when you need to think about this. Normally considering the simple version is sufficient.

### The logic: Applying the tables

1. apply routing => find the next hop (either gateway or direct)
2. apply ARP translation to the "next hop" host
3. send out on the right port

In P4:
```
apply {
    routing.apply();  // fills out metadata.next_hop
    arp.apply();      // sets pkt.ethernet.dst_addr to the MAC of next_hop
    dmac.apply();     // sends out on the port for pkt.ethernet.dst_addr
}
```

(Note: While this is conceptually correct, we actually also want to apply the auxiliary table mentioned below. The full code contains that.)

## The Control Plane: How to Fill the Tables

Starting at the bottom for a change:

### L3 / routing table

Filled out by the control plane, depending on the context:

* In your home router, it probably has only two entries: the local network (something like 192.168.0.0/24) => direct on the internal interface, and a default route via your ISP's gateway on the external interface.
   In this case, the routing table is static and is filled out by the firmware according to the settings.
* In a small company router, there might be a direct network such as 10.0.0.0/24, a remote office in 10.0.1.0/24 via a VPN server, and a default route from the ISP.
   The default route and the direct route would also be filled out from the settings (by its OS), and the route to the remote office would be added by the VPN software.
* In an ISP's router, there might be a few routes for connections to other ISPs and no default route. In this case, the control plane would keep an extended version of the table (the routing information base, or RIB) with extra info such as costs or multiple paths, and it would compute the routing table from that. The RIB would be filled using some control-plane protocol such as BGP.
* In a software-defined networking exercise, it would be filled by the controller script that reads the network topology from the simulator and does something reasonable with it :-)

### L2.5 / ARP table

We need to fill this using the ARP protocol. When we try to find an IP address and it isn't in our ARP table, we send an ARP request and add the entry when we receive a reply. The entries expire (the timeout is usually a couple of minutes to a couple of hours).

The ARP requests are usually handled by the control plane, because it is a good idea to throttle them (otherwise sending to a non-existent host would be an easy DOS attack).

(In our code, we send the ARP requests from the data plane because the controller might be on a different physical network. We don't throttle anything either, because we're lazy. Don't do that in production!)

### L2 / MAC table

This is filled out by the switch observing the traffic: if a packet appears on a port, that packet's source MAC address is associated with that port.

In detail, what we want to do is: If we see a packet with a source MAC address that we have not seen before, we add it to the MAC table. When implementing this with P4, the easiest option is to have an extra table with seen addresses and learn the MAC if we have a miss in that one. The code is:

```
action mac_learn() {
    // send pkt.ethernet.src_addr and standard_metadata.ingress_port to the controller
    // so that it adds it to the dmac table
    // (tables must be modified in the control plane in P4)
    // the control plane must also add it to smac with a NoAction
}

// Used to know if we have seen this source MAC before and learn it if not.
// Trick: we set the default_action to mac_learn and we always add entries with a NoAction.
// That way we do the hard work only on a miss.
table smac {
    key = {
        hdr.ethernet.src_addr: exact;
    }
    
    actions = {
		mac_learn;
		NoAction;
    }
    default_action = mac_learn;
    size = ARP_TABLE_SIZE;
}

// and in the apply block:
apply {
    ...
    smac.apply();
    ...
}

```

# Gimme the code!

TODO full code

# Next steps

Want to know more than this overview? Read _TCP/IP Illustrated, Vol. 1 & 2_! I put this together by talking to a person who reads that book a lot, so it must be good :D

-------------------------------------------------------------------------------------------------

<a name="fn1">1</a>: Or that is what I think -- [complain](https://github.com/AnotherKamila/kamila.is/issues/new?labels=teaching&title=[teaching/how-routers-work]+Title) if I am wrong!

<a name="fn2">2</a>: If you come from e.g. FreeBSD, you might be screaming that the routing table sometimes knows about MAC addresses. This is a shortcut to speed things up by avoiding the extra lookup, but conceptually it should not exist. if you happen to be implementing a router that does not care about performance, you don't have to do that (and I have not).

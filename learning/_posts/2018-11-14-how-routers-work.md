---
title: How do Routers Work, Really?
---

This is the inside view of how exactly a router operates. You only need to know this if you are poking inside a router implementation. If that is the case, my condolences.

At the end of this exposition, I will give you the complete source code to a functional router (written in P4 (the new & shiny software-defined networking thing)). My aim is that you will understand every line of that.

# 0. Some Terminology

* Some stuff is done by _the control plane/in the slow path/on the CPU/by the controller_ or similar phrases. I will refer to all of this as "the control plane".
* Other stuff is done by _the data plane/in the fast path/in the hardware/in the switch_ and such. I will refer to this as "the data plane".

Software-defined networking makes these distinctions not always entirely obvious, but I will mention cases like that.

# 1. High-level Overview

A switch (or an L2 switch :-) ) is an L2-only thing. It knows about L2 stuff such as MAC addresses and ports. It does **not** know about anything like IP addresses. It has a **forwarding table**: it maps MAC addresses to ports.

A router (or an L3 switch by some people's vocabulary) operates on L3 only. It knows about L3 stuff such as IP addresses and interfaces and hosts. It does **not** know about L2 stuff such as MAC addresses or ports.<sup>[1](#fn1)</sup> It has a **routing table** (details later): a table of subnets/prefixes and how to reach them.

What you normally call a router (that box sitting over there) is actually a router (for handling L3) and one or more switches (for handling L2). They may in fact be separate chips in hardware.

You need glue to put together L2 and L3 switches. This "L2.5" glue is ARP (or NDP for IPv6). It usually lives in the router layer, but it is glue, not routing, and it can be thought of separately.

## The Data Plane: Life of a Packet

When a packet arrives and needs to be sent further, these things have to happen to it:

1. It needs to be _routed_: the **router**, based on L3 information, decides where it needs to go ,in L3 speak -- it will decide which _host_ to send it to, but not how. This corresponds to the _routing table_ (or FIB).
2. It needs to be passed down to L2: this is where the L2.5 ARP/NDP **glue** translates the L3-speak IP address to L2-speak MAC address. This is the _ARP table_.
3. It needs to be _forwarded_ on the correct port: the **switch** puts the packet on the correct port. This is the _forwarding table_.

# The details: What *exactly* is going on?

## Life of a Packet, Now Properly

### 1. "It needs to be routed": L3/router

The packet has a destination IP address. This is matched in the _routing table_, using a longest-prefix match (LPM), i.e. it matches prefixes. It may either be for a host the router is directly connected to (on some interface), or it may need to be sent further, through a _gateway_ (through some interface). Therefore: **The routing table maps a _prefix_ to either _a next hop through a gateway and an interface_, or _a direct connection through an interface_**.

```
routing_table : Prefix -> NextHop (GatewayIP, Interface) | Direct Interface
```

The P4 code defining the IPv4 routing table is:

```
action through_gateway(ipv4_addr_t gateway, interface_t iface) {
    metadata.out_interface = iface;
    metadata.next_hop      = gateway;  // send through the gateway
}

action direct(interface_t iface) {
    metadata.out_interface = iface;
    metadata.next_hop      = hdr.ipv4.dst_addr;  // send directly to the destination
}

table routing_ipv4 {
    key = {
        hdr.ipv4.dst_addr: lpm;  // match prefixes
    }
    actions = {
        through_gateway;         // through_gateway(gateway, iface)
        direct;                  // direct(iface)
        drop;
    }
    default_action = drop();     // if there is no route, drop it -- in reality, we might want to send an ICMP "No route to host" packet
                                 // note that this is the default route, so control plane might want to set a default gateway here instead of dropping
    size = ROUTING_TABLE_SIZE;
}
```
(and equivalent for IPv6)

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
    pkt.ethernet.dst_addr = dst_addr;
}

table arp {
    key = {
        metadata.next_hop: exact;       // next_hop is the host we found in the routing step; we want to send to that
        metadata.out_interface: exact;  // actually next_hop should be unique, so this can be removed if low on table memory
    }
    actions = {
        set_dst_mac;                    // set_dst_mac(mac)
        drop;
    }
    default_action = drop;
    size = ARP_TABLE_SIZE;
}
```

IPv6 uses NDP instead of ARP, which is different but the same ;-)

### 3. "It needs to be forwarded": L2/switch

This is L2 / the switch. It works on each interface separately (it could be multiple chips in hardware (TODO is it?)). It gets a packet with some destination MAC address, and it decides on which port it should put it. It uses a _forwarding table_ to do it:

```
forwarding_table : MACAddress -> Port
```

P4 code:

```
// note: we're operating on metadata.out_interface

action set_out_port(port_t ports) {
    standard_metadata.egress_spec = port;
}

action flood() {
    // Implementation depends on the switch.
    // In v1model, use a multicast group corresponding to all ports on metadata.out_interface.
}

table forwarding {
    key = {
        hdr.ethernet.dst_addr: exact;
    }
    actions = {
        set_out_port;  // set_out_port(port)
        flood;         // no params, uses metadata.out_interface
                       // remember to set flood for 0xffffffffffff in the control plane
        drop;
    }
    default_action = drop();
    size = ARP_TABLE_SIZE;  // we can have at most as many ports as MAC addresses
}

```

### The logic: Applying the tables

1. apply routing => find the next hop (either gateway or direct)
2. apply ARP translation to the "next hop" host
3. send out on the right port

In P4:
```
apply {
    routing.apply();
    arp.apply();
    forwarding.apply();
}
```

No `if`s are necessary, because misses and different actions (gateway vs direct on L3, or send to destination vs broadcast on L2.5) can be handled uniformly by the layer below.

## The Control Plane: How to Fill the Tables

Starting at the bottom for a change:

### L3 / routing table

Filled out by the control plane, depending on the context:

* In your home router, it probably has only two entries: the local network (something like 192.168.0.0/24) => direct on the internal interface, and a default route via your ISP's gateway on the external interface.
   In this case, the routing table is static and is filled out by the firmware according to the settings.
* In a small company router, there might be a direct network such as 10.0.0.0/24, a remote office in 10.0.1.0/24 via a VPN server, and a default route from the ISP.
   The default route and the direct route would also be filled out from the settings (by its OS), and the route to the remote office would be added by the VPN software.
* In an ISP's router, there might be a few routes for connections to other ISPs and no default route. The routing table would be filled using some control-plane protocol such as BGP. In this case, the control plane would keep an extended version of the table (the RIB) with extra info such as costs or multiple paths, and it would compute the routing table from that.
* In a software-defined networking exercise, it would be filled by the controller script that reads the network topology from the simulator and does something reasonable with it :-)

### L2.5 / ARP table

TODO

### L2 / forwarding table

TODO

## Gimme the code!

TODO full code

-------------------------------------------------------------------------------------------------

<a name="fn1">1</a>: If you come from e.g. FreeBSD, you might be screaming that the routing table sometimes knows about MAC addresses. This is a shortcut to speed things up by avoiding the extra lookup, but conceptually it should not exist. if you happen to be implementing a router that does not care about performance, you don't have to do that (and I have not).

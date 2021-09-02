---
title: Route tables and Iptables for Routing
author: Stack Underflow
date: '2021-08-28'
categories:
  - Tech
tags:
  - networking
  - Linux
image: images/iptables-filter.jpeg
description: the first post in series Kubernetes Networking Explained
---

Route tables and iptables are two utilities that are used in Linux for routing. This article explains how to use them for routing packets. 

## Route tables

*Routing* is a process that determines the output interface of an IP packet by matching its destination address. Other information could also be set during routing, like the gateway address of the packet. The routing process is generally configured by *route tables*, which is shown using the command `ip route`. 

### Home network configuration

Below is a typical configuration for a home network, where the two machine (PC1 and PC2) are in a local network (`192.168.1.0/24`) and they connect to the Internet by a router. Note that here I show two PCs, but actually they could be any devices (mobiles phones, TVs, or gaming consoles, etc.) and there could be any number of them.

![home network](images/network.png)

You might be wondering: Oh I do not have a switch at home! In fact, the typical home routers have a switch embedded. All machines connected to a LAN interface are connected to the embedded switch, which in turn connects to a router port. The WAN interface is connected to the other router port. Note that the router in typical home routers has only two ports, so it only creates two networks: the local network and the public network.

![home router](images/home-router.png)

What will the route table on PC1 look like? Possibly the following (simplest case)

``` sh
192.168.1.0/24 dev eth0  # rule for local network
default via 192.168.1.1 dev eth0   # rule for public network
```

The first rule directs all traffic whose destination matching `192.168.1.0/24` to `eth0`, and the second rule directs traffic matching `0.0.0.0/0` (`default` is the alias for `0.0.0.0/0`) to the same interface. The route table rules follow the "longest prefix match" rule. For example, a packet goes to `192.168.1.3` matches both the first and second rule (the second rule matches any IP addresses since its prefix is `/0`), but the first rule is selected since it has a longer prefix length (`24 > 0`). As a result of the longest prefix rule, the route table above sends all local network traffic to rule 1, and all other traffic (public network traffic) to rule 2. The difference between the two rules is that rule 2 has a `via` keyword.

What does the keyword `via` mean in the second rule? The address after `via` is called *gateway*. Basically, for addresses in the same local network, we do not have a gateway (like the first rule). While for addresses which are in a different network, we set the gateway to the address of the router (like the second rule). 

### Local traffic

For route table rules without a gateway, the destination machine must be in the same network as the sending machine. The sending machine finds the MAC address for the destination by the *ARP (Address Resoluition Protocol)* protocol. Below is the example of sending a packet from PC1 to PC2. This traffic matches the first rule in the route table, which doesn't have a gateway address. In this case, PC1 assembles the Ethernet frame by

1. PC1 looks for the MAC address of `192.168.1.3` by broadcasting an ARP request to the local network, which asks all other hosts: *Who is `192.168.1.3`*?
2. PC2 answers PC1 with its own MAC address, and PC1 sets it as the destination MAC address of the enclosing Ethernet frame.

![local routing](images/route-local.png)

Note that in this case, the destination MAC address and the destination IP address point to the same device: PC2.

An important detail is that the ARP table (the table which maps IP address to their corresponding MAC addresses) is not stored at the switch. A switch works at layer 2 (data link layer), which means it doesn't have the knowledge of IP addresses. Instead, the ARP tables are stored at the devices connecting to the switch. In our case, PC1, PC2 and the router.

### Traffic to other networks

So what if PC1 sends a packet to a different network? Here is an example of sending a packet from PC1 to a public IP address (say, `8.8.8.8`). This traffic matches the second rule in the route table. This rule is equipped with a gateway address, which is the address of the router, `192.168.1.1`. In this case, PC1 assembles the Ethernet frame by

1. PC1 looks for the MAC address of the gateway (`192.168.1.1`) by the ARP protocol. Note that when a gateway is set, the host looks for the MAC address of the gateway, instead of the destination IP address. If it looked for the destination IP no one would answer because that IP is not even in the local network.
2. The router answers PC1 with its MAC address, and PC1 sets it as the destination MAC address of the enclosing Ethernet frame.

![public routing](images/route-public.png)

Note that in this case, the destination MAC address and the destination IP address point to different devices. The MAC address points to the router, which inspects the enclosed IP packet upon receiving this frame. The router then finds that this packet is not destined to itself, and sends it out to the public network via `eth0`.

Note that it illustrates the difference between a host and a router. A host (like PC1 and PC2) usually only receive and process packets destining itself, while a router typically handles packets destining other devices (there are also traffic destining a router, like when you manage the router's configuration via a web console). 

Route table rules also have other fields like `scope` and `src`, you can take a look at [the manual](http://linux-ip.net/html/tools-ip-route.html).

## Iptables

Iptables provide 5 tables (filter, nat, mangle, security, raw), but the most commonly used are the _filter_ table and the _nat_ table. Tables are organized as _chains_, and there are totally 5 predefined chains, PREROUTING, POSTROUTING, INPUT, FORWARD and OUTPUT.

Here we focus only on the _nat_ table. The _filter_ table is also important but it's mainly used for firewalls, so we do not discuss it here. The _nat_ tables is used for network address translation, and it's available in PREROUTING, POSTROUTING and OUTPUT chains.

Below is the general diagram of the nat table in iptables.

![iptables](images/iptables.png)

It helps to break the diagram down. For an incoming packet, it goes through the PREROUTING chain, then the kernel makes the routing decision based on the routing table. If the packet is destined for the machine itself, it's put into a local process listening on the machine. If it's destined for another machine (so this machine serves as a router), the kernel will put it through the POSTROUTING chain and then send it to an output interface.

A packet generated from a local process will be routed and then put into the OUTPUT chain. The _reroute check_ step is a little bit complicated so we will explain it afterwards. The packet then goes to POSTROUTING and then goes to an output interface. For the discussion of the nat table, we assume that output packets never go to localhost.

### SNAT in iptables

Below is the home network shown in the route tables section

![home network](images/network.png)

To access the Internet from the two computers in the local network, the following SNAT (S stands for _source_) rule has to be added to the router:

```shell
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 50.60.70.80
```

Here `-t` stands for _table_, `-A` stands for _append_, `-o` stands for _output interface_, and `-j` snads for _jump_. This rule will replace the source address of any IP packets going to the Internet through interface eth0 with 50.60.70.80, the public IP address of the router. For any response packets, the reverse operation that sets the destination address to the private address of the connecting device (e.g. 192.168.1.2) is automatically applied so we do not need to worry about configuring it. The network address translation process works like the diagram below.

![snat](images/snat.png)

Note that SNAT rules has to be configured in the POSTROUTING chain. Sometimes this requirement is reasonable. For example, in our case, the matching condition includes the output interface (`eth0`) so it can only be used when the routing decision has been made. But sometimes we do not match against output interfaces, and it's still a requirement that SNAT can only be in the POSTROUTING chain.

However, our router may be allocated a dynamic address by DHCP instead of being configured a static one. In this case, we could use the MASQUERADE target, which is similar to SNAT except that we do not need to specify `--to-source` address. The public address of the router will be used automatically.

 ``` shell
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
 ```

MASQUERADE can only be used in the POSTROUTING chain because the source IP address of the packet will be obtained from the network interface it gets routed to (in our case, `eth0`). 

### DNAT in iptables

Another use case of NAT is _port forwarding_. when we are running a service on one of our computers, say, PC1, and would like to access it from the Internet. Suppose the service is opened on port 80 and we want to access it on port 8080 of the router, we will add the following DNAT (D stands for _destination_) rule.

```shell
iptables -t nat -A PREROUTING -m tcp -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
```

The option `-p tcp` (`p` stands for _protocol_) matches for TCP connections (so this rule doesn't apply to UDP), while `--dport 8080` (`--dport` stands for _destination port_) matches for TCP traffic destining port 8080. This rule will replace the destination address of any IP packets going to port 8080 with 192.168.1.2, the IP address of the service. The port is also replaces with 80, the port of the service. For any response packets, the reverse operation is also automatically applied, just like in SNAT. The process works like the diagram below.

![dnat](images/dnat.png)

You might be wondering why iptables, a layer 3 (network layer) tool, could do modifications to port information, which is on layer 4 (transport layer). In fact, although iptables mainly work on layer 3, it has been extended to work on layer 4 as well, so working with ports is also possible for iptables. In our case, the option `-m tcp` (`m` stands for "match" an extension) loads an extension called `tcp` to work on ports, and the `--dport` option is provided by this extension. In fact, the option `-p tcp` implicitly loads the `tcp` extension so `-m tcp` is actually redundant.  [Here](https://manpages.ubuntu.com/manpages/xenial/man8/iptables-extensions.8.html) is the documentation for all standard extensions on Ubuntu. Also note that DNAT rules must be configured on the PREROUTING chain or OUTPUT chain because it's useless to modify the destination IP address after routing decision has been made.

There is an issue with the setup above: A process running on the router cannot access the service on port 8080. Note that our command puts the DNAT rule into the PREROUTING chain, but packets generated by local process goes to the OUTPUT chain instead of PRERPOUTING. As a result, we have to also configure DNAT on the OUTPUT chain in order for a local process to access the service.

```shell
iptables -t nat -A OUTPUT -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
```

Note that `-m tcp` is removed in the command above. It's also valid. Also note that, unlike machines from the Internet, the local process does not access the service on using IP address 50.60.70.80. It might use 127.0.0.1 (or other localhost address instead).

### Load balancing

What if the service is running on both PC1 and PC2? In this case, the router has to do load balancing by distributing traffic to PC1 and PC2 randomly. We use an extension called `statistic` to implement it. Again, `statistic` is also a [standard extension](https://manpages.ubuntu.com/manpages/xenial/man8/iptables-extensions.8.html) on Ubuntu.

```shell
iptables -t nat -A PREROUTING -p tcp --dport 8080 -m statistic --mode random --probability 0.5 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.3:80
```

The first rule sends TCP traffic destining port 8080 to 192.168.1.2:80 with 50% probability, while the second rule sends all (remaining) traffic to 192.168.1.3:80. You might be wondering why the second rule is not configured to use `statistic`. In fact, iptables match rules linearly. 50% of the traffic matches the first rule and the second rule only matches the remaining 50%, so just sending all of them to PC2 is sufficient to it. The effective probability of sending traffic to PC2 is `(1-50%)*100%=50%`.

Note that the two command above only configure load balancing for packets sending to the router from elsewhere, To also configure load balancing for local processes, we need to add them to the OUTPUT chain.

```shell
iptables -t nat -A OUTPUT -p tcp --dport 8080 -m statistic --mode random --probability 0.5 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A OUTPUT -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.3:80
```

### What is reroute check?

For a packet generated by a local process, it goes to the OUTPUT chain after the routing decision. The local process specifies the destination address of the packet, but usually the source address is not specified. The kernel matches the destination address against the route table, selects an output interface and takes the IP address of the selected interface as its source address. As a result, we have access to both source address and destination address of the packet in the OUTPUT chain. 

However, as is shown above, we might do NAT in the output chain which might change the destination address. In this case, we need to reselect an output interface, and this is what the "reroute check" step does.

![reroute check](images/reroute-check.png)

### Hairpin NAT

Let's go back to the scenario that the service is running only on PC1. What happens when, say, PC2 accesses 50.60.70.80:8080 and gets redirected to PC1? The process is illustrated in the diagram below. Red line stands for request packets and green line stands for response packets.

![hairpin not working](images/hairpin-notworking.png)

The destination address of the request packet gets translated to 192.168.1.2:80, which is the IP address and port for the service running on PC1. The request packet arrives at PC1 successfully. The response packet will not go to the router, since the source address of the request packet is 192.168.1.3 and it's in the local network. As a result, the response packet arrives at PC2 through the switch, without going to the router.

What is the problem here? From the perspective of PC2, it sends a packet to 50.60.70.80:8080 but receives a packet from 192.168.1.2:80. In this case, it will not think the received packet as the response for the request packet, so the upper layer (the transport layer) will not put them into the same connection. For example, if the request packet is a TCP handshake SYN and the response is a TCP handshake SYN/ACK, the handshake will not succeed.

The reverse rule of DNAT which sets the source address to 50.60.70.80:80 applies automatically for response IP packets going through the router, but here the response packets never go to the router. What is the solution for this issue? It turns out we must add a MASQUERADE rule.

```shell
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth1 -p tcp --dport 80 -j MASQUERADE
```

The whole process after MASQUERADE is added looks like the diagram below.

![hairpin working](images/hairpin-working.png)

When the request packet goes through the router, its destination address is translated to 192.168.1.2:80 by the DNAT rule. In addition to this, its source address is translated to 192.168.1.1, the address of the gateway, by the MASQUERADE rule. Because of the translation of the source address, the response packet will be sent to the router, which gives the reverse rule of DNAT a chance to apply, so the source address of the response packet is translated to 50.60.70.80:8080. PC2 now sees the source address of the response packet to match the destination address of the request packet, so the upper layer is able to put them into one connection.

The situation that a machine in the internal network access a service exposed through DNAT is called _hairpin NAT_, since the packet is forwarded back, just like a hairpin.

![hairpin](images/hairpin.jpeg)

The case where a machine access itself through hairpin NAT is similar, but a little bit harder to reason. I include a diagram below. You might be wondering why a machine may access itself through hairpin NAT. One situation is when a group of machines are providing a service which is load-balanced by the router. If one of the machines access the service and the router happens to route the request back to the machine itself, a hairpin NAT looping back to the same machine will happen.

![hairpin same machine](images/hairpin-same.png)



---
title: Network Namespaces
author: Stack Underflow
date: '2021-09-03'
slug: network-namespaces
categories:
  - Tech
tags:
  - Linux
  - networking
description: the second post in series Kubernetes Networking Explained
---

## Recommended Resources

The video [Network Namespaces Basics Explained in 15 Minutes](https://youtu.be/j_UUnlVC2Ss) is a great introduction about how Linux network namespaces work.

## Set up the environment

We can set up two network namespaces connected to a bridge with the following commands:

```shell
# two namespaces
ip netns add red
ip netns add blue
# the bridge connecting the two namespaces
ip link add bridge0 type bridge
# the veth pair connecting red namespace to the bridge
ip link add veth-red type veth peer name veth-red-br
ip link set veth-red netns red
ip link set veth-red-br master bridge0
# turn the interfaces up
ip -n red link set veth-red up
ip link set veth-red-br up
# do the same for blue namespace
ip link add veth-blue type veth peer name veth-blue-br
ip link set veth-blue netns blue
ip link set veth-blue-br master bridge0
ip -n blue link set veth-blue up
ip link set veth-blue-br up
# turn the bridge up
ip link set bridge0 up
```

The namespaces `red` and `blue` are connected to a bridge, `bridge0`, like the diagram below.

![setup](images/setup.png)

## Network namespaces communication

How can the two network namespaces, `red` and `blue`, communicate with each other? The answer is: we need to assign IP addresses to them. The following commands assign 192.168.15.2/24 to `red` and 192.168.15.3/24 to `blue`.

```shell
ip -n red addr add 192.168.15.2/24 dev veth-red
ip -n blue addr add 192.168.15.3/24 dev veth-blue
```

`ping` to `blue` from `red` to verify that it works.

``` shell
ip netns exec red ping 192.168.15.3
```

How does it work? It turns out that a routing rule is added to the route table of `red` automatically when assigning it the IP address `192.168.15.2/24`

```shell
192.168.15.0/24 dev veth-red scope link
```

The routing rule has no gateway address, and the `scope` is set to `link`. This is exactly what the network prefix `/24` in the IP address 192.168.15.2/24 implies: All the addresses in the `192.168.15.0/24` range could be reached directly from interface `veth-red`, without going through any routers.

The same goes for network namespace `blue`, where the following routing rule is added automatically.

```shell
192.168.15.0/24 dev veth-blue scope link
```

## Network namespaces and host communication

To communicate with the host, we have to assign an IP address to the bridge0 interface.

```shell
ip addr add 192.168.15.1/24 dev bridge0
```

Now we can reach the network namespaces from the host
```shell
ping 192.168.15.2
```

and the host can be reached from the network namespace using IP address of the bridge.
```shell
ip netns exec red ping 192.168.15.1
```

This is exactly how docker [bridge network](https://docs.docker.com/network/bridge/) works. The difference is that the default Docker bridge is called `docker0` instead of `bridge0`.

## What is a virtual bridge?

The configuration becomes the following after IP addresses are added.

![setup with ip addresses](images/setup-withip.png)

You might be wondering why a bridge is added by `ip link` command, which is normally used to manage network interfaces. It turns out that a virtual bridge is a bridge + an interface. The setup above is equivalent to the following physical network setup.

![physical setup](images/setup-physical.png)

The name `bridge0` appears twice: It's not only the name of the bridge connecting `red`, `blue` and the host machine together, but also the name of the interface the is connect to the bridge on the host machine. In other words, it's a bridge from the perspective of the network namespaces, but it's an interface from the perspective of the host machine. When we add an IP address to `bridge0` using `ip addr add 192.168.15.1/24 dev bridge0`, we are actually adding an IP address to the interface. It's crucial to understand the "double role" of a virtual bridge.

## Access the Internet from network namespace

To access the Internet from `red`, first we add a default routing rule which sends non-local traffic to the host machine.

```shell
ip -n red route add default via 192.168.15.1 dev veth-red
```

We should also setup NAT in the host machine. According to [SNAT in iptables](/p/route-tables-and-iptables-for-routing/#snat-in-iptables), the rule should look like the following (`!` stands for negation).

```shell
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 ! -o bridge0 -j MASQUERADE
```

Then, run the following `ping` command to verify `red` has Internet access.

```shell
ip netns exec red ping 8.8.8.8
```

If it doesn't work, turn on the kernel parameter `net.ipv4.ip_forwarding` and try again.

```shell
sysctl -w net.ipv4.ip_forward=1
```

This is exactly how Docker containers access the Internet. Basically, we need 

- a default routing rule sending traffic to the host machine
- a MASQUERADE rule in the host machine setting up source NAT
- `net.ipv4.ip_forward=1`

Docker sets up the MASQUERADE rule and the parameter `net.ipv4.ip_forward=1` on installation, and the default routing rule is configured when a container is started.

### Hosts as routers

Why we need the kernel parameter `net.ipv4.ip_forwarding=1`? It turns out that it's because the host machine should act like a router. Remember that routers are devices that accept traffic that does not go directly to itself. The host is accepting traffic from 192.168.15.2 destining 8.8.8.8, but its own IP address is 172.16.94.12, neither the source address nor the destination address. The difference between the host and a regular router is that instead of routing traffic for physical machines, the host routes traffic for network namespaces running on themselves. 

`net.ipv4.ip_forward` is the parameter controlling whether the host could act like a router. To set the parameter temporarily, use the following command

```shell
sysctl -w net.ipv4.ip_forward=1
```

To set it permanently, add a file under `/etc/sysctl.d/` (like `/etc/sysctl.d/99-docker.conf`), and add the following line to this file

```shell
net.ipv4.ip_forward=1
```

## Publish a port

If we run a web server in `red` on port 80, how to access it from port 8080 on the host?

We need the following DNAT rule, according to [DNAT in iptables](/p/route-tables-and-iptables-for-routing/#dnat-in-iptables). We add it to both the PREROUTING and the OUTPUT chain so that it's accessible on both localhost and a remote machine.

```shell
iptables -t nat -A PREROUTING ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.15.2:80
iptables -t nat -A OUTPUT ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.15.2:80
```

This is exactly how Docker [publishes ports](https://docs.docker.com/config/containers/container-networking/#published-ports). You can run a container to experiment (like `docker run --rm -it -p 8080:80 nginx`)


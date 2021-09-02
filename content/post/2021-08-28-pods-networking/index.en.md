---
title: 'Pods Networking'
author: Stack Underflow
date: '2021-08-28'
slug: pods-networking
categories: 
  - Tech
tags: 
  - kubernetes
  - networking
description: the third post in series Kubernetes Networking Explained
---


## Networking requirements

The first requirement for Kubernetes networking is that containers running in the same pod could communicate with each other using localhost address.This is simple, we just put all the containers belonging to the same pod into one network namespace. It's implemented with a `pause` container which creates the network namespace, and other containers just attach themselves to it.

![intrapod](images/intrapod.png)

### Pods and their IP addresses

The diagram below shows how IP addresses are assigned to pods. Basically, a pod has a single unique IP address in Kubernetes, and this IP address is usually different from the IP address of the node that the pod is running on. For example, the pod `hello-world-1` is assigned an IP address 192.168.1.10 while running on node 172.16.94.11.

![pods](images/pods.png)

For a Kubernetes cluster, the pods IP addresses are allocated from a different address range than the nodes addresses. In the case of our example, the nodes IP address range is 172.16.94.0/24 but the pods address range is 192.168.0.0/16. The pods address range could be divided into subnets by nodes, for example, pods allocated on the control plane node are in the range 192.168.0.0/24, and `node1` serves the range 192.168.1.0/24.

You might notice that some pods has the same IP address as the underlying nodes, like the `kube-apiserver`, `kube-etcd` pods running on the control plane node (`cp`) and the `kube-proxy-*` pods running on all nodes. That is because they are special-purpose pods supporting the Kubernetes system and they have the [`hostNetwork`](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces) option turned on to share the host network namespace instead of creating a new one. Generally for pods deployed by ourselves, they will not use the host network.

## Pods commmunication

The most important requirement for pods networking is: *Pods should communicate with each other without NAT*.

What does it mean? For example, pod 1 with IP address 192.168.1.10 sends a packet to pod 2 192.168.2.10. In NAT-less communication, pod 2 sees the source address to be 192.168.1.10 instead of something else. In other words, the packet has not been NATed.

![natless](images/natless.png)

The NAT-less communication can be extended to nodes-to-pods communication. In other words, if a process running on a node (but outside of any pods) communicates with a pod, either on the same node or on another node, no NAT happens.

In the sections below, let's discuss how the NAT-less communication is implemented. Basically, we will discuss four implementations:

- switched network
- kubenet
- flannel
- calico

## Switched network

Let's start from the simplest case: switched network where all nodes are connected to a switch, and a router, also connected to the switch, is used to provide Internet access.

![switched network](images/switched-network.png)

The diagram above shows two worker nodes and two deployments. The deployment `frontend` has only one pods and the other deployment, `backend`, has two pods. The nodes are in IP address range 172.16.94.0/24 and the pods are in IP address range 192.168.0.0/16. The pods running on the same node are connected to a shared network bridge called `bridge0`, which is used for same-node pods communication.

### Same-node pods communication

The route table for the pods running on node1 looks like below.

```shell
192.168.1.0/24 dev eth0
default via 192.168.1.1 dev eth0
```

It's a typical route table configuration for [home network](/p/route-tables-and-iptables-for-routing#home-network-configuration). If `frontend-1` communicates with `backend-1`, the traffic is sent through `bridge0`. Otherwise the traffic is sent to the host. 

### Cross-nodes pods communication

The nodes are connected using the two route table rules

```shell
default via 172.16.94.1 dev eth0
172.16.94.0/24 dev eth0
```

If `frontend-1` communicates with `backend-2`, it goes through the outgoing rule 

```shell
192.168.2.0/24 via 172.16.94.12 dev eth0
```

which sends the traffic to node2. The incoming rule on node2 sends the traffic to `bridge0`

```shell
192.168.2.0/24 dev bridge0
```

In this way, the two pods communicate with each other without using NAT. The rule `192.168.2.0/24 via 172.16.94.12 dev eth0` only enables communication between pods on node1 and node2. If another node, say, node3 is added, which serves pods 192.168.3.0/24 with IP address 172.16.94.13, we need another rule `192.168.3.0/24 via 172.16.94.13 dev eth0`. Generally, for a cluster with `n` nodes, each node has `n-1` outgoing rules for all of its neighbors.

Also note that nodes-to-pods communication also goes through the outgoing and incoming routing rules, so nodes and pods could communicate without NAT as well. It doesn't mean we do not do NAT at all. NAT also happens when a pod access something outside of the nodes and pods network, for example, the Internet.

### Hosts as routers

What is interesting is that node2 is accepting traffic destining 192.168.2.0/24, but its own IP address is 172.16.94.12. it's handling traffic that does not go to its own IP address. In other words, node2 acts like a router (remember that routers are devices that accept traffic that does not go directly to itself) even though it's actually a host. The same goes for node1, and any other nodes in this Kubernetes cluster. The difference between those nodes and regular routers is that instead of routing traffic to physical machines, they route traffic to network namespaces running on themselves. 

As a result, nodes in a Kubernetes cluster should set the kernel parameter `net.ipv4.ip_forward` to 1 to enable working like routers. This applies not only to the switched network setup, and not only to Kubernetes. This option has to be set to 1 if we would like to run containers on a host (e.g. use Docker to develop apps locally) otherwise the container cannot communicate with the outside world.

A lot of Linux distributions have this parameter default to 1, but if it's not 1, we need to set it manually. To set the parameter temporarily, use the following command

```shell
sysctl -w net.ipv4.ip_forward=1
```

To set it permanently, add a file under `/etc/sysctl.d/` (like `/etc/sysctl.d/99-kubernetes-cri.conf`), and add the following line to this file

```shell
net.ipv4.ip_forward=1
```

## Kubenet

The switched network configuration only works fine for small clusters, small enough that all the machines can be switched together. What should we do for larger clusters? One possible optimization for kubenet you might be thinking is moving all the routing rules like `192.168.2.0/24 via 172.16.94.12 dev eth0` to the router `172.16.94.1` to manage them centrally. In fact, this is what kubenet does, with the difference that we usually use kubenet only on cloud clusters.

![kubenet](images/kubenet.png)

From the diagram above, we see that the outgoing rules have been removed from each node. Instead, a centralized routing table is added to the nodes subnet to enable cross-nodes pods address routing.

```shell
192.168.2.0/24 -> 172.16.94.12
192.168.1.0/24 -> 172.16.94.11
```

The route tables is provided by the cloud provider's networking infrastructure. That is why kubenet works only on cloud environments.

Similar to the switched network configuration, NAT is required to access something outside of the nodes subnet, like a database node at 172.16.150.120. Generally, only pods and node-to-pods communication are NAT-less in all Kubernetes cluster configurations.

If you want to do experiment with kubenet you could create a Kubernetes cluster on Azure, which provides kubenet networking mode.

### Flannel

Kubenet works only on cloud environment and it works well only for small or medium clusters. What if we are deploying a Kubernetes cluster on premise or the cluster is too large to use kubenet?

One possible solution is flannel. It works like the diagram below.

![flannel](images/flannel.png)

Here I omitted the host network routing rules since they are no longer important. `flannel0` is a TUN device created by `flanneld`. It look like an interface for the host, just like `bridge0` and `eth0`. But it has some special behaviors:

- If the route table routes a packet to `flannel0`, the packet will be handed to `flanneld`
- If `flanneld` writes a packet to `flannel0`, the host treats it like an incoming packet from `flannel0`, just like an incoming packet from other interfaces like `eth0`.

flannel packs every IP packets in an UDP datagram and sends them to the destination node. For example, if `frontend-1` sends a packet to `bakcend-2`, the steps will be:

1. the packet is routed to `flannel0` interface on node1 since it matches the rule `192.168.0.0/16 dev flannel0`.
2. `flannel0` hands the packet over to `flanneld`, which packs the packet in an UDP datagram. The destination port is set to 8285, while the source and destination IP addresses are set to the addresses of node1 and node2. flannel stores a mapping from pod network ranges to node IP addresses in etcd, so it's able to determine that 192.168.2.10 runs on 172.16.94.12 (node2) by looking up this mapping.
3. the UDP datagram is transmitted to node2. Note that this transmission does not depend on any underlying network architecture as long as node2 is reachable from node1. They could be switched together, or they could be very far away and the traffic has to go through several routers along the way. They could even be virtual machines running on the cloud and thus the networking architecture is a blackbox to us. That is why we omit the host network routing rules in the diagram above.
4. the UDP datagram reaches node2 at port 8285. The `flanneld` process also listens on port 8285 so it gets the datagram.
5. `flanneld` unpacks the UDP datagram and writes it to `flannel0`. The inner IP packet destining 192.168.2.10 gets revealed and routed to `bridge0` by the rule `192.168.2.0/24 dev flannel0`. Note that it will not match the `flannel0` rule because of longest prefix matching.
6. `bridge0` send the packet to `backend2` whose address matches the destination address of the packet, 192.168.2.10.

The diagram below shows how the packet gets packed and unpacked.

![packet flannel](images/flannel-packet.png)

This is how the default backend, the `udp` backend, works. flannel also provides a `vxlan` backend, which achieves better performance by doing all the UDP packing and unpacking stuff in the kernel instead of an user-space process `flanneld`.

## Calico






---
title: Kubernetes Networking Explained
author: Stack Underflow
date: '2021-09-01'
slug: kubernetes-networking-explained
categories: 
  - Tech
tags: 
  - kubernetes
  - networking
  - linux
image: images/future-world.jpg
description: a blog post series
---

This is a series of blog posts explaining how Kubernetes networking behaves. 

## Posts

- [Route Tables Explained](/p/route-tables-explained/)
- [Iptables for Routing](/p/iptables-for-routing/)
- [Network Namespaces and Docker](/p/network-namespaces-and-docker/)
- [Pods Networking](/p/pods-networking/)
- Services Networking (todo)
- DNS (todo)

## Resources

### General

The Pluralsight course [Configuring and Managing Kubernetes Networking, Services, and Ingress](https://app.pluralsight.com/library/courses/configuring-managing-kubernetes-networking-services-ingress/table-of-contents) is a good introduction to Kubernetes networking.

The following videos are great overview 
- [技术分享：Kubernetes Networking Model （赵锟）](https://youtu.be/HxS4s11rmyA)
- [Tutorial: Communication Is Key - Understanding Kubernetes Networking - Jeff Poole, Vivint Smart Home](https://youtu.be/InZVNuKY5GY)

### Networking basics

The video series below explains iptables and other networking related topics in great detail.

- [金枪鱼之夜：坏人的 iptables 小讲堂](https://youtu.be/w_vGD-96O54)
- [金枪鱼之夜：坏人的 iptables 小讲堂第二弹](https://youtu.be/Vnh8hYk6wZE) 

### Network namespaces

[Network Namespaces Basics Explained in 15 Minutes](https://youtu.be/j_UUnlVC2Ss) is a great introductory video on network namespaces. However there are several mistakes in this video so pay attention.

### Pods networking

[Kubernetes: Flannel networking](https://blog.laputa.io/kubernetes-flannel-networking-6a1cb1f8ec7c)

[Calico Routing Modes](https://youtu.be/MpbIZ1SmEkU)

### Services networking

[Kubernetes Services and Iptables](https://msazure.club/kubernetes-services-and-iptables/)

### DNS

[Understanding CoreDNS in Kubernetes](https://youtu.be/qRiLmLACYSY)

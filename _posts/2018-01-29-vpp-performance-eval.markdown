---
layout:     post
title:      Vector Packet Processing (VPP)
subtitle:   Comparing Linux networking stack with VPP Datapth
date:       2018-1-28
author:     Fan Zhongyi
catalog: true
tags:
    - Cisco
    - VPP
    - Openvswitch
    - Namespace
---
# Vector Packet Processing (VPP): Pushing performance limits

## Abstract

Cisco's Vector Packet Processing (VPP) technology has a proven record of advanced features: high performance
and a packet-processing stack that can run on commodity CPUs.
The VPP platform is an extensible and open source framework that provides out-of-the-box production quality
switch or router functionality.
Due to its modularity and flexibility,
more and more products have been implemented with VPP to achieve better I/O packet processing performance.

This report will mainly introduce the principle of VPP, compare original Linux datapath with VPP datapath,
show VPP working process and then implement VPP with an experimental network
to evaluate performance versus Openvswitch and veth pairs.

After analysis of the results, it can be confirmed that integrating VPP into the network system can
help to achieve a better performance when comparing with openvswitch and veth pairs in same experimantal
network environment. Furtherly, it is a promising future direction to add DPDK
(Data Path Development Kit) as a graph node inside VPP framework to push performance limits.

## Linux networking stack

To understand VPP working process, it would be better to compare Linux networking stack with VPP datapath.

It is known that Linux is designed as an Internet host to the concept of an “End-system” used in the OSI
suite. The Linux networking kernel code is a very large part of the Linux kernel code. Below Figure shows 
packet walkthrough in the kernel.

![linux-networking-stack.png](https://raw.githubusercontent.com/fzyitt/fzyitt.github.io/master/img/linux-networking-stack.png)

There are two most important structures of Linux kernel network layer, sk\_buff and netdevice. sk\_buff
represents data and headers. In the user space from the application layer will send a request to lower
layers to create socket to forward packets in kernel space. When session layer receives the request,
it will create socket and handle memory data, until it finds socket buffer with enough space. Then
transport layer will copy from user space to kernel space. After routing and filtering in network layer,
data will be output to the data link layer drivers. In the end, it will be received by the network 
interface card. The receiving process will just be the opposite. 
  
All the packets should pass in the kernel one by one like this, which is called scalar processing. This kind
of packet processing occurs trashing in the I-cache and each packet incurs an identical set of I-cache 
missing. Classical stack design allows system to serve multiple applications since application is seperated
from sensitive kernel code and its structure is quite simple. Due to these features, nowadays networking 
stack takes a huge part of kernel. The code is so generic and every change in the protocol requires new 
kernel version update. Some considerations are brought up to move redundency from kernel space to user space. 

## VPP Datapath

VPP is a key open source project of FD.io donated by Cisco in Linux foundation project. Different from
the classical packet processing, as its name implies, VPP uses vector processing not scalar processing.
Vector processing can process more than one packet in a vector at a time. Each vector just need to hit
cache once, which will exactly fix the I-cache trashing problem and improve circuit time. Especially
as the vector size increases, processing cost per packet decreases by passing the vector through directed
graph node of nodes.

![vpp-datapath.png](https://raw.githubusercontent.com/fzyitt/fzyitt.github.io/master/img/vpp-datapath.png)

It is obviously from figure above that the datapath of VPP emphasizes on the user space. The whole
packet process is independent from the kernel. It can be seen that VPP includes full scale Layer 3
and Layer 2 stack, which means fully routing and switching functionality. For the I/O calling part,
VPP employs polling mode. Polling mode will consumes higher CPU usages but when dealing with burst
packet traffic like a large size packet vector, it spends less circuit time than interrupt mode.

## Conclusion

VPP is a very young and promising project and at present it can not only fullfill almost all the functions
in layer 2 and layer 3, it also has been integrated with Openstack, Opendaylight and Kubernetes projects.

It has been tested by Cisco that VPP has a better performance than other existing packet processing
schemes. In the following parts of the report an experimental network will be set up to show real VPP
working process and test the performance of VPP as well as Openvswitch and Veth pairs.







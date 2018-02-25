---
layout:     post
title:      Openstack DPDK Deployment
subtitle:   Migrated from my Repo on Bitbucket thus in English
date:       2017-12-19
author:     Fan Zhongyi
#header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Openstack
    - DPDK
    - Devstack
    - HPC
---

# Deploy Openstack integrated with DPDK by using Devstack
This wiki describes how to set up a network coding demonstration for Service Function Chaining (SFC) and Data Plane Development Kit (DPDK) to run together on OpenStack, using DevStack as the deployment tool and the Neutron ML2/GRE Tunnel plugin.
## System requirements
* Devstack in version **stable/pike**
* One Control node and three compute nodes and each should have two network interfaces
* All nodes should have at least one [DPDK-supported network interfaces](http://dpdk.org/doc/nics) 
* System must support VT-d,VT-x and both should be enabled in BIOS
## Operating system
* Ubuntu 17.04
## Installation
### Design the network
![design_network.png](img/design_network.png)
### Prepare for Devstack
#### 1. Prepare the system

```
#!shell
	# install required packets
	sudo apt update && sudo apt -y upgrade
	sudo apt-get install -y openssh-server git vim
```	

#### 2. Set up the network

```
#!shell	
	# disable NetworkManager and enable networking
	sudo systemctl stop NetworkManager
	sudo systemctl disable NetworkManager
	
	# define the network interface in static
	sudo vim /etc/network/interfaces
	# add:
	#auto <net-iface>
	#iface <net-iface> inet static
	#	address xxx.xxx.xxx.xxx
	#	netmask xxx.xxx.xxx.xxx
	#	gateway xxx.xxx.xxx.xxx
	#	dns-nameserver xxx.xxx.xxx.xxx
	
	# retstart networking service
	sudo systemctl retstart networking
```	

#### 3. Create a non-root user 'stack'

```
#!shell	
	# create 'stack' and /home/stack 
	sudo useradd -m stack
	
	# set the password
	sudo passwd stack
	
	# set the user group and  authority
	sudo usermod -aG root stack
	echo "stack ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```	

#### 4. Git clone devstack in version stable/pike

```
#!shell	
	cd /home/stack/
	git clone https://github.com/openstack-dev/devstack.git -b stable/pike
```	

#### 5. Configure the local.conf, samples can be found in [Repo](https://bitbucket.org/comnets/dpdk-openstack/src/768f3d845896/devstack_conf/?at=master)

```
#!shell	    
	cd /home/stack/devstack
	cp xxx_node.conf /home/stack/devstack/local.conf
```	

* Make sure to change network parameters in local.conf for your own situation

#### 6. Stack

```
#!shell	
	su stack
	cd /home/stack/devstack
	./stack.sh
```	

### DPDK setting
With the plugin [networking-ovs-dpdk](https://github.com/openstack/networking-ovs-dpdk) default setting, we will probably not be able to lauch the instance after successfully stack. We should make some modifications in the settings

```
#!shell	
	cd /opt/stack/networking-ovs-dpdk/devstack/
	sudo vim settings
	sudo vim /ovs-dpdk/ovs-dpdk-conf 
	# change OVS_SOCKET_MEM = ${OVS_SOCKET_MEM:-auto} -> OVS_SOCKET_MEM = ${OVS_SOCKET_MEM:-128}
	# change OVS_SOCKET_MEM to smaller one, with auto it will claim for 2048Mb Hugepage memory.
	
	sudo vim plugin.sh
	# change ${OVS_DB_SOCKET_DIR}/${OVS_VHOST_USER_SOCKET_DIR} -> /tmp
	# we do not have the permission to lauch the vhost in default location, set to /tmp to get the permission.
	
	# Then stack again
	./unstack.sh
	./stack.sh
```	

* After successfullt running openstack on Control node, do the same steps with specific local.conf on Compute nodes

### Lauch the instance

```
#!shell	
	# discover the compute host on 
	cd /home/stack/devstack/
	./tools/discover_hosts.sh

	# log into Horizon and download Openstack RC file v3 and source the file
	. /home/control/Downloads/demo_openrc.sh
	# inut password
	
	# set flavor as Hugepage flavor, here we choose m1.tiny
 	openstack flavor set m1.tiny --property hw:mem_page_size=large
```

* Then we set the security groups rules, add rules of ALL ICMP and ALL TCP ingress and egress in all CIDR
* Last step is to lauch the instance, make sure to choose the m1.tiny we have pre-defined as the flavor
	 

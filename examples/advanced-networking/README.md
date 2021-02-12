# Advanced Networking for Virtualization

## Table of Contents

<!-- TOC -->
- [Advanced Networking for Virtualization](#advanced-networking-for-virtualization)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Creating a nodeNetworkConfigPolicy](#creating-a-nodenetworkconfigpolicy)
  - [Connecting a VM to your bridge network](#connecting-a-vm-to-your-bridge-network)
  - [Booting from the Network](#booting-from-the-network)
<!-- TOC -->

## Introduction

OpenShift uses the network plugin Multus to allow for multiple network connections to containers, as well as to virtual machines. 


## Creating a nodeNetworkConfigPolicy

In order to create a bridge which can be used to connect to networks. Lets take a look at the example nodeNetworkConfigPolicy.


## Connecting a VM to your bridge network


## Booting from the Network

Now that we have a bridge network that we can use to attach a virtual machine directly to your existing networking infrastructure we can boot from the network using [PXE - Preboot eXecution Environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment). We will not be setting up a PXE environment in this lab, but if you have support for it in your network, it is possible to create a VM within OpenShift virtualization and boot it from your network.

For this lab we will assume that "VLAN10", supports PXE and is properly configured to boot a machine with a random MAC address. If you need to specify a MAC address to allow for PXE booting in your environment update the examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml by uncommenting the "macAddress" field and specifying the MAC address you wish to use.

We can now create the VM by applying the YAML:

```
oc create -f examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml
# connect to the console of the VM and see that it is now booting over the network
virtctl console vm-pxeboot
```
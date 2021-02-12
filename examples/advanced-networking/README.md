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

OpenShift uses the network plugin Multus to allow for multiple network connections to containers, as well as to virtual machines. This README file will concentrate on the virtual machine use cases. The document assumes that you have configured your worker nodes as follows:

* each worker node has TWO network interfaces
* the primary network interface is attached to your primary network when you built your OpenShift cluster
* the secondary network interface is trunked to your upstream switch which allows vlan tagging
* the examples will use VLANs 10, and 15 however you can substitute any VLAN numbers that you may use in your configuration

## Creating a nodeNetworkConfigPolicy

We will start by creating a "NodeNetworkConfigurationPolicy" which tells OpenShift how to configure a bridge network on worker nodes. To start we will label the worker nodes that have the proper network connection. List out your nodes, and for each node, use the oc command to label the node with "networking.config/bridgenabled=true". We will use this label to apply the NodeNetworkConfigurationPolicy to only the appropriate machines.

```
$ oc get nodes
NAME      STATUS   ROLES    AGE   VERSION
master0   Ready    master   25d   v1.19.0+1833054
master1   Ready    master   25d   v1.19.0+1833054
master2   Ready    master   25d   v1.19.0+1833054
sriov0    Ready    worker   24d   v1.19.0+1833054
worker0   Ready    worker   25d   v1.19.0+1833054
worker1   Ready    worker   25d   v1.19.0+1833054
worker2   Ready    worker   25d   v1.19.0+1833054
$ oc label nodes <nodeName> networking.config/bridgenabled=true
```

Now that the nodes are labeled properly, we will apply our NodeNetworkConfigurationPolicy. Edit the examples/advanced-networking/nodeNetworkConfigPolicy.yml and be sure to update the device name of your secondary network card, if it is not ens224. Save the file and apply with

```
$ oc create -f examples/advanced-networking/bridge-networking/nodeNetworkConfigPolicy.yml
$ oc get NodeNetworkConfigurationPolicy
NAME              STATUS
br1-eth1-policy   SuccessfullyConfigured
```

We will now create a "NetworkAttachmentDefinition" which is created on a project/namespace level. You must have a NetworkAttachmentDefinition defined in your target namespace before you can use it. We will apply the vlan15Bridge.yaml file to the same project we used for the OpenShift Virtualization lab called "demovms". If you want to use a different VLAN be sure to update the file before applying it with your target vlan number.

```
$ oc project demovms
$ oc create -f examples/advanced-networking/bridge-networking/vlan15Bridge.yml
$ oc get network-attachment-definition
NAME     AGE
br1-10   148m
```

You OpenShift cluster is now configured with a bridge connection to your layer-2 network and a connection can be made within the demovms project. Let's see how we can use this.

## Connecting a VM to your bridge network

This demo works best if you have DHCP on your target vlan so you can see that the secondary connection comes on line and properly configures to your layer-2 network, but you can manually configure a network on the secondary interface to test if you do not have DHCP.

To start a vm with a secondary network interface, review the configuration in examples/advanced-networking/fedora-brnet.yml and update if you made any changes when creating the target vlan and then apply the Yaml to your cluster. We will then connect to the console and log in as "fedora:fedora" and run a few commands to see that you have a second interface and it is available on your target vlan.

```
$ oc create -f examples/advanced-networking/fedora-brnet.yml
$ oc get vmi
# Wait until vmi shows running
$ virtctl console vm-fedora-brnet
# login using fedora:fedora
$ ip a
# note that there are two network interfaces
# and that one interface has an IP on the OpenShift SDN
# the second interface has an IP address assigned from the DHCP server on your target vlan
```

## Booting from the Network

Now that we have a bridge network that we can use to attach a virtual machine directly to your existing networking infrastructure we can boot from the network using [PXE - Preboot eXecution Environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment). We will not be setting up a PXE environment in this lab, but if you have support for it in your network, it is possible to create a VM within OpenShift virtualization and boot it from your network.

For this lab we will assume that "VLAN10", supports PXE and is properly configured to boot a machine with a random MAC address. If you need to specify a MAC address to allow for PXE booting in your environment update the examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml by uncommenting the "macAddress" field and specifying the MAC address you wish to use.

We can now create the VM by applying the YAML:

```
oc create -f examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml
# connect to the console of the VM and see that it is now booting over the network
virtctl console vm-pxeboot
```
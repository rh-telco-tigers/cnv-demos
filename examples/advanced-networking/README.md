# Advanced Networking for Virtualization

## Table of Contents

<!-- TOC -->
- [Advanced Networking for Virtualization](#advanced-networking-for-virtualization)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Creating a nodeNetworkConfigPolicy](#creating-a-nodenetworkconfigpolicy)
  - [Connecting a VM to your bridge network](#connecting-a-vm-to-your-bridge-network)
  - [Booting from the Network](#booting-from-the-network)
  - [SRIOV Support](#sriov-support)
    - [Configuring your cluster for SRIOV](#configuring-your-cluster-for-sriov)
      - [Enabling Unsupported Cards](#enabling-unsupported-cards)
    - [Using SRIOV device in a virtual machine](#using-sriov-device-in-a-virtual-machine)
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

An alternative method of applying the network configuration on only the worker nodes is by specifying their role as nodeSelector in the NodeNetworkConfigurationPolicy manifest - 

```
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

Now the NodeNetworkConfigurationPolicy manifest can be applied on the cluster. Edit the examples/advanced-networking/bridgeNetworkConfigPolicy.yml and be sure to update the device name of your secondary network card, if it is not ens224. Save the file and apply with

```
$ oc create -f examples/advanced-networking/bridge-networking/bridgeNetworkConfigPolicy.yml
$ oc get NodeNetworkConfigurationPolicy
NAME              STATUS
br1-eth1-policy   SuccessfullyConfigured
```

We will now create a "NetworkAttachmentDefinition" which is created on a project/namespace level. You must have a NetworkAttachmentDefinition defined in your target namespace before you can use it. We will apply the vlan15Bridge.yml file to the same project we used for the OpenShift Virtualization lab called "demovms". If you want to use a different VLAN be sure to update the file before applying it with your target vlan number.

```
$ oc project demovms
$ oc create -f examples/advanced-networking/bridge-networking/vlan15Bridge.yml
$ oc get network-attachment-definition
NAME     AGE
br1-10   148m
```

You OpenShift cluster is now configured with a bridge connection to your layer-2 network and a connection can be made within the demovms project. Let's see how we can use this.

Note: In addition to bridge interface, you can configure the cluster nodes' network with other interfaces like Ethernet, Bond, VLAN, etc. using the NodeNetworkConfigurationPolicy manifest. 

An example of creating a bond interface connecting two node network interfaces, which in turn is attached to a bridge interface is given in examples/advanced-networking/bondNetworkConfigPolicy.yml. You would need two or more additional network interface cards on the nodes for creating such a configuration. Make sure to update the device names of the network interface cards in the yaml. 

## Connecting a VM to your bridge network

This demo works best if you have DHCP on your target vlan so you can see that the secondary connection comes on line and properly configures to your layer-2 network, but you can manually configure a network on the secondary interface to test if you do not have DHCP. You can also assign a static IP to the secondary interface either via 
(1) the NetworkAttachmentDefinition, as seen in vlan15BridgeStaticIP.yml, or
(2) the cloud-init, as shown below -

```
kind: VirtualMachine
spec:
...
  volumes:
  - cloudInitNoCloud:
      networkData: |
        version: 2
        ethernets:
          eth1: 
            addresses:
            - 10.10.10.14/24 
```

Note that when assigning the IP via a NetworkAttachmentDefinition, only one vm can be attached using that definition.
Also ensure that while assigning a static IP, the IP address falls in the machine CIDR range defined for the cluster.

To start a vm with a secondary network interface, review the configuration in examples/advanced-networking/fedora-brnet.yml and update if you made any changes when creating the target vlan and then apply the Yaml to your cluster. We will then connect to the console and log in as "fedora:fedora" and run a few commands to see that you have a second interface and it is available on your target vlan.

```
$ oc create -f examples/advanced-networking/fedora-brnet/fedora-brnet.yml
$ oc get vmi
# Wait until vmi shows running
$ virtctl console vm-fedora-brnet
# login using fedora:fedora
$ ip a
# note that there are two network interfaces
# and that one interface has an IP on the OpenShift SDN
# the second interface has an IP address assigned from the DHCP server on your target vlan
```

You can now connect to the vm using the second interface's IP as well. Additional vms can be attached to the same bridge network to facilitate secure communication amongst them.

## Booting from the Network

Now that we have a bridge network that we can use to attach a virtual machine directly to your existing networking infrastructure we can boot from the network using [PXE - Preboot eXecution Environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment). We will not be setting up a PXE environment in this lab, but if you have support for it in your network, it is possible to create a VM within OpenShift virtualization and boot it from your network.

For this lab we will assume that "VLAN10", supports PXE and is properly configured to boot a machine with a random MAC address. If you need to specify a MAC address to allow for PXE booting in your environment update the examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml by uncommenting the "macAddress" field and specifying the MAC address you wish to use.

We can now create the VM by applying the YAML:

```
oc create -f examples/advanced-networking/vm-pxe-boot/pxeboot-brnet.yml
# connect to the console of the VM and see that it is now booting over the network
virtctl console vm-pxeboot
```

## SRIOV Support

OpenShift has support for [SR-IOV - Single Root - Input Output Virtualization](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization) network cards. Support for these cards can be managed using an Operator. The following steps will enable SR-IOV and in subsequent steps we will connect a virtual machine to these cards.

### Configuring your cluster for SRIOV

Start by creating a new namespace (not project) called "openshift-sriov-network-operator”
This can be done from the UI by going to “Administration->Namespaces” and clicking create OR by running the following command: 


```
$ cat << EOF| oc create -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-sriov-network-operator
EOF
```

Once you have created the namespace, log into the OpenShift console and install the “SR-IOV Network Operator”
1. Log into the console
2. Select Operators->Operator Hub
3. Search for "SR-IOV Network Operator" and select it
4. Click install
5. Ensure that you install it to the "openshift-sriov-network-operator" namespace you created earlier
6. Wait for the operator install to complete
7. 

We can now check on the status of the operator to see if it was able to identify SRIOV devices. We will be using the command line for the next set of steps:

```
$ oc login <cluster>
$ oc project openshift-sriov-network-operator
$ oc get SriovNetworkNodeState
# ensure that there is a SriovNetworkNodeState for each machine in your cluster)
# for each node that you think will have an SRIOV device run:
$ oc describe SriovNetworkNodeState/<nodeName>
```

You will see a list of all network devices detected. You are looking for a device to be detected that lists “Totalvfs:   <some number>” where some number is the total number of SRIOV devices that the network card supports. If you do not see any cards identified but you are sure your card does have SRIOV support you can enable unsupported card discovery by following the steps in the section below called [Enabling Unsupported Cards](#enabling-unsupported-cards)

Review the file examples/advanced-networking/intel-sriov-node-network.yml file and update the following fields to match the cards you have discovered.

* metadata.name
* spec.numVfs
* spec.nicSelector.vendor
* spec.nicSelector.deviceID

Once you update the Yaml, apply this to your cluster. Note that the nodes that have SRIOV cards *will reboot*. When the reboot is complete, re-run the `oc describe SriovNetworkNodeState/sriov0` command and you should now see a list of SRIOV devices.

#### Enabling Unsupported Cards

If your network card is not listed as a supported card you will need to run the following:

```
oc patch sriovoperatorconfig default --type=merge \
-n openshift-sriov-network-operator \
--patch '{ "spec": { "enableOperatorWebhook": false } }'
```

*NOTE THIS MEANS YOUR NETWORK CARD IS NOT SUPPORTED, DON'T CALL FOR HELP.*

### Using SRIOV device in a virtual machine

Now that we have properly configured SR-IOV for your cluster, we will tell the SR-IOV Operator to create a NetworkAttachmentDefinition in the demovms project for us to consume. Review the file examples/advanced-networking/sriov-networking/vlan15sriov.yml and ensure that the proper target vlan is defined and then apply it to your cluster

```
$ oc create -f examples/advanced-networking/sriov-networking/vlan15sriov.yml
$ oc get network-attachment-definition
# ensure that your new network is created
```

We can now use the template located in examples/advanced-networking/fedora-sriov/fedora-sriov.yml to create a vm that will leverage a SR-IOV network card virtual function to directly attach to your layer-2 network.

```
$ oc create -f examples/advanced-networking/fedora-sriov/fedora-sriov.yml
$ oc get vmi
# Wait until vmi shows running
$ virtctl console vm-fedora-brnet
# login using fedora:fedora
$ ip a
# note that there are two network interfaces
# and that one interface has an IP on the OpenShift SDN
# the second interface has an IP address assigned from the DHCP server on your target vlan
```

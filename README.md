# Kicking the tires on OpenShift Virtualization

So you are trying out OpenShift Virtualization, but don't know where to start. Here are a few different examples to see what you can do with OpenShift Virtualization. This document will evolve over time to add more things to try out, so keep checking back to see some of the new demos that you can run.

## Prerequisites

This README assumes that you already have OpenShift Virtualization installed, and you have storage avaialble to you. Please see [Installing OpenShift Virtualization](https://docs.openshift.com/container-platform/4.6/virt/install/preparing-cluster-for-virt.html) for instructions on how to prepare your cluster and install OpenShift Virtualization.

You will need the following tools installed on your machine to interact with the OpenShift cluster as well as the OpenShift Virtualization operator.
* oc - 
* virtctl - This tool can be downloaded from the [Red Hat Customer Support Portal](https://access.redhat.com/downloads/content/473/ver=2.5/rhel---8/2.5.3/x86_64/product-software)

You will need to have a cloned copy of this repo to use the example files. The instructions below will assume you have a local copy of the files that you can edit before applying the yaml.

```
$ git clone https://github.com/rh-telco-tigers/cnv-demos.git
$ cd cnv-demos
```

## Your first VM - Fedora VM from Container Disk

The easiest way to kick off a VM inside OpenShift is leveraging a container disk and a Virtual Machine Instance (or vmi). A containerdisk is a way of storing a base virtual machine disk image in a container registry. We will be using a pre-made containerDisk that has a Fedora base install in it from quay.io.

Using your favorite editor, open the file `examples/fedora-cd/fedora-ephemeral.yml` and update line 44 with your ssh key. 

Now we will create our new vm from the fedora-ephemeral.yml file.

```
# start by logging into the cluster
oc login <openshift cluster url>
# next we create a new project (similar to k8s namespace) called demovms to hold our virtual machine
$ oc new-project demovms
# finally we apply yaml to create the vm
$ oc create -f examples/fedora-cd/fedora-ephemeral.yml
```

We can now check on the status of the VM using the oc and virtctl commands

```
oc get vmi
# once the vmi shows running we can connect to the console using the virtctl command
virtctl connect vmi-fedora-1
# login with 

```

## Making a persistent Fedora VM

The first example VM we created was an ephemeral VM. Just like a container deployed in OpenShift, if the container disappears or is restarted, any data that was stored with that container also goes away. This is not a typical scenario when it comes to running virtual machines, and most times you will expect your vm to survive a reboot or a shutdown. To do this, we will build upon our last fedora deployment but this time we will leverage a dataVolume to create a persistent disk image from the base image we import from the container disk.

Be sure to update line 50 in `examples/fedora-cd/fedora-persistent.yml` with your SSH key for later use before proceeding.

```
# start by logging into the cluster
oc login <openshift cluster url>
# next we create a new project (similar to k8s namespace) called demovms to hold our virtual machine
$ oc new-project demovms
# finally we apply yaml to create the vm
$ oc create -f examples/fedora-cd/fedora-persistent.yml
```

We can now check on the status of the VM using the oc and virtctl commands

```
oc get vm
oc get vmi
# once the vmi shows running we can connect to the console using the virtctl command
virtctl connect vmi-fedora-2
# login with 

```

## Exposing a service from a VM

Just like apps running inside a container, we can expose applications/services from a running VM. We will create a new VM and expose a very simple "hello world" service from this VM in this demo.




## Cloning a VM

Now that we have a persistent fedora virtual machine, we can clone that vm and make multiple copies of it.

## Importing a VM from vSphere

It is possible to import an existing virtual machine from a vSphere cluster. There are a few caveats to be aware of before attempting this import.
1. the vm you wish to import must be powered off. Importing running vms is not supported at this time
2. drivers 

## Deploying a Windows VM from ISO

### Create new VM

We will start by uploading a ISO boot cd to our cluster. We will use the virtctl command to upload the ISO image. You will need to supply your own copy of the Windows 10 install ISO. The steps below can also be used to install similar Microsoft based operating systems such as Windows Server 2012r2, Windows Server 2016 and Windows Server 2019.

```
$ virtctl image-upload --uploadproxy-url=https://cdi-uploadproxy-openshift-cnv.apps.ocp4rhv.example.com dv iso-win10-dv --size=4Gi --image-path=/home/markd/en_windows_10_multiple_editions_x64_dvd_6846432.iso --insecure
```

Using the file "templates/win10vm1-pvc.yaml" create a PVC to store your virtual machine hard disk on, updating your required disk size and storageClass you want to use.

Create a VirtualMachine instance from yaml using file "templates/win10vm1.yaml". Be sure to update the device name under GPU for the name you gave your GPU in the kubevirt-cr configuration step.

From the console, power on the VM and follow standard install procedures for your Windows 10 OS. If you are using virtio for the hard disk controller you will need to follow the steps outlined [here](https://kubevirt.io/user-guide/#/creation/virtio-win?id=how-to-install-during-windows-install) but be sure to select the "w10" directory to load the proper drivers.

### Accessing a Windows VM directly via RDP

If you enable RDP from within the WIndows VIrtual machine, it is possible to directly connect to that VM using RDP through the exposure of the RDP service via a NodePort service.

```
$ virtctl expose virtualmachine win10vm1 --name windows-app-server-rdp --port 3389 --target-port 3389 --type NodePort
```


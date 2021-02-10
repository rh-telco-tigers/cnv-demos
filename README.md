# Kicking the tires on OpenShift Virtualization

## Table of Contents

<!-- TOC -->
* [Introduction](#introduction)
* [Prerequisites](#architecture)
* [Prerequisites](#prerequisites)
* [Your First VM](#your-first-vm)
  * [Authenticate gcloud](#authenticate-gcloud)
  * [Setup this project](#setup-this-project)
  * [Cluster Deployment](#cluster-deployment)
  * [Provisioning the Kubernetes Engine Cluster](#provisioning-the-kubernetes-engine-cluster)
  * [Accessing the Private Cluster](#accessing-the-private-cluster)
* [Guided Demos](#guided-demos)
* [Teardown](#teardown)
* [Troubleshooting](#troubleshooting)
<!-- TOC -->

## Introduction

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

## Your first VM 

**Fedora VM from Container Disk**

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
virtctl console vm-fedora-ephemeral
# login using fedora:fedora 
# Run dmesg to see this is a fully functioning vm
dmesg
# logout and disconnect from the console
exit
# press CTRL + ] to disconnect from console
```

### Using the OpenShift UI

The OpenShift UI supports visualization of your running VMs, including metrics, and connecting to the console. To see and explore this do the following:

1. Log into the OpenShift Console (https://console-openshift-console.apps.\<your cluster dns name\>/)
2. Select "Workloads" from the left hand side
3. Select "Virtualization"
4. Use the "Project Drop down" selector to select the "demovms" project
5. Select "vm-fedora-ephemeral" 
6. Review the information shown on the Overview page. 
   1. Things like "Utilization" will start to populate after the vm has been running for a few minutes
7. Select "Console"
8. Log into the console with "fedora:fedora"
9. Have fun with your ephemeral VM

Lets delete this VMI and move onto a more permanent virtual machine. (NOTE: if you have the OpenShift console still open, you will see the VM dissapear from the console when you run this command)

```
$ oc delete vmi/vm-fedora-ephemeral
$ oc get vmi
No resources found in demovms namespace.
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
oc describe vm/vm-fedora-persist
# wait until the State shows as "Off"
# Now we will start the VM with virtctl
# This can also be done from the OpenShift Console
virtctl start vm-fedora-persist
# Validate that the vm is running
oc get vmi
# once the vmi shows running we can connect to the console using the virtctl command
virtctl console vm-fedora-persist
# login with fedora:fedora
# Create a helloworld file
$ touch helloworld
$ ls
# logout and disconnect from the console
exit
# press CTRL + ] to disconnect from console
```

### Migrating a vm from one host to another

Running VMs can be migrated from one host to another from both the UI as well as from the command line. Lets migrate our running vm from one hsot to another.

```
# lets get the Node that the vmi is running on
$ oc describe vmi/vm-fedora-persist | grep "Node Name"
  Node Name:                      worker1
# take note of the node name (eg. worker1 in above output)
$ virtctl migrate vm-fedora-persist
$ oc describe vmi/vm-fedora-persist | grep "Node Name"
```

You can also run a vm migration from within the OpenShift console
1. Log into the OpenShift Console (https://console-openshift-console.apps.\<your cluster dns name\>/)
2. Select "Workloads" from the left hand side
3. Select "Virtualization"
4. Use the "Project Drop down" selector to select the "demovms" project
5. Select "vm-fedora-ephemeral" 
6. From the "Actions" dropdown select "Migrate Virtual Machine"
7. Select the "Migrate" button from the popup
8. Watch the events window as well as the "Node" indicator showing which Node the vm is running on

### Cleanup

At this point we will now shut down the VM 

```
$ virtctl stop vm-fedora-persist
$ oc get vmi
$ oc get vm
```

Note that while there is no longer a virtual machine instance (vmi) but there is an virtual machine. We will now delete that virtual machine.

```
oc delete vm/vm-fedora-persist
```

## Exposing a service from a VM

Just like apps running inside a container, we can expose applications/services from a running VM. We will create a new VM and expose a very simple "hello world" service from this VM in this demo. For this demo we will use an ephemeral VM, that exposes a very simple "Hello World" service on port 1500. We will continue to leverage our Fedora containerDisk from quay.io as a base, and will inject a few setup commands via cloudinit to get the service up and running.

Start by taking a look at `examples/fedora-hello-world/fedora-hw.yml` as you will see it is basically the same YAML that we used in our very first VM, but this time we have installed a small software package and created a "server" using netcat on port 1500. Lets start this VMI

```
# start by creating our new VM
$ oc create -f examples/fedora-hello-world/fedora-hw.yml
# lets create a k8s service to expose the hello-world server we are running
$ oc create -f examples/fedora-hello-world/fedora-hw-svc.yml
# and now lets expose this service with an OpenShift route
$ oc expose svc/helloworld
$ oc get route
NAME         HOST/PORT                                    PATH   SERVICES     PORT   TERMINATION   WILDCARD
helloworld   helloworld-demovms.apps.cnv.example.com             helloworld   1500                 None
$ curl http://helloworld-demovms.apps.cnv.example.com
Hello World! from vm-fedora-hw
```

Congratulations! You have created a vm, installed an application and made that application available for others to use. While this is a very contrived example, the same principals can be used to expose any service or application running inside a vm to your project, or to the outside world. Go ahead and delete this VM, and we will create a new one, this time with liveness and readyness probes.

```
oc delete vmi/vm-fedora-hw
```

*** Warning there be dragons here ***

We are now going to use a virtualMachineInstanceReplicaSet. This is a very powerful construct, but is not yet GA in OpenShift Virtualization. Use at your own peril.

### VMIRS and Liveness Probes

We are now going to build on our last "hello world" service, but instead of running just one ephemeral virtual machine, we are going to run multiple instances, giving us high availability for our simple hello-world service.

Lets create our virtual machine instance replica set:
`oc create -f examples/fedora-hello-world/fedora-hw-vmris.yaml`

Now list out the running vm instances:
`oc get vmi`

Note that there are THREE virtual machines running. This is similar to creating a replicaSet in kubernetes, we specified that we want 3 instances of this VM running.

Now if you curl the endpoint we created earlier a few times you will see that the message changes. This is becuase we are now load balancing our requests across all three VMs.

But we can do even better than this, open a new terminal window and run the following commands making sure to update the URL for your specific instance:

```
while true;
do
 curl http://helloworld-demovms.apps.cnv.example.com
 sleep 10
done
```

Leave this simple "client" running and go back to your first window. Now lets connect to one of our running vms and kill the server, to simulate a service failure

```
$ oc get vmi
NAME                   AGE     PHASE     IP            NODENAME
vm-fedora-hw           26m     Running   10.130.0.68   worker1
vm-fedora-hw-rsbf86p   8m34s   Running   10.128.2.20   worker0
vm-fedora-hw-rsgj22d   8m34s   Running   10.128.2.21   worker0
vm-fedora-hw-rsqz9jd   8m34s   Running   10.130.0.75   worker1
$ virtctl console vm-fedora-hw-rsbf86p
# log in with fedora:fedora
# we will become root, and then look for our helloworld server and kill it
$ sudo su -
# ps -ef | grep helloworld
root        1123       1  0 22:46 ?        00:00:00 /usr/bin/nc -klp 1500 -e /usr/bin/cat /etc/helloworld
# kill -9 1123
```

From the openshift-console, watch the list of VMs that are currently running in the "demovms" project. After about 60 seconds you should see that one of your vms (the one you just killed the process on) is terminating, and another vm is being brought up in its place. This is the liveness probe doing its job.

Go back to the window where we are running curl in a loop. Note that even though one of the VMs had an issue, the service continued to provide a response, and without any interaction, and the new vm was placed into the rotation once it was fully up and running.

OK enough playing with our ephemeral VMs, lets clean this all up and try something else.

### Cleanup

``` 
oc delete vmirs/vm-fedora-hw-rs
oc delete svc/helloworld
oc delete route/helloworld
```

## Cloning a VM

Now that we have a persistent fedora virtual machine, we can clone that vm and make multiple copies of it.

## Importing a VM from vSphere

It is possible to import an existing virtual machine from a vSphere cluster. There are a few caveats to be aware of before attempting this import.
1. the vm you wish to import must be powered off. Importing running vms is not supported at this time
2. drivers 


## Importing a VMDK


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


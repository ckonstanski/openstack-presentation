# OpenStack Presentation

## What Is a Cloud?

A cloud is a platform for running virtual machines. But it's more than
that. What differentiates a virtual machine farm from a cloud is the
tooling and automation that makes it possible for users to manage all
aspects of their VMs without the intervention of a system
administrator. The management of the VMs themselves, networking,
storage volumes, SSH keys and more is controlled by tooling that is
accessible to the user via a web UI, CLI, REST API and orchestration
services.

There are several public cloud platforms in existence: Azure,
Rackspace, AWS and Google Cloud are the most popular. But if you want
to run your own private cloud, the choices are limited: Azure or
OpenStack.

OpenStack is the cloud platform that many public cloud providers
use. Rackspace actually invented it in 2010. And there are a plethora
of hosting providers both big and small that use OpenStack without
necessarily advertising it as a cloud.

But the real interest in OpenStack is for implementing your own
private cloud. There are all sorts of good business reasons to do
this, but that is a topic for another presentation.

Once you decide you want a private cloud, your choices are: pay
Microsoft for an Azure license, or pay with sweat equity for
OpenStack. I'm not going to lie to you: implementing OpenStack is not
easy. But it is free and open source. And there is a massive community
working on it at all times. It is stable and mature enough for
companies like Charter, Comcast, Verizon, Walmart and Volkswagen to
bet their businesses on it.

## Basic Components of Any Cloud

We'll tackle this from top down, starting with an abstract overview
and finishing with the nuts and bolts at the OS level.

### Abstract Requirements

* Virtual machines
  * Images
  * Flavors
* Storage
  * Block storage
  * Object storage
* Networking
  * Internal subnets
  * External network
  * Routers
* Secure console/shell access to VMs
* Firewall
* DNS
* Orchestration
* Security for all of the above

### OpenStack Services Providing These Requirements

All of these functions are hot handled by one monolithic
application. Instead the following list of services exist which each
handle a specific subset of tasks. These services are called "modules"
or "projects" depending on the context. When speaking of the OpenStack
architecture (as we are doing now), we would use the term
"modules". When speaking of the development effort behind these
modules, i.e. the community of developers who work on the code and the
tooling built around the code repositories, we would call them
"projects". Each of these modules has its own project in GitHub.

* Nova (compute)
* Glance (images)
* Cinder (block storage)
* Swift (object storage)
* Neutron (networking)
* Designate (DNS)
* Heat (orchestration)
  * Ceilometer (metrics/events)
* Keystone

### OS Services Underneath the OpenStack Services

OpenStack runs on Linux. Therefore when we speak of OS support, we are
talking about Linux tools. You can think of OpenStack as a very large
and disributed application that ties a myriad of Linux utilities
together to create a cloud platform.

Here are some of the most important building blocks at the OS level:

* QEMU/KVM
* libvirt
* iptables
* OpenVSwitch
* Network namespaces
* MySQL
* RabbitMQ

#### Virtual Machines

##### QEMU/KVM

QEMU and KVM are the userspace and kernelspace parts of the basic
underlying support for virtualization in Linux. There are two other
native Linux virtualization systems, namely usermode and xen. But
QEMU/KVM has dominated since 2010 when OpenStack adopted it.

You can run VMs with nothing other than a correctly configured kernel
and the QEMU package installed. You have to set up networking
yourself, which entails DNS, DHCP, and either a bridging or NAT
solution. Once these are in place you can create and run VMs with qemu
commands.

##### libvirt

Doing all the network and VM management manually for your QEMU VMs is
fine for a laptop, but does not scale to the enterprise. libvirt is a
collection of services that automates much of it. You still have to
spin up VMs yourself, but the networking is handled automatically. And
the command to spin up a VM is much simpler. Plus there is a GUI
called libvirt-manager that looks like every other virtualization GUI
you are familiar with (VMWare, VirtualBox).

The important thing about libvirt where OpenStack is concerned is that
everything can be done from the command line. This is how Nova
controls libvirt.

#### Networking

The OpenStack networking stack is quite complicated. It would take an
hour just to do an overview of it. We're just going to touch on three
key components of it: iptables, openvswitch and network namespaces.

##### iptables

When you make security groups, you're really creating iptables
rules. Every VM has a tap interface. These iptables rules are
associated with that interface. If we view an OpenStack networking
diagram, we would see that the iptables rules "live" right next to the
VM itself.

##### OpenVSwitch

OpenVSwitch is a full-featured Linux software bridging solution. If
you have multiple VMs running on a single hypervisor host, you need a
way to connect them all to the physical network interface, much like a
hardware switch connects one physical interface to many
others. OpenVSwitch actually does a lot more than that, but bridging
is its most ubiquitous function.

##### Network Namespaces

Linux lets you create containerized network namespaces. This is the
magic that allows OpenStack users to create private subnets. Each
network namespace encapsulates a router, the private subnet behind the
router, and the public IPs (called floating IPs or FIPs) that the
router NATs to the instances in the private subnet.

Network namespaces allow many users on the same hypervisor to create
subnets with the same IP range. For instance it is perfectly OK for 10
users to each create 192.168.0.0/24 networks. They will not collide
because they are encapsulated within separate namespaces.

##### MySQL

All of the OpenStack modules need to keep some sort of stateful
information. They do this via MySQL. When you upload an SSH key, it
gets stored in the nova database. When you add a user or a role to a
project, it gets stored in the keystone database. Each module has its
own database.

##### RabbitMQ

When you perform a high-level action in OpenStack, for instance
spinning up a new VM, there is an entire chain of events that has to
occur. Messages must be passed from one module to the next. (Remember
that the OpenStack modules are completely separate applications with
no tight coupling.) RabbitMQ is the vehicle for transmitting these
messages. All the messages that go through RabbitMQ are transitory in
nature. If RabbitMQ happens to crash while you're trying to perform an
operation, then the worst that can happen is that the operation will
not happen and you'll have to do it again. Unlike a database, when you
restore RabbitMQ you don't worry about restoring data.

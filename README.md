# OpenStack Presentation

## What Is a Cloud?

A cloud is a platform for running virtual machines. But it's more than
that. What differentiates a virtual machine farm from a cloud is the
tooling and automation that makes it possible for users to manage all
aspects of their VMs without the intervention of a system
administrator. The management of the VMs themselves, networking,
storage volumes, object storage, SSH keys and more is controlled by
tooling that is accessible to the user via a web UI, CLI, REST API,
programming API and orchestration services.

There are several public cloud platforms in existence. Azure,
Rackspace, AWS and Google Cloud are a few of the most popular. But if
you want to run your own private cloud, there are two clear choices:
Azure or OpenStack.

Once you decide you want to build a private cloud, your choices are:
pay Microsoft for an Azure license, or pay with sweat equity for
OpenStack. I'm not going to lie to you: implementing OpenStack is not
easy. But it is free and open source. And there is a massive community
working on it at all times. It is stable and mature enough for
companies like Rackspace, CERN, GoDaddy, Charter, Comcast, Verizon,
Walmart, Volkswagen and many others to bet their businesses on it.

## Evolution of the Cloud

It is useful to understand why we have cloud platforms. They evolved
to solve very real business problems. IT and baremetal solutions just
aren't fast enough to keep up with today's business requirements. The
cloud is the answer to self-service computing infrastructure.

## Basic Components of An OpenStack Cloud

We'll tackle this from top down, starting with an abstract overview
and finishing with the nuts and bolts at the OS level.

### Generic Requirements

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
* Firewall (between projects, between VMs and control plane)
* DNS
* Orchestration
* Dashboard
* APIs
* Authentication

### OpenStack Services Providing These Requirements

These functions are not all handled by one monolithic application. The
following list of services exist which each handle a specific subset
of tasks. These services are called "modules" or "projects" depending
on the context. When speaking of the OpenStack architecture (as we are
doing now), we would use the term "modules". When speaking of the
development effort behind these modules, i.e. the community of
developers who work on the code and the tooling built around the code
repositories, we would call them "projects". Each of these modules has
its own project in GitHub.

* Keystone (authentication)
* Nova (compute)
* Glance (images)
* Cinder (block storage)
* Swift (object storage)
* Neutron (networking)
* Designate (DNS)
* Heat (orchestration)
* Ceilometer (telemetry)
* Horizon (dashboard)
* CLI
* REST APIs
* Python modules

### OS Services Underneath the OpenStack Services

OpenStack runs on Linux. Therefore when we speak of OS support, we are
speaking about Linux tools. You can think of OpenStack as a very large
and disributed application that ties a myriad of Linux utilities
together to create a cloud platform.

Here are some of the most important building blocks at the OS level:

* QEMU/KVM
* libvirt
* iptables
* OpenVSwitch
* VLAN
* Network namespaces
* MySQL
* RabbitMQ
* Python

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
collection of tooling and services that simplifies the management of
multiple VMs. It looks much like other VM management systems you may
have seen, for example VMWare Workstation, XenServer and
VirtualBox. There is a GUI that lets you click your way through the
common operations and a CLI where you can do literally everything that
libvirt supports. libvirt handles networking for your VMs, giving you
such options as bridging, NAT and host-only.

The important thing about libvirt where OpenStack is concerned is that
everything can be done from the command line. This is how Nova
controls libvirt. libvirt is the service that actually controls the
VMs on a single hypervisor. Nova tracks the states and locations of
all the VMs across the entire cloud, decides which hypervisor any
given VM should run on (called "scheduling"), and sends libvirt on
that hypervisor the necessary commands to control the state of a VM.

Nova's state information can get out of sync with the reality that is
represented by libvirt's state. If a VM is actually running on compute
host 1, libvirt will always correctly report it as such. But nova
might think that the VM is on host 2. We call such VMs "ghosts". A VM
will happily keep running in this condition, but its state cannot be
changed until the problem is fixed because nova will send the libvirt
commands to the wrong host. In fact the entire OpenStack
infrastructure can fall down, but as long as the compute hosts are
still running and the VMs are still running on them, they will
continue to do so.

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

#### VLAN

VLAN is a network capability that allows multiple separate logical
networks to exist on the same physical network infrastructure. Packets
are tagged with an identifier. Those packets with the correct
identifier are accepted as belonging to the network, while those
without the correct identifier are ignored. The identifier is called a
tag. Packets can be VLAN-tagged either by the Linux OS or by network
switches.

OpenStack makes use of VLANs to keep the VM network separate from the
control plane network. It would be a huge security risk if VMs could
access the OpenStack control plane network. One could use two separate
physical networks, or one could cut costs in half by using two VLANs
on the same hardware.

##### Network Namespaces

Linux lets you create containerized network namespaces. This is the
magic that allows OpenStack users to create private subnets. Each
network namespace encapsulates a router, the private subnet behind the
router, and the public IPs (called floating IPs or FIPs) that the
router NATs to the instances in the private subnet.

Network namespaces allow many users on the same hypervisor to create
subnets with the same IP range. For instance it is perfectly OK for 10
users to each create 192.168.0.0/24 networks. They will not collide
because they are encapsulated within separate namespaces. At some
point they have to route to the "outside world", and these gateway IPs
must be globally unique. These are called Floating IPs in OpenStack.

### Data and Messaging

##### MySQL

All of the OpenStack modules need to keep some sort of stateful
information. They do this via MySQL. When you spin up a VM, its
location needs to be known. When you upload an SSH key, it gets stored
in the nova database. When you add a user or a role to a project, it
gets stored in the keystone database. Each module has its own
database.

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

### Python

All of the OpenStack projects are written in Python. There are tools
that OpenStack uses that are written in other languages. For instance
Puppet is written in Clojure and JRuby, Jenkins is written in Java,
and RabbitMQ is written in Erlang. But those are third-party
applications. If it's in the OpenStack github project, it's
python. This is useful to know because if you spend enough time in
OpenStack, you will eventually want to fix something. There is an
entire process for getting approved as a code contributer. Anyone with
enough patience can do it. The process is a bit bureaucratic, but it's
open to anyone.

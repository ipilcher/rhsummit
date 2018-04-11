![Red Hat logo](/images/redhat-33.png)

# Red Hat Summit 2018 - San Francisco, California

**Title:** Understanding Containerized Red Hat OpenStack Platform (L1018)  
**Date:** May 8, 2017

**Authors:**
* Ian Pilcher <<ipilcher@redhat.com>>
* Greg Charot <<gcharot@redhat.com>>
* Jacob Liberman <<jliberma@redhat.com>>
* Rhys Oxenham <<roxenham@redhat.com>>

## Lab Contents

* **Lab 0:** Introduction
* **Lab 1:** Lab Environment
* **Lab 2:** Everything else

## Lab 0: Introduction

Thank you for joining us at Red Hat Summit 2018, and welcome to this afternoon's
hands-on lab - **Understanding Containerized Red Hat OpenStack Platform**.

In this lab, we will be taking an in-depth look at the latest release of Red
Hat OpenStack Platform, OSP 12 (which is based on the upstream OpenStack Pike
release).  In particular, we will be looking at OSP 12's implementation of
containerized OpenStack and infrastructure services and how containerization
affects tasks such as deployment, updates, monitoring, troubleshooting, etc.

This is an intermediate-to-advanced lab.  It is not an introduction to Linux
containers, OpenStack, Red Hat OpenStack Platform, or OSP director/TripleO.
This doesn't mean that you need to be an expert with these technologies, but we
simply don't have time to talk about them individually in any depth.  (If you're
interested in learning more about OSP director, we recommend attending **Hands
on with Red Hat OpenStack Platform director** (L1010) at this time tomorrow.)

We have a number of Red Hat OpenStack subject matter experts in the room with us
today.  If you have any problems or questions, please raise your hand, and one
of them will be with you as soon as possible.

### OpenStack and Containers

Having written that we're not going to talk about containers, we're now going to
talk about containers, but only a few aspects that are relevant to Red Hat
OpenStack Platform 12.

**We are not talking about running containerized applications on OpenStack.**
OpenStack is a fantastic platform for running containerized applications and
container orchestration engines, such as Red Hat OpenShift Container Platform,
but that is not the subject of this lab.
 
Instead, this lab is concerned with running the services that manage an
OpenStack environment within containers.  Why is Red Hat doing this?

Running OpenStack (and other) services within containers provides a number of
benefits.

* **Stability** - Each container runs as an independent stack, with no
  dependencies on other containers.
* **Security** - Applications running within containers are isolated from the
  host operating system and each other.  Because container images are immutable,
  any compromise is contained, and containers can easily be recreated from
  sources.
* **Lifecycle management** - Because containers are isolated, atomic units, each
  containerized service can be updated independently.  Unlike package-based
  deployments, containerized services can be updated without breaking shared
  library dependencies.
* **Flexibility** - Red Hat OpenStack Platform 10 introduced the concept of
  composable service roles.  Rather than offering only a handful of monolithic
  roles with pre-defined services (such as the controller and compute roles),
  OSP 10 and later allow administrators to create roles with their own unique
  combinations of services.  Containerization eases the deployment of these
  custom service combinations.
* **Control** - Resource consumption can be limited at the container level,
  ensuring that a rogue service does not starve other services of resources.
  Architectures such as Red Hat Hyperconverged Infrastructure for Cloud could
  not be reliably deployed without this control.

Note that there are a few ways in which the services in Red Hat OpenStack
platform don't fit the traditional containerized application model.

* **Networking** - Red Hat OpenStack Platform has supported network isolation -
  separating OpenStack and workload traffic onto multiple physical networks -
  since its very first release.  Container runtimes and orchestrators, however,
  tend to be optimized for services that each have a single IP address on a
  single flat network.  To accomodate OSP's requirement for multiple networks,
  all containers in OSP 12 run with ``host`` networking, where all host network
  interfaces, IP addresses, etc., are exposed to the containerized applications.
* **Configuration** - OSP 12 is the sixth version of Red Hat OpenStack Platform
  to use OSP director (TripleO) for installation and configuration.  Over those
  six releases, OSP director has developed a robust system for creating and
  customizing OpenStack (and other) configuration files - including support for
  site-specific customizations and a large ecosystem of partner plug-ins.  A
  later lab will explore how OSP 12 preserves compatibility with customer
  configurations and partner-plugins - including compatibility across upgrades
  from non-containerized OSP 11 to containerized OSP 12!
* **Logging** - Just like OpenStack services expect to read their configuration
  from local configuration files, they expect to write their logs to local log
  files.  OSP 12 makes these log files visible to the host operating system,
  although their locations differ from those of the non-containerized services
  in OSP 11.  (A later lab will provide details.)  Tools that parse these log
  files will need to be updated for OSP 12.
* **Privileged operations** - Isolation from the host operating system is one of
  the benefits of containerizing applications, but there are certain services
  within OSP that require privileged access to the host OS.  For example,
  libvirt interacts with KVM to manage virtual machines, and Neutron creates
  and destroys network namespaces and Open vSwitch bridges and ports.  For this
  reason, a few containers within OSP 12 run as ``privileged`` containers.

### TripleO Terminology

![TripleO](/images/overcloud-undercloud.png)

OpenStack Platform director (OSP director) is based on the upstream TripleO
project.  TripleO stands for "OpenStack on OpenStack," but what does this mean?

We often talk about instances (usually virtual machines) running "on" a
particular cloud or infrastructure, such as OpenStack - meaning that that
infrastructure is responsible for deploying, managing, metering, and ultimately
retiring the instance.  In addition to traditional virtual machine instances,
OpenStack supports bare-metal instances via a service called **Ironic**.  Thus,
it is possible to run a bare-metal server "on" an OpenStack cloud, and we use
this capability to deploy and manage Red Hat OpenStack Platform.

This brings us to two very important terms, that will be used throughout this
lab.

* **Undercloud** - The undercloud (sometimes refered to as the OSP director
  node) is an "all-in-one" OpenStack deployment that deploys, configures, and
  manages a set of bare-metal servers which run Red Hat OpenStack Platform.
* **Overcloud** - The OSP servers managed by the undercloud are refered to as
  the overcloud, because they run "on top of" the undercloud.

OpenStack on OpenStack!

## Lab 1: Lab Environment

This lab features a realistic OSP 12 deployment (except for the number of
compute nodes), including:

* A highly available OpenStack control plane
* Isolated networks (VLANs) for
  * internal control plane - APIs, messaging, database, etc.
  * storage and storage management
  * external API/tenant access (floating IPs)
* Pre-determined IP addresses
* Bonded network interfaces
* Red Hat Ceph Storage

This environment runs as a set of virtual machines in the Ravello cloud, which
allows us to simulate much of the network infrastructure (NICs, switches, VLANs)
found in real datacenters.  Red Hat has developed an IPMI "bridge," which maps
IPMI commands to Ravello API calls, allowing us to use standard power control
agents.  Finally, it also supports nested virtualization, which allows our
compute node VM(s) to run KVM guests themselves.

You will access your personal lab environment via a bastion VM.  To log in to
the bastion, ...

The bastion also acts as an NTP server, yum repository server, and container
image registry for your environment.

![Lab diagram](/images/lab-diagram.svg)
![Lab networks](/images/lab-networks.svg)


Deploying this OSP 12 architecture into the Ravello environment takes at least
60 minutes, so we won't spend the first half of our time watching a deployment.
Instead, we'll start with a fully deployed environment, do some exploration
and testing, and then finish up by deleting the overcloud and starting a new
deployment (which probably won't complete).

![Lab diagram](/images/lab-diagram.svg)

Networks:

![Lab networks](/images/lab-networks.svg)

|Overcloud Host     |IPMI        |External       |Internal    |VXLAN<br>Tunnels|Storage     |Storage<br>Management|
|-------------------|------------|---------------|------------|----------------|------------|---------------------|
|overcloud-ctrl01   |172.16.0.131|192.168.122.201|172.17.1.201|172.17.2.201    |172.17.3.201|172.17.4.201         |
|overcloud-ctrl02   |172.16.0.132|192.168.122.202|172.17.1.202|172.17.2.202    |172.17.3.202|172.17.4.202         |
|overcloud-ctrl03   |172.16.0.133|192.168.122.203|172.17.1.203|172.17.2.203    |172.17.3.203|172.17.4.203         |
|overcloud-ceph01   |172.16.0.136|               |            |                |172.17.3.221|172.17.4.221         |
|overcloud-ceph02   |172.16.0.137|               |            |                |172.17.3.222|172.17.4.222         |
|overcloud-ceph03   |172.16.0.138|               |            |                |172.17.3.223|172.17.4.223         |
|overcloud-compute01|172.16.0.134|               |172.17.1.211|172.17.2.211    |172.17.3.211|                     |
|overcloud-compute02|172.16.0.135|               |172.17.1.212|172.17.2.212    |172.17.3.212|                     |

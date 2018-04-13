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
* **Lab 2:** Looking Around
* **Lab 3:** Everything else

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

**Networks**

![Lab networks](/images/lab-networks.svg)

Because we're using pre-determined IP addresses, we already know what our
overcloud nodes' addresses will be on these networks.  (The exceptions are the
provisioning network addresses, which are always dynamically assigned.  We
happen to be sharing that network with our IPMI BMCs, so we actually have a mix
of static and dynamic addresses on the same network.)

|Overcloud Host|IPMI        |External       |Internal    |VXLAN<br>Tunnels|Storage     |Storage<br>Management|
|--------------|------------|---------------|------------|----------------|------------|---------------------|
|lab-ctrl01    |172.16.0.131|192.168.122.201|172.17.1.201|172.17.2.201    |172.17.3.201|172.17.4.201         |
|lab-ctrl02    |172.16.0.132|192.168.122.202|172.17.1.202|172.17.2.202    |172.17.3.202|172.17.4.202         |
|lab-ctrl03    |172.16.0.133|192.168.122.203|172.17.1.203|172.17.2.203    |172.17.3.203|172.17.4.203         |
|lab-ceph01    |172.16.0.136|               |            |                |172.17.3.221|172.17.4.221         |
|lab-ceph02    |172.16.0.137|               |            |                |172.17.3.222|172.17.4.222         |
|lab-ceph03    |172.16.0.138|               |            |                |172.17.3.223|172.17.4.223         |
|lab-compute01 |172.16.0.134|               |172.17.1.211|172.17.2.211    |172.17.3.211|                     |
|lab-compute02 |172.16.0.135|               |172.17.1.212|172.17.2.212    |172.17.3.212|                     |

Deploying this OSP 12 architecture into the Ravello environment takes at least
60 minutes, and we don't want to spend half of our time just watching a
deployment.  So we'll start with a fully deployed environment in which we can
do som exploration and testing.  As a final step, we'll delete the overcloud and
look at the deployment process.

## Lab 2: Looking Around

In this lab, we'll examine our containerized OSP 12 deployment.

First, log in to the undercloud.  (Enter ``redhat`` as the password.)

```
[lab-user@bastion-XXXX ~]$ ssh stack@undercloud.example.com
The authenticity of host 'undercloud.example.com (192.168.122.253)' can't be established.
ECDSA key fingerprint is SHA256:wRBB50VRT5XX6IqOy668ZGM28clpKbJ/K5FdHevw7Wg.
ECDSA key fingerprint is MD5:0b:33:69:86:26:4a:07:5e:04:a1:63:2e:ef:da:01:1a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'undercloud.example.com,192.168.122.253' (ECDSA) to the list of known hosts.
stack@undercloud.example.com's password:
Last login: Thu Apr 12 19:37:45 2018 from bastion.example.com
```

Before doing anything else, let's check that our deployment is working properly.
(It was just powered on for the first time in several weeks.)

Source the ``stackrc`` file to access OpenStack services on the undercloud.

```
[stack@undercloud ~]$ . stackrc
(undercloud) [stack@undercloud ~]$
```

Check the status of our overcloud nodes.  (Remember that these nodes are
managed by the Ironic service on the **undercloud**.)

Please let one of the instructors know if any of your nodes show ``None`` in the
power state column or ``True`` in the maintenance column.

```
(undercloud) [stack@undercloud ~]$ openstack baremetal node list
+--------------------------------------+---------------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name                | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+---------------------+--------------------------------------+-------------+--------------------+-------------+
| b77a7893-e5ec-4107-b6db-63d755548cc7 | overcloud-ctrl01    | 5c2f6fd7-3351-4ca6-bd41-3fafb1de5162 | power on    | active             | False       |
| c1c3a604-07d1-4510-8172-69c2bd2318d7 | overcloud-ctrl02    | f8c7a2b3-73c8-476f-87a9-4c0af28e7595 | power on    | active             | False       |
| cccc3857-c2b5-4c31-b5ee-d2b2ede4ea75 | overcloud-ctrl03    | b837722d-0d91-4e50-a359-223487fbdb2e | power on    | active             | False       |
| 6b990f06-a262-4fd4-b132-4775ef1bf0c8 | overcloud-ceph01    | 9e7924fd-4611-41de-a29a-c600502e12a0 | power on    | active             | False       |
| 72b19247-5104-4e73-87fc-fa96a9fd504f | overcloud-ceph02    | 66515ba8-15eb-480a-ad37-c3e91da47df8 | power on    | active             | False       |
| 497a58e0-607d-47b0-a08c-049e2377f566 | overcloud-ceph03    | 47e02f2f-b3fe-4f0a-83b6-d0305004aec9 | power on    | active             | False       |
| 329eac87-50b7-43fb-8de4-03d62fd63a17 | overcloud-compute01 | 87920ee2-dd27-432d-b8b1-52a2ab49a9ff | power on    | active             | False       |
| 29dbc509-85c8-4acf-902d-b03d70fa3541 | overcloud-compute02 | None                                 | power off   | available          | False       |
+--------------------------------------+---------------------+--------------------------------------+-------------+--------------------+-------------+
```



```
(undercloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+------------------+--------+----------------------+----------------+--------------+
| ID                                   | Name             | Status | Networks             | Image          | Flavor       |
+--------------------------------------+------------------+--------+----------------------+----------------+--------------+
| 47e02f2f-b3fe-4f0a-83b6-d0305004aec9 | lab-ceph02       | ACTIVE | ctlplane=172.16.0.23 | overcloud-full | ceph-storage |
| 5c2f6fd7-3351-4ca6-bd41-3fafb1de5162 | lab-controller03 | ACTIVE | ctlplane=172.16.0.36 | overcloud-full | control      |
| b837722d-0d91-4e50-a359-223487fbdb2e | lab-controller01 | ACTIVE | ctlplane=172.16.0.32 | overcloud-full | control      |
| f8c7a2b3-73c8-476f-87a9-4c0af28e7595 | lab-controller02 | ACTIVE | ctlplane=172.16.0.22 | overcloud-full | control      |
| 9e7924fd-4611-41de-a29a-c600502e12a0 | lab-ceph03       | ACTIVE | ctlplane=172.16.0.33 | overcloud-full | ceph-storage |
| 66515ba8-15eb-480a-ad37-c3e91da47df8 | lab-ceph01       | ACTIVE | ctlplane=172.16.0.31 | overcloud-full | ceph-storage |
| 87920ee2-dd27-432d-b8b1-52a2ab49a9ff | lab-compute01    | ACTIVE | ctlplane=172.16.0.25 | overcloud-full | compute      |
+--------------------------------------+------------------+--------+----------------------+----------------+--------------+

(undercloud) [stack@undercloud ~]$ ssh heat-admin@172.16.0.32 sudo ceph -s
    cluster 62e500d2-3e8a-11e8-8abf-2cc26041e5e3
     health HEALTH_WARN
            clock skew detected on mon.lab-controller02, mon.lab-controller03
            too many PGs per OSD (352 > max 300)
            Monitor clock skew detected 
     monmap e2: 3 mons at {lab-controller01=172.17.3.201:6789/0,lab-controller02=172.17.3.202:6789/0,lab-controller03=172.17.3.203:6789/0}
            election epoch 10, quorum 0,1,2 lab-controller01,lab-controller02,lab-controller03
     osdmap e31: 6 osds: 6 up, 6 in
            flags sortbitwise,require_jewel_osds,recovery_deletes
      pgmap v15024: 704 pgs, 6 pools, 64802 kB data, 790 objects
            404 MB used, 269 GB / 269 GB avail
                 704 active+clean
                 
(undercloud) [stack@undercloud ~]$ . testrc
(test@overcloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| ID                                   | Name  | Status  | Networks                       | Image               | Flavor  |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| 828a5d3b-3a01-43dc-ac8b-592e22d3e818 | test1 | SHUTOFF | test=10.0.1.4, 192.168.122.154 | cirros-0.3.4-x86_64 | m1.tiny |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+

(test@overcloud) [stack@undercloud ~]$ openstack server start test1
(test@overcloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+
| ID                                   | Name  | Status | Networks                       | Image               | Flavor  |
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+
| 828a5d3b-3a01-43dc-ac8b-592e22d3e818 | test1 | ACTIVE | test=10.0.1.4, 192.168.122.154 | cirros-0.3.4-x86_64 | m1.tiny |
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+

(test@overcloud) [stack@undercloud ~]$ ssh cirros@192.168.122.154
The authenticity of host '192.168.122.154 (192.168.122.154)' can't be established.
RSA key fingerprint is SHA256:8/gIYkbqVlCt0N5Kte8fZPETgbp6TAbdXTrh4tJUABg.
RSA key fingerprint is MD5:f9:17:09:dc:ab:b6:6f:5a:86:35:a5:5e:50:69:43:5d.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.122.154' (RSA) to the list of known hosts.
$
```

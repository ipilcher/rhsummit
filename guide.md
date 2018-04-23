![Red Hat logo](/images/redhat-33.png)

# Red Hat Summit 2018 - San Francisco, California

**Title:** Understanding Containerized Red Hat OpenStack Platform (L1018)  
**Date:** May 8, 2017  
**Author:** Ian Pilcher <<ipilcher@redhat.com>>

**Contributors:**
* Greg Charot <<gcharot@redhat.com>>
* Jacob Liberman <<jliberma@redhat.com>>
* Rhys Oxenham <<roxenham@redhat.com>>

## Lab Contents

* [**Lab 0:** Introduction](#lab-0-introduction)
  - [OpenStack and Containers](#openstack-and-containers)
  - [TripleO Terminology](#tripleo-terminology)
* [**Lab 1:** Lab Environment](#lab-1-lab-environment)
* [**Lab 2:** Looking Around](#lab-2-looking-around)
  - [Is This Thing On?](#is-this-thing-on)
  - [Containers (Three Different Ones)](#containers-three-different-ones)
    * [Pacemaker-Managed Containers](#pacemaker-managed-containers)
    * [Ceph Containers](#ceph-containers)
    * [Everything Else](#everything-else)
  - [The Anatomy of a Container](#the-anatomy-of-a-container)
    * [Let Me Out! Let Me Out!](#let-me-out-let-me-out)
    * [Logging](#logging)
    * [Configuration Files](#configuration-files)
  - [Other Node Types](#other-node-types)
    * [Compute Nodes](#compute-nodes)
    * [Ceph Storage Nodes](#ceph-storage-nodes)
* [**Lab 3:** Updating the Overcloud](#lab-3-updating-the-overcloud)
  - [Step 1: Update the Undercloud](#step-1-update-the-undercloud)
    * [Prepare the Undercloud](#prepare-the-undercloud)
    * [Start the Update](#start-the-update)
    * [Update Container Images](#update-container-images)
    * [Verify Undercloud Update Completion](#verify-undercloud-update-completion)
    * [Update Overcloud Images](#update-overcloud-images)
  - [Step 2: Update the Overcloud](#step-2-update-the-overcloud)
    * [Update the Plan](#update-the-plan)
* [**Lab 4:** Troubleshooting and Testing](#lab-4-troubleshooting-and-testing)
* [**Lab 5:** Deploying a New Overcloud](#lab-5-deploying-a-new-overcloud)

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

* **Stability** &mdash; Each container runs as an independent stack, with no
  dependencies on other containers.
* **Security** &mdash; Applications running within containers are isolated from the
  host operating system and each other.  Because container images are immutable,
  any compromise is contained, and containers can easily be recreated from
  sources.
* **Lifecycle management** &mdash; Because containers are isolated, atomic units, each
  containerized service can be updated independently.  Unlike package-based
  deployments, containerized services can be updated without breaking shared
  library dependencies.
* **Flexibility** &mdash; Red Hat OpenStack Platform 10 introduced the concept of
  composable service roles.  Rather than offering only a handful of monolithic
  roles with pre-defined services (such as the controller and compute roles),
  OSP 10 and later allow administrators to create roles with their own unique
  combinations of services.  Containerization eases the deployment of these
  custom service combinations.
* **Control** &mdash; Resource consumption can be limited at the container level,
  ensuring that a rogue service does not starve other services of resources.
  Architectures such as Red Hat Hyperconverged Infrastructure for Cloud could
  not be reliably deployed without this control.

Note that there are a few ways in which the services in Red Hat OpenStack
platform don't fit the traditional containerized application model.

* **Networking** &mdash; Red Hat OpenStack Platform has supported network isolation -
  separating OpenStack and workload traffic onto multiple physical networks -
  since its very first release.  Container runtimes and orchestrators, however,
  tend to be optimized for services that each have a single IP address on a
  single flat network.  To accomodate OSP's requirement for multiple networks,
  all containers in OSP 12 run with ``host`` networking, where all host network
  interfaces, IP addresses, etc., are exposed to the containerized applications.
* **Configuration** &mdash; OSP 12 is the sixth version of Red Hat OpenStack Platform
  to use OSP director (TripleO) for installation and configuration.  Over those
  six releases, OSP director has developed a robust system for creating and
  customizing OpenStack (and other) configuration files - including support for
  site-specific customizations and a large ecosystem of partner plug-ins.  A
  later lab will explore how OSP 12 preserves compatibility with customer
  configurations and partner-plugins - including compatibility across upgrades
  from non-containerized OSP 11 to containerized OSP 12!
* **Logging** &mdash; Just like OpenStack services expect to read their configuration
  from local configuration files, they expect to write their logs to local log
  files.  OSP 12 makes these log files visible to the host operating system,
  although their locations differ from those of the non-containerized services
  in OSP 11.  (A later lab will provide details.)  Tools that parse these log
  files will need to be updated for OSP 12.
* **Privileged operations** &mdash; Isolation from the host operating system is one of
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
particular cloud or infrastructure, such as OpenStack &mdash; meaning that that
infrastructure is responsible for deploying, managing, metering, and ultimately
retiring the instance.  In addition to traditional virtual machine instances,
OpenStack supports bare-metal instances via a service called **Ironic**.  Thus,
it is possible to run a bare-metal server "on" an OpenStack cloud, and we use
this capability to deploy and manage Red Hat OpenStack Platform.

This brings us to two very important terms, that will be used throughout this
lab.

* **Undercloud** &mdash; The undercloud (sometimes refered to as the OSP director
  node) is an "all-in-one" OpenStack deployment that deploys, configures, and
  manages a set of bare-metal servers which run Red Hat OpenStack Platform.
* **Overcloud** &mdash; The OSP servers managed by the undercloud are refered to as
  the overcloud, because they run "on top of" the undercloud.

OpenStack on OpenStack!

## Lab 1: Lab Environment

This lab features a realistic OSP 12 deployment (except for the number of
compute nodes), including:

* A highly available OpenStack control plane
* Isolated networks (VLANs) for
  * internal control plane &mdash; APIs, messaging, database, etc.
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

### Is This Thing On?

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

Now list look at the overcloud hosts.  From the undercloud's point of view,
these are **instances** that run on the bare-metal nodes.  (The instance IDs can
be used to map the instances to their bare-metal nodes.)

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
```

Check the status of the Ceph storage cluster.  (The clock skew and "too many PGs
per OSD" warnings are expected.  Let an instructor know if you see any other
warnings or errors.)

```
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
```

A number of items have been pre-created in the **overcloud**.

* ``external`` network and subnet
* ``m1.tiny`` instance flavor
* ``cirros-0.3.4-x86_64`` image
* ``test`` tenant and user
* ``test`` network, subnet, and router
* ``stack`` SSH keypair
* ``test1`` instance

Source the ``testrc`` file to access the overcloud as the ``test`` user.

```                 
(undercloud) [stack@undercloud ~]$ . testrc
(test@overcloud) [stack@undercloud ~]$
```

Check the status of the instance.  It should be ``SHUTOFF``.

```
(test@overcloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| ID                                   | Name  | Status  | Networks                       | Image               | Flavor  |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| 828a5d3b-3a01-43dc-ac8b-592e22d3e818 | test1 | SHUTOFF | test=10.0.1.4, 192.168.122.154 | cirros-0.3.4-x86_64 | m1.tiny |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
```

Start the instance and wait for it to become ``ACTIVE``.

```
(test@overcloud) [stack@undercloud ~]$ openstack server start test1
(test@overcloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| ID                                   | Name  | Status  | Networks                       | Image               | Flavor  |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+
| 828a5d3b-3a01-43dc-ac8b-592e22d3e818 | test1 | SHUTOFF | test=10.0.1.4, 192.168.122.154 | cirros-0.3.4-x86_64 | m1.tiny |
+--------------------------------------+-------+---------+--------------------------------+---------------------+---------+

(...)

(test@overcloud) [stack@undercloud ~]$ openstack server list
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+
| ID                                   | Name  | Status | Networks                       | Image               | Flavor  |
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+
| 828a5d3b-3a01-43dc-ac8b-592e22d3e818 | test1 | ACTIVE | test=10.0.1.4, 192.168.122.154 | cirros-0.3.4-x86_64 | m1.tiny |
+--------------------------------------+-------+--------+--------------------------------+---------------------+---------+
```

You should be able to log in to the the instance using the ``stack`` user's
default SSH key.

```
(test@overcloud) [stack@undercloud ~]$ ssh cirros@192.168.122.154
The authenticity of host '192.168.122.154 (192.168.122.154)' can't be established.
RSA key fingerprint is SHA256:8/gIYkbqVlCt0N5Kte8fZPETgbp6TAbdXTrh4tJUABg.
RSA key fingerprint is MD5:f9:17:09:dc:ab:b6:6f:5a:86:35:a5:5e:50:69:43:5d.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.122.154' (RSA) to the list of known hosts.
$

$ exit
Connection to 192.168.122.154 closed.
```

Success!  We haz cloud!

### Containers (Three Different Ones)

Now that we've verified that we have a working OSP deployment, let's take a look
at the running containers that make it work, starting on one of the controllers.

```
(test@overcloud) [stack@undercloud ~]$ . stackrc
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
(undercloud) [stack@undercloud ~]$ ssh heat-admin@172.16.0.32
Last login: Thu Apr 12 22:53:57 2018 from 172.16.0.1
```

We can use the ``docker ps`` command to see all of the containers running on our
controller.

```
[heat-admin@lab-controller01 ~]$ sudo docker ps
CONTAINER ID        IMAGE                                                                       COMMAND                  CREATED             STATUS                    PORTS               NAMES
00c1febf51d2        172.16.0.1:8787/rhosp12/openstack-haproxy:pcmklatest                        "/bin/bash /usr/lo..."   33 minutes ago      Up 33 minutes                                 haproxy-bundle-docker-0
43cfbcd74a4b        172.16.0.1:8787/rhosp12/openstack-redis:pcmklatest                          "/bin/bash /usr/lo..."   36 minutes ago      Up 35 minutes                                 redis-bundle-docker-0
fc081b98c13e        172.16.0.1:8787/rhosp12/openstack-mariadb:pcmklatest                        "/bin/bash /usr/lo..."   37 minutes ago      Up 37 minutes                                 galera-bundle-docker-0
511de88ad6c4        172.16.0.1:8787/rhosp12/openstack-rabbitmq:pcmklatest                       "/bin/bash /usr/lo..."   37 minutes ago      Up 37 minutes (healthy)                       rabbitmq-bundle-docker-0
b2a5531d8b29        172.16.0.1:8787/ceph/rhceph-2-rhel7:latest                                  "/entrypoint.sh"         37 minutes ago      Up 37 minutes                                 ceph-mon-lab-controller01
aac508a9e1b7        172.16.0.1:8787/rhosp12/openstack-gnocchi-api:12.0-20180309.1               "kolla_start"            3 days ago          Up 37 minutes                                 gnocchi_api
fbd82bec10d2        172.16.0.1:8787/rhosp12/openstack-gnocchi-statsd:12.0-20180309.1            "kolla_start"            3 days ago          Up 29 minutes                                 gnocchi_statsd
239cc2d1e27f        172.16.0.1:8787/rhosp12/openstack-gnocchi-metricd:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes                                 gnocchi_metricd
fc2ec5181b0f        172.16.0.1:8787/rhosp12/openstack-panko-api:12.0-20180309.1                 "kolla_start"            3 days ago          Up 37 minutes                                 panko_api
6daf33d978b1        172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_metadata
29a778ff874d        172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_api
0933cf235b28        172.16.0.1:8787/rhosp12/openstack-glance-api:12.0-20180309.1                "kolla_start"            3 days ago          Up 37 minutes (healthy)                       glance_api
39aaba03d12c        172.16.0.1:8787/rhosp12/openstack-swift-account:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_account_server
6f3f80430fe1        172.16.0.1:8787/rhosp12/openstack-aodh-listener:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       aodh_listener
95d6aea9aa80        172.16.0.1:8787/rhosp12/openstack-swift-container:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_container_auditor
975b83fc3b72        172.16.0.1:8787/rhosp12/openstack-heat-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       heat_api_cron
a1a9cb580dfc        172.16.0.1:8787/rhosp12/openstack-swift-proxy-server:12.0-20180309.1        "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_object_expirer
2bb87c565d5a        172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_object_updater
9d7dad2c82b9        172.16.0.1:8787/rhosp12/openstack-swift-container:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_container_replicator
2eaebf03043b        172.16.0.1:8787/rhosp12/openstack-swift-account:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_account_auditor
947d4ac93e31        172.16.0.1:8787/rhosp12/openstack-cron:12.0-20180309.1                      "kolla_start"            3 days ago          Up 37 minutes                                 logrotate_crond
4b3b73d4462a        172.16.0.1:8787/rhosp12/openstack-heat-api-cfn:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       heat_api_cfn
c49ece44bdb0        172.16.0.1:8787/rhosp12/openstack-nova-conductor:12.0-20180309.1            "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_conductor
6245d6b572e7        172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_object_replicator
83e73ff5b689        172.16.0.1:8787/rhosp12/openstack-swift-container:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_container_server
1177bb32afc4        172.16.0.1:8787/rhosp12/openstack-heat-engine:12.0-20180309.1               "kolla_start"            3 days ago          Up 37 minutes (healthy)                       heat_engine
c41932eea6a4        172.16.0.1:8787/rhosp12/openstack-aodh-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes                                 aodh_api
2a2cfd159237        172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_rsync
f016068c3a0d        172.16.0.1:8787/rhosp12/openstack-nova-novncproxy:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_vnc_proxy
97ba84956183        172.16.0.1:8787/rhosp12/openstack-ceilometer-notification:12.0-20180309.1   "kolla_start"            3 days ago          Up 37 minutes (healthy)                       ceilometer_agent_notification
d3e3b00c11aa        172.16.0.1:8787/rhosp12/openstack-swift-account:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_account_reaper
285a675d9cd9        172.16.0.1:8787/rhosp12/openstack-nova-consoleauth:12.0-20180309.1          "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_consoleauth
07ba693a6e11        172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       nova_api_cron
c8932637db4b        172.16.0.1:8787/rhosp12/openstack-aodh-notifier:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       aodh_notifier
c93cb7b9e75a        172.16.0.1:8787/rhosp12/openstack-ceilometer-central:12.0-20180309.1        "kolla_start"            3 days ago          Up 37 minutes (healthy)                       ceilometer_agent_central
48e8befd1dc2        172.16.0.1:8787/rhosp12/openstack-swift-account:12.0-20180309.1             "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_account_replicator
f578f22a8357        172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_object_auditor
055c87a99e69        172.16.0.1:8787/rhosp12/openstack-heat-api:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       heat_api
32ad554e92d6        172.16.0.1:8787/rhosp12/openstack-swift-proxy-server:12.0-20180309.1        "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_proxy
d662bfec9a1f        172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1              "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_object_server
c8801163d88b        172.16.0.1:8787/rhosp12/openstack-nova-scheduler:12.0-20180309.1            "kolla_start"            3 days ago          Up 33 minutes (healthy)                       nova_scheduler
c9aa47cfce16        172.16.0.1:8787/rhosp12/openstack-swift-container:12.0-20180309.1           "kolla_start"            3 days ago          Up 37 minutes (healthy)                       swift_container_updater
68d372e5242b        172.16.0.1:8787/rhosp12/openstack-aodh-evaluator:12.0-20180309.1            "kolla_start"            3 days ago          Up 37 minutes (healthy)                       aodh_evaluator
f4bbdda9a0c4        172.16.0.1:8787/rhosp12/openstack-keystone:12.0-20180309.1                  "/bin/bash -c '/us..."   3 days ago          Up 37 minutes (healthy)                       keystone_cron
6788a5076fba        172.16.0.1:8787/rhosp12/openstack-keystone:12.0-20180309.1                  "kolla_start"            3 days ago          Up 37 minutes (healthy)                       keystone
9aaac2cb4978        172.16.0.1:8787/rhosp12/openstack-nova-placement-api:12.0-20180309.1        "kolla_start"            3 days ago          Up 37 minutes                                 nova_placement
a5b320025088        172.16.0.1:8787/rhosp12/openstack-horizon:12.0-20180309.1                   "kolla_start"            3 days ago          Up 37 minutes                                 horizon
db5b448fc09a        172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1                   "kolla_start"            3 days ago          Up 37 minutes                                 clustercheck
3248787f3cfc        172.16.0.1:8787/rhosp12/openstack-memcached:12.0-20180309.1                 "/bin/bash -c 'sou..."   3 days ago          Up 37 minutes                                 memcached
```

There are 49 separate containers running on our controller.  They can be roughly
divided into three different varieties:

1. Pacemaker-managed containers,
1. Ceph containers, and
1. Everything else.

#### Pacemaker-Managed Containers

Pacemaker has long been used as the high-availability infrastructure in Red Hat
OpenStack Platform.  In versions prior to OSP 12, Pacemaker managed key
infrastructure services on which OpenStack depends; in OSP 12, Pacemaker is
managing **exactly the same set of services**, but almost all of them are now
running as containers.

(In fact, one of the key things to understand about the containerized services
in OSP 12 is that the overall architecture is **exactly the same** as
non-containerized OSP 11.  Services all run in exactly the same place &mdash; as
determined by the standard/customer roles; they are just running as
containers, rather than traditional services.)

Let's look at our Pacemaker cluster.

```
[heat-admin@lab-controller01 ~]$ sudo pcs status
Cluster name: tripleo_cluster
Stack: corosync
Current DC: lab-controller02 (version 1.1.16-12.el7_4.8-94ff4df) - partition with quorum
Last updated: Mon Apr 16 20:49:06 2018
Last change: Thu Apr 12 20:57:04 2018 by root via cibadmin on lab-controller01

12 nodes configured
37 resources configured

Online: [ lab-controller01 lab-controller02 lab-controller03 ]
GuestOnline: [ galera-bundle-0@lab-controller01 galera-bundle-1@lab-controller02 galera-bundle-2@lab-controller03 rabbitmq-bundle-0@lab-controller01 rabbitmq-bundle-1@lab-controller02 rabbitmq-bundle-2@lab-controller03 redis-bundle-0@lab-controller01 redis-bundle-1@lab-controller02 redis-bundle-2@lab-controller03 ]

Full list of resources:

 Docker container set: rabbitmq-bundle [172.16.0.1:8787/rhosp12/openstack-rabbitmq:pcmklatest]
   rabbitmq-bundle-0    (ocf::heartbeat:rabbitmq-cluster):      Started lab-controller01
   rabbitmq-bundle-1    (ocf::heartbeat:rabbitmq-cluster):      Started lab-controller02
   rabbitmq-bundle-2    (ocf::heartbeat:rabbitmq-cluster):      Started lab-controller03
 Docker container set: galera-bundle [172.16.0.1:8787/rhosp12/openstack-mariadb:pcmklatest]
   galera-bundle-0      (ocf::heartbeat:galera):        Master lab-controller01
   galera-bundle-1      (ocf::heartbeat:galera):        Master lab-controller02
   galera-bundle-2      (ocf::heartbeat:galera):        Master lab-controller03
 Docker container set: redis-bundle [172.16.0.1:8787/rhosp12/openstack-redis:pcmklatest]
   redis-bundle-0       (ocf::heartbeat:redis): Master lab-controller01
   redis-bundle-1       (ocf::heartbeat:redis): Slave lab-controller02
   redis-bundle-2       (ocf::heartbeat:redis): Slave lab-controller03
 ip-172.16.0.250        (ocf::heartbeat:IPaddr2):       Started lab-controller01
 ip-192.168.122.150     (ocf::heartbeat:IPaddr2):       Started lab-controller02
 ip-172.17.1.10 (ocf::heartbeat:IPaddr2):       Started lab-controller03
 ip-172.17.1.150        (ocf::heartbeat:IPaddr2):       Started lab-controller01
 ip-172.17.3.150        (ocf::heartbeat:IPaddr2):       Started lab-controller02
 ip-172.17.4.150        (ocf::heartbeat:IPaddr2):       Started lab-controller03
 Docker container set: haproxy-bundle [172.16.0.1:8787/rhosp12/openstack-haproxy:pcmklatest]
   haproxy-bundle-docker-0      (ocf::heartbeat:docker):        Started lab-controller01
   haproxy-bundle-docker-1      (ocf::heartbeat:docker):        Started lab-controller02
   haproxy-bundle-docker-2      (ocf::heartbeat:docker):        Started lab-controller03
 openstack-cinder-volume        (systemd:openstack-cinder-volume):      Started lab-controller01

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

This shows us several different resource types.

1. IP addresses &mdash; Just like earlier releases, Pacemaker manages a virtual IP
   address on each network.  HAProxy listens on these VIPs and distributes
   work across the controllers.

2. ``openstack-cinder-volume`` &mdash; Our old friend, the Cinder volume service, is
   still running in active/passive mode, so we need Pacemaker to ensure that
   **exactly** one instance is running at any particular time.
   
   Cinder volume defaults to running as a traditional, non-containerized service
   in OSP 12 (although a containerized version is included as a tech preview).
   This was done because this service often makes use of storage-specific
   plugins, which must be built into the container image.  Neutron services are
   non-containerized in OSP 12 for the same reason (and also available in
   containerized versions as a tech preview).  Red Hat plans to containierize
   both the Cinder volume and Neutron services in OSP 13.

3. Docker container sets &mdash; We can see 4 Docker container "bundles" being
   managed by Pacemaker &mdash; ``rabbitmq``, ``galera``, ``redis``, and ``haproxy``.

Bundles are a new Pacemaker resource type that define how Pacemaker should run
a container.  For example:

```
[heat-admin@lab-controller01 ~]$ sudo pcs resource show haproxy-bundle
 Bundle: haproxy-bundle
  Docker: image=172.16.0.1:8787/rhosp12/openstack-haproxy:pcmklatest network=host options="--user=root --log-driver=journald -e KOLLA_CONFIG_STRATEGY=COPY_ALWAYS" replicas=3 run-command="/bin/bash /usr/local/bin/kolla_start"
  Storage Mapping:
   options=ro source-dir=/var/lib/kolla/config_files/haproxy.json target-dir=/var/lib/kolla/config_files/config.json (haproxy-cfg-files)
   options=ro source-dir=/var/lib/config-data/puppet-generated/haproxy/ target-dir=/var/lib/kolla/config_files/src (haproxy-cfg-data)
   options=ro source-dir=/etc/hosts target-dir=/etc/hosts (haproxy-hosts)
   options=ro source-dir=/etc/localtime target-dir=/etc/localtime (haproxy-localtime)
   options=ro source-dir=/etc/pki/ca-trust/extracted target-dir=/etc/pki/ca-trust/extracted (haproxy-pki-extracted)
   options=ro source-dir=/etc/pki/tls/certs/ca-bundle.crt target-dir=/etc/pki/tls/certs/ca-bundle.crt (haproxy-pki-ca-bundle-crt)
   options=ro source-dir=/etc/pki/tls/certs/ca-bundle.trust.crt target-dir=/etc/pki/tls/certs/ca-bundle.trust.crt (haproxy-pki-ca-bundle-trust-crt)
   options=ro source-dir=/etc/pki/tls/cert.pem target-dir=/etc/pki/tls/cert.pem (haproxy-pki-cert)
   options=rw source-dir=/dev/log target-dir=/dev/log (haproxy-dev-log)
   options=ro source-dir=/etc/pki/tls/private/overcloud_endpoint.pem target-dir=/var/lib/kolla/config_files/src-tls/etc/pki/tls/private/overcloud_endpoint.pem (haproxy-cert)
```

You can see that this defines things like:

* The image used to run the container,
* The network type to use (``host`` as always)
* The number of replicas that Pacemaker should run
* The command to run within the container
* Host directories that are bind mounted (mostly read-only) into the container

We can use the ``docker inspect`` command to see how these settings have been
applied to the running container.  (``docker inspect`` outputs a large amount of
JSON, so we'll use the ``jq`` command to see the parts in which we're
interested, but it can be instructive to look through the complete output.)

```
[heat-admin@lab-controller01 ~]$ sudo docker inspect haproxy-bundle-docker-0 | jq .[0].Config.Image
"172.16.0.1:8787/rhosp12/openstack-haproxy:pcmklatest"

[heat-admin@lab-controller01 ~]$ sudo docker inspect haproxy-bundle-docker-0 | jq .[0].HostConfig.NetworkMode
"host"

[heat-admin@lab-controller01 ~]$ sudo docker inspect haproxy-bundle-docker-0 | jq .[0].Config.Cmd
[
  "/bin/bash",
  "/usr/local/bin/kolla_start"
]

[heat-admin@lab-controller01 ~]$ sudo docker inspect haproxy-bundle-docker-0 | jq .[0].HostConfig.Binds
[
  "/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro",
  "/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro",
  "/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro",
  "/var/lib/kolla/config_files/haproxy.json:/var/lib/kolla/config_files/config.json:ro",
  "/var/lib/config-data/puppet-generated/haproxy/:/var/lib/kolla/config_files/src:ro",
  "/etc/hosts:/etc/hosts:ro",
  "/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro",
  "/etc/localtime:/etc/localtime:ro",
  "/dev/log:/dev/log:rw",
  "/etc/pki/tls/private/overcloud_endpoint.pem:/var/lib/kolla/config_files/src-tls/etc/pki/tls/private/overcloud_endpoint.pem:ro"
]
```

One final item, which is true of all Pacemaker-managed containers.

```
[heat-admin@lab-controller01 ~]$ sudo docker inspect haproxy-bundle-docker-0 | jq .[0].HostConfig.RestartPolicy.Name
"no"
```

If one of the Pacemaker-managed containers dies for some reason, we don't want
the ``docker`` daemon to automatically restart it; we always want these
containers to be completely managed by Pacemaker.

#### Ceph Containers

The second variety of container in our deployment is a Ceph container.  Each of
our controllers has a Ceph monitor container running and each Ceph storage node
has 2 Ceph OSD containers running (one for each data disk).

Ceph containers are different from "normal" containers (discussed below),
mainly in the way that they are deployed and configured.  Starting with OSP 12,
OSP director deploys Ceph containers with ``ceph-ansible`` &mdash; the same
mechanism that is used for standalone Ceph installations.  This will be
discussed further in the **Deployment** lab below.

One thing which is significant is the startup mechanism for the Ceph containers,
so let's take a look.

```
[heat-admin@lab-controller01 ~]$ sudo docker inspect ceph-mon-lab-controller01 |  jq .[0].HostConfig.RestartPolicy
{
  "MaximumRetryCount": 0,                                                                                                                                                                                          
  "Name": "no"
}
```

So Docker hasn't been configured to automatically start these containers.  How
are they started?

The answer is ``systemd``.  ``ceph-ansible`` creates a ``systemd`` service for
each container that it wishes to run.


```
[heat-admin@lab-controller01 ~]$ systemctl status ceph-mon@lab-controller01.service
● ceph-mon@lab-controller01.service - Ceph Monitor
   Loaded: loaded (/etc/systemd/system/ceph-mon@.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2018-04-16 19:35:23 UTC; 4h 5min ago
 Main PID: 4805 (docker-current)
   CGroup: /system.slice/system-ceph\x2dmon.slice/ceph-mon@lab-controller01.service
           └─4805 /usr/bin/docker-current run --rm --name ceph-mon-lab-controller01 --net=host --memory=1g --cpu-quota=100000 -v /var/lib/ceph:/var/lib/ceph -v /etc/ceph:/etc/ceph -v /etc/localtime:/etc/local...

Apr 16 23:40:13 lab-controller01 docker[4805]: 2018-04-16 23:40:13.665708 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18043: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:18 lab-controller01 docker[4805]: 2018-04-16 23:40:18.640536 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18044: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:23 lab-controller01 docker[4805]: 2018-04-16 23:40:23.650519 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18045: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:28 lab-controller01 docker[4805]: 2018-04-16 23:40:28.648097 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18046: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:33 lab-controller01 docker[4805]: 2018-04-16 23:40:33.643755 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18047: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:38 lab-controller01 docker[4805]: 2018-04-16 23:40:38.643607 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18048: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:41 lab-controller01 docker[4805]: 2018-04-16 23:40:41.339339 7f55decf3700  0 mon.lab-controller01@0(leader).data_health(10) update_stats avail 81% total 61427 MB, used 11181 MB, avail 50246 MB
Apr 16 23:40:43 lab-controller01 docker[4805]: 2018-04-16 23:40:43.644051 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18049: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:48 lab-controller01 docker[4805]: 2018-04-16 23:40:48.652448 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18050: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Apr 16 23:40:53 lab-controller01 docker[4805]: 2018-04-16 23:40:53.646715 7f55e0108700  0 log_channel(cluster) log [INF] : pgmap v18051: 704 pgs: 704 active+clean; 62135 kB data, 415 MB used, 26.../ 269 GB avail
Hint: Some lines were ellipsized, use -l to show in full.

[heat-admin@lab-controller01 ~]$ systemctl is-enabled ceph-mon@lab-controller01.service
enabled

[heat-admin@lab-controller01 ~]$ cat /etc/systemd/system/ceph-mon@.service
[Unit]
Description=Ceph Monitor
After=docker.service

[Service]
EnvironmentFile=-/etc/environment
ExecStartPre=-/usr/bin/docker rm ceph-mon-%i
ExecStartPre=$(command -v mkdir) -p /etc/ceph /var/lib/ceph/mon
ExecStart=/usr/bin/docker run --rm --name ceph-mon-%i --net=host \
  --memory=1g \
--cpu-quota=100000 \
-v /var/lib/ceph:/var/lib/ceph \
  -v /etc/ceph:/etc/ceph \
-v /etc/localtime:/etc/localtime:ro \
--net=host \
-e IP_VERSION=4 \
    -e MON_IP=172.17.3.201 \
      -e CLUSTER=ceph \
  -e FSID=62e500d2-3e8a-11e8-8abf-2cc26041e5e3 \
  -e CEPH_PUBLIC_NETWORK=172.17.3.0/24 \
  -e CEPH_DAEMON=MON \
   \
  172.16.0.1:8787/ceph/rhceph-2-rhel7:latest
ExecStopPost=-/usr/bin/docker stop ceph-mon-%i
Restart=always
RestartSec=10s
TimeoutStartSec=120
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target
```

#### Everything Else

The remainder of the containers running on our controllers are what we will call
"normal" containers (for lack of a better term).  These containers are set to be
automatically started by the Docker daemon.

```
[heat-admin@lab-controller01 ~]$ sudo docker inspect memcached | jq .[0].HostConfig.RestartPolicy.Name
"always"
```

### The Anatomy of a Container

Let's take a detailed look at one of our running containers, to understand how
it's been configured to run an OpenStack service.

First, let's look at the full output of ``docker inspect``.

```
[heat-admin@lab-controller01 ~]$ sudo docker inspect nova_scheduler | less
(...)
        "Path": "kolla_start",
(...)
            "Binds": [
                "/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro",
                "/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro",
                "/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro",
                "/etc/ssh/ssh_known_hosts:/etc/ssh/ssh_known_hosts:ro",
                "/var/lib/kolla/config_files/nova_scheduler.json:/var/lib/kolla/config_files/config.json:ro",
                "/var/log/containers/nova:/var/log/nova",
                "/etc/hosts:/etc/hosts:ro",
                "/etc/localtime:/etc/localtime:ro",
                "/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro",
                "/dev/log:/dev/log",
                "/etc/puppet:/etc/puppet:ro",
                "/var/lib/config-data/puppet-generated/nova/:/var/lib/kolla/config_files/src:ro",
                "/run:/run"
            ],
(...)
            "User": "nova",
(...)
            "Env": [
                "KOLLA_CONFIG_STRATEGY=COPY_ALWAYS",
(...)
```

We've copied a few of the more significant bits of output from this command
above.

#### Let Me Out! Let Me Out!

Thus far, we've done all of our investigation from our controller's host
operating system &mdash; effectively examining containers from the outside.
Let's get an "inside" view.

```
[heat-admin@lab-controller01 ~]$ sudo docker exec -it nova_scheduler /bin/sh
()[nova@lab-controller01 /]$
```

Note that our containerized shell is running as the ``nova`` user.  This matches
the user setting of the container that we saw with ``docker inspect``.

Let's see what is running in the container.

```
()[nova@lab-controller01 /]$ ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
nova           1  0.1  0.7 273240 96276 ?        Ss   19:11   0:10 /usr/bin/python2 /usr/bin/nova-scheduler
nova        1375  0.0  0.0  11776  1748 ?        Ss   20:37   0:00 /bin/sh
nova        1525  0.0  0.0  47448  1664 ?        R+   20:44   0:00 ps aux
```

Not much!  Other than our shell and ``ps`` command, only the ``nova-scheduler``
process is running.

What about networking?

```
()[nova@lab-controller01 /]$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 2c:c2:60:01:02:04 brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond1 state UP mode DEFAULT qlen 1000
    link/ether 2c:c2:60:1a:18:15 brd ff:ff:ff:ff:ff:ff
4: eth2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond1 state UP mode DEFAULT qlen 1000
    link/ether 2c:c2:60:1a:18:15 brd ff:ff:ff:ff:ff:ff
5: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 46:fb:82:8c:fb:b0 brd ff:ff:ff:ff:ff:ff
8: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 3e:79:55:2b:b6:41 brd ff:ff:ff:ff:ff:ff
11: vlan401: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether fe:e6:c3:f1:30:04 brd ff:ff:ff:ff:ff:ff
12: vlan101: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether 4e:ec:95:50:e1:88 brd ff:ff:ff:ff:ff:ff
13: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether 2c:c2:60:1a:18:15 brd ff:ff:ff:ff:ff:ff
14: vlan201: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether 3e:ab:1f:fa:60:2c brd ff:ff:ff:ff:ff:ff
15: vlan301: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether ce:6f:e6:5f:a9:cd brd ff:ff:ff:ff:ff:ff
16: vxlan_sys_4789: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65470 qdisc noqueue master ovs-system state UNKNOWN mode DEFAULT qlen 1000
    link/ether ee:59:fb:45:35:15 brd ff:ff:ff:ff:ff:ff
17: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 06:91:55:06:9a:4c brd ff:ff:ff:ff:ff:ff
18: bond0: <BROADCAST,MULTICAST,MASTER> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
    link/ether 3e:41:77:25:bf:72 brd ff:ff:ff:ff:ff:ff
19: bond1: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue master ovs-system state UP mode DEFAULT qlen 1000
    link/ether 2c:c2:60:1a:18:15 brd ff:ff:ff:ff:ff:ff
20: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN mode DEFAULT 
    link/ether 02:42:e6:c0:ac:7e brd ff:ff:ff:ff:ff:ff
```

That's ``host`` networking in action.  All of the host's network interfaces,
IP addresses, routing tables, and firewall rules are visible within the
container.  (Actually, that's only true of network objects in the host's default
network namespace.  Interfaces, etc., in other network namespaces &mdash; such
as Neutron's ``qdhcp-...`` and ``qrouter-...`` namespaces are inaccessible from
within the container.)

#### Logging

Now let's look at logging.  OpenStack Nova services generally write their logs
to files in ``/var/log/nova``, so let's look at that directory.

```
()[nova@lab-controller01 /]$ ls -l /var/log/nova
total 52532
-rw-r--r--. 1 nova nova  4324332 Apr 17 21:01 nova-api-metadata.log
-rw-r--r--. 1 nova nova 10835400 Apr 13 12:00 nova-api-metadata.log.1
-rw-r--r--. 1 nova nova 10340388 Apr 17 21:01 nova-api.log
-rw-r--r--. 1 nova nova 10491224 Apr 13 11:00 nova-api.log.1
-rw-r--r--. 1 nova nova 10539568 Apr 13 04:00 nova-api.log.2
-rw-r--r--. 1 nova nova    29890 Apr 17 19:12 nova-conductor.log
-rw-r--r--. 1 nova nova    16783 Apr 17 19:12 nova-consoleauth.log
-rw-r--r--. 1 nova nova    75225 Apr 12 20:46 nova-manage.log
-rw-r--r--. 1 nova nova     1776 Apr 17 19:06 nova-novncproxy.log
-rw-r--r--. 1 nova nova   699968 Apr 17 21:01 nova-placement-api.log
-rw-r--r--. 1 nova nova        0 Apr 13 00:01 nova-rowsflush.log
-rw-r--r--. 1 nova nova   151254 Apr 17 21:00 nova-scheduler.log
```

We can see logs for all of the Nova services in that directory, but we know that
only ``nova-scheduler`` is running in this container.  What's going on?

Recall the ``Binds`` stanza from the ``docker inspect`` output above:

```
"Binds": [
    "/etc/pki/ca-trust/extracted:/etc/pki/ca-trust/extracted:ro",
    "/etc/pki/tls/certs/ca-bundle.crt:/etc/pki/tls/certs/ca-bundle.crt:ro",
    "/etc/pki/tls/cert.pem:/etc/pki/tls/cert.pem:ro",
    "/etc/ssh/ssh_known_hosts:/etc/ssh/ssh_known_hosts:ro",
    "/var/lib/kolla/config_files/nova_scheduler.json:/var/lib/kolla/config_files/config.json:ro",
    "/var/log/containers/nova:/var/log/nova",
    "/etc/hosts:/etc/hosts:ro",
    "/etc/localtime:/etc/localtime:ro",
    "/etc/pki/tls/certs/ca-bundle.trust.crt:/etc/pki/tls/certs/ca-bundle.trust.crt:ro",
    "/dev/log:/dev/log",
    "/etc/puppet:/etc/puppet:ro",
    "/var/lib/config-data/puppet-generated/nova/:/var/lib/kolla/config_files/src:ro",
    "/run:/run"
],
```

Note that the host directory ``/var/log/containers/nova`` has been  **bind**
mounted into the container at ``/var/log/nova``.  (A bind mount creates an
alternative path to an existing directory or file, similar to a symbolic link.)
Note also that this particular bind does **not** end with ``:ro``; writing to a
log file obviously requires read/write access.

After looking at the contents of this directory, it isn't much of a stretch to
guess that the other Nova containers running on this host are also bind mounting
this directory (``/var/log/containers/nova``) &mdash; effectively sharing it.

Let's add an entry to the Nova scheduler log file.  Later, we'll verify that it
is visible from the host operating system.

```
()[nova@lab-controller01 /]$ echo 'This is not a dance!' >> /var/log/nova/nova-scheduler.log
```

#### Configuration Files

We might assume that configuration files would use an approach similar to that
of the log directories and simply bind mount configuration directories into our
container.  Looking back at the ``Binds`` stanza again, we can see that only a
few files and directories under ``/etc`` are bind mountes &mdash; a small subset
of the files in our container's ``/etc`` directory tree.  So how do the correct
versions of all those files get there?

To answer that question, we need to look at our container's startup process.  We
saw with ``docker inspect`` that our container's entry point is something called
``kolla_start``.  Let's look at that script.

```
()[nova@lab-controller01 /]$ which kolla_start
/usr/local/bin/kolla_start

()[nova@lab-controller01 /]$ cat /usr/local/bin/kolla_start
#!/bin/bash
set -o errexit

# Processing /var/lib/kolla/config_files/config.json as root.  This is necessary
# to permit certain files to be controlled by the root user which should
# not be writable by the dropped-privileged user, especially /run_command
sudo -E kolla_set_configs
CMD=$(cat /run_command)
ARGS=""

if [[ ! "${!KOLLA_SKIP_EXTEND_START[@]}" ]]; then
    # Run additional commands if present
    . kolla_extend_start
fi

echo "Running command: '${CMD}${ARGS:+ $ARGS}'"
exec ${CMD} ${ARGS}
```

The comment at the beginning of the script provides a good explanation of what
it's doing &mdash; run (as ``root``) the ``kolla_set_configs`` script and then
run the command specified in ``/run_command``.  Let's start at the end and check
the command that will be run:

```
()[nova@lab-controller01 /]$ cat /run_command && echo
/usr/bin/nova-scheduler
```

(Apparently cool kids don't use newlines.)

OK, that makes sense.  Once ``kolla_set_configs`` has done it's work, this
container (``nova_scheduler``) will run ``/usr/bin/nova-scheduler``.  Now let's
look at what ``kolla_set_configs`` is doing.

Rather than reading all 422 lines of ``/usr/local/bin/kolla_set_configs``, let's
look at the JSON file that tells it what to do (as noted in the comment above).
First, note that ``/var/lib/kolla/config_files/config.json`` is bind mounted
from ``/var/lib/kolla/config_files/nova_scheduler.json``.  That naming
convention allows different ``config.json`` files, for different containers, to
coexist in the host's ``/var/lib/kolla/config_files`` directory.  What is
actually in that file?

```
()[nova@lab-controller01 /]$ cat /var/lib/kolla/config_files/config.json
cat: /var/lib/kolla/config_files/config.json: Permission denied
```

Oops.  Recall that ``kolla_start`` uses ``sudo`` to run ``kolla_set_configs``
with ``root`` privileges, and we're currently running as the ``nova`` user.
Let's exit our containerized shell, so that we can start a new one as ``root``.

```
()[nova@lab-controller01 /]$ exit
exit
[heat-admin@lab-controller01 ~]$
```

Before starting our new shell, let's check that our log entry is visible from
the host operating system.

```
[heat-admin@lab-controller01 ~]$ grep dance /var/log/containers/nova/*
/var/log/containers/nova/nova-scheduler.log:This is not a dance!
```

Now that we've confirmed that, start the new shell.

```
[heat-admin@lab-controller01 ~]$ sudo docker exec -u root -it nova_scheduler /bin/sh
()[root@lab-controller01 /]$
```

Recall that ``kolla_start`` runs ``sudo -E kolla_set_configs``.  Let's look at
the rules that allow that.

```
()[root@lab-controller01 /]$ cat /etc/sudoers
# The idea here is a container service adds their UID to the kolla group
# via usermod -a -G kolla <uid>.  Then the kolla_start may run
# kolla_set_configs via sudo as the root user which is necessary to protect
# the immutability of the container

# anyone in the kolla group may sudo -E (set the environment)
Defaults: %kolla setenv

# root may run any commands via sudo as the network seervice user.  This is
# neededfor database migrations of existing services which have not been
# converted to run as a non-root user, but instead do that via sudo -E glance
root ALL=(ALL) ALL

# anyone in the kolla group may run /usr/local/bin/kolla_set_configs as the
# root user via sudo without password confirmation
%kolla ALL=(root) NOPASSWD: /usr/local/bin/kolla_set_configs

#includedir /etc/sudoers.d
```

Now we can look at ``config.json``.  (``jq`` isn't available in the container,
so we'll use a Python module to format the JSON.)

```
()[root@lab-controller01 /]$ cat /var/lib/kolla/config_files/config.json | python -m json.tool
{
    "command": "/usr/bin/nova-scheduler",
    "config_files": [
        {
            "dest": "/",
            "merge": true,
            "preserve_properties": true,
            "source": "/var/lib/kolla/config_files/src/*"
        }
    ],
    "permissions": [
        {
            "owner": "nova:nova",
            "path": "/var/log/nova",
            "recurse": true
        }
    ]
}
```

This is pretty straitforward.  ``kolla_set_configs`` is going to copy the
contents of ``/var/lib/kolla/config_files/src`` (which is a read-only bind mount
of ``/var/lib/config-data/puppet-generated/nova``) to the root directory of the
container, merging the two directory trees and preserving the properties of the
copied files (ownership, permissions, etc.).  It's also going to recursively
set the permissions of the ``/var/log/nova`` directory.

Why is the destination directory ``/``, rather than ``/etc``?  Let's look at the
files under ``/var/lib/kolla/config_files/src``.

```
()[root@lab-controller01 /]$ ls /var/lib/kolla/config_files/src
etc  var

()[root@lab-controller01 /]$ find /var/lib/kolla/config_files/src/var -type f
/var/lib/kolla/config_files/src/var/spool/cron/nova
/var/lib/kolla/config_files/src/var/www/cgi-bin/nova/nova-api
```

In addition to the expected files under ``/etc``, there are a couple of files
under ``/var`` that also need to be copied.  (This could likely have also been
accomplished by having two elements in the ``config_files`` array &mdash; one
for ``/etc`` and one for ``/var``.)

This approach gives our containerized application exactly what it expects
&mdash; a read/write set of configuration files (depending on file ownership and
permissions) &mdash; while keeping the container immutable.  If one of the
containerized services in OSP 12 were to be compromised and made to corrupt one
of it's configuration files, the configuration could be restored to the desired
state by simply restarting the container.

A final note about ``kolla_set_configs`` &mdash; All of its actions are logged
to ``stderr``/``stdout``, and they can be viewed with the ``docker logs``
command.

```
()[root@lab-controller01 /]$ exit
exit

[heat-admin@lab-controller01 ~]$ sudo docker logs nova_scheduler 2>&1 | less
INFO:__main__:Loading config file at /var/lib/kolla/config_files/config.json
INFO:__main__:Validating config file
INFO:__main__:Kolla config strategy set to: COPY_ALWAYS
INFO:__main__:Copying service configuration files
INFO:__main__:Creating directory /etc/httpd
INFO:__main__:Creating directory /etc/httpd/conf.d
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/httpd/conf.d/10-nova_api_wsgi.conf to /etc/httpd/conf.d/10-nova_api_wsgi.conf
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/httpd/conf.d/ssl.conf to /etc/httpd/conf.d/ssl.conf
INFO:__main__:Creating directory /etc/httpd/conf.modules.d
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/httpd/conf.modules.d/access_compat.load to /etc/httpd/conf.modules.d/access_compat.load
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/httpd/conf.modules.d/actions.load to /etc/httpd/conf.modules.d/actions.load
INFO:__main__:Copying /var/lib/kolla/config_files/src/etc/httpd/conf.modules.d/alias.conf to /etc/httpd/conf.modules.d/alias.conf
(...)
INFO:__main__:Writing out command to execute
INFO:__main__:Setting permission for /var/log/nova
INFO:__main__:Setting permission for /var/log/nova/nova-manage.log
(...)
```

### Other Node Types

Thus far, we've only looked at one of our controller nodes.  Before moving on to
the next lab, let's take a quick look at our other two node types &mdash;
compute and Ceph storage.

#### Compute Nodes

First, log out of the controller and log in to the compute node.

```
[heat-admin@lab-controller01 ~]$ exit
logout
Connection to 172.16.0.32 closed.

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

(undercloud) [stack@undercloud ~]$ ssh heat-admin@172.16.0.25
Last login: Thu Apr 12 22:56:44 2018 from 172.16.0.1
```

List the running containers.

```
[heat-admin@lab-compute01 ~]$ sudo docker ps
CONTAINER ID        IMAGE                                                                  COMMAND             CREATED             STATUS                  PORTS               NAMES
2b24ef01f508        172.16.0.1:8787/rhosp12/openstack-cron:12.0-20180309.1                 "kolla_start"       6 days ago          Up 22 hours                                 logrotate_crond
eadec14e872f        172.16.0.1:8787/rhosp12/openstack-nova-compute:12.0-20180309.1         "kolla_start"       6 days ago          Up 22 hours (healthy)                       nova_migration_target
0cc416053d1f        172.16.0.1:8787/rhosp12/openstack-ceilometer-compute:12.0-20180309.1   "kolla_start"       6 days ago          Up 22 hours                                 ceilometer_agent_compute
51c5a1f97ade        172.16.0.1:8787/rhosp12/openstack-nova-compute:12.0-20180309.1         "kolla_start"       6 days ago          Up 22 hours (healthy)                       nova_compute
b55823464afc        172.16.0.1:8787/rhosp12/openstack-nova-libvirt:12.0-20180309.1         "kolla_start"       6 days ago          Up 22 hours                                 nova_libvirt
7a9b459e16af        172.16.0.1:8787/rhosp12/openstack-nova-libvirt:12.0-20180309.1         "kolla_start"       6 days ago          Up 22 hours                                 nova_virtlogd
```

As one might expect, we see a number of Nova compute and libvirt containers
running.  All of them are using ``kolla_start`` to set up their configuration
files.  (The ``qemu-kvm`` process that represents our test instance, however, is
not running in a container.)

Also as expected, the compute services make extensive use of privileged
containers to manage virtual machines.

```
[heat-admin@lab-compute01 ~]$ for TAINER in `sudo docker ps --format '{{ .Names }}'` ; do echo -n "${TAINER}:  " ; sudo docker inspect $TAINER | jq .[0].HostConfig.Privileged ; done
logrotate_crond:  true
nova_migration_target:  true
ceilometer_agent_compute:  false
nova_compute:  true
nova_libvirt:  true
nova_virtlogd:  true
```

#### Ceph Storage Nodes

Now let's look at the containers on one of our Ceph storage nodes.

```
[heat-admin@lab-compute01 ~]$ exit
logout
Connection to 172.16.0.25 closed.

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

(undercloud) [stack@undercloud ~]$ ssh heat-admin@172.16.0.31
[heat-admin@lab-ceph01 ~]$

[heat-admin@lab-ceph01 ~]$ sudo docker ps
CONTAINER ID        IMAGE                                                    COMMAND             CREATED             STATUS              PORTS               NAMES
9012f9714a02        172.16.0.1:8787/ceph/rhceph-2-rhel7:latest               "/entrypoint.sh"    23 hours ago        Up 23 hours                             ceph-osd-lab-ceph01-vdb
e34e9894d493        172.16.0.1:8787/ceph/rhceph-2-rhel7:latest               "/entrypoint.sh"    23 hours ago        Up 23 hours                             ceph-osd-lab-ceph01-vdc
e54c65b75b77        172.16.0.1:8787/rhosp12/openstack-cron:12.0-20180309.1   "kolla_start"       6 days ago          Up 23 hours                             logrotate_crond
```

We see that ``ceph-ansible`` has created a separate container for each Ceph OSD,
each of which corresponds to a single data disk.

And just like the Ceph monitor containers on our controllers, each of these
containers is a separate ``systemd`` unit, rather than being automatically
started by the Docker daemon.

```
[heat-admin@lab-ceph01 ~]$ sudo docker inspect ceph-osd-lab-ceph01-vdb | jq .[0].HostConfig.RestartPolicy
{
  "MaximumRetryCount": 0,
  "Name": "no"
}

[heat-admin@lab-ceph01 ~]$ systemctl list-units | grep ceph-osd
  ceph-osd@vdb.service                                                                                  loaded active running   Ceph OSD
  ceph-osd@vdc.service                                                                                  loaded active running   Ceph OSD
  
[heat-admin@lab-ceph01 ~]$ systemctl status ceph-osd@vdb.service
● ceph-osd@vdb.service - Ceph OSD
   Loaded: loaded (/etc/systemd/system/ceph-osd@.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2018-04-18 19:58:27 UTC; 23h ago
  Process: 2292 ExecStartPre=/usr/bin/docker rm -f ceph-osd-lab-ceph01-%i (code=exited, status=1/FAILURE)
  Process: 2277 ExecStartPre=/usr/bin/docker stop ceph-osd-lab-ceph01-%i (code=exited, status=1/FAILURE)
 Main PID: 2308 (ceph-osd-run.sh)
   CGroup: /system.slice/system-ceph\x2dosd.slice/ceph-osd@vdb.service
           ├─2308 /bin/bash /usr/share/ceph-osd-run.sh vdb
           └─2356 /usr/bin/docker-current run --rm --net=host --privileged=true --pid=host --memory=3g --cpu-quota=100000 -v /dev:/dev -v /etc/localtime:/etc/localtime:ro -v /var/lib/ceph:/var/lib/ceph -v /etc/ceph:/et...

Apr 18 19:59:37 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:37.839496 7faac8504700  0 -- :/1290131804 >> 172.17.3.201:6789/0 pipe(0x7faab80052b0 sd=5 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab800be10).fault
Apr 18 19:59:40 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:40.839784 7faac8403700  0 -- :/1290131804 >> 172.17.3.202:6789/0 pipe(0x7faab8009f60 sd=5 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab800df20).fault
Apr 18 19:59:43 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:43.839931 7faac8504700  0 -- :/1290131804 >> 172.17.3.203:6789/0 pipe(0x7faab800e970 sd=5 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab800fc30).fault
Apr 18 19:59:46 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:46.840187 7faac8403700  0 -- :/1290131804 >> 172.17.3.201:6789/0 pipe(0x7faab8010260 sd=4 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab8011520).fault
Apr 18 19:59:49 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:49.840657 7faac8504700  0 -- :/1290131804 >> 172.17.3.202:6789/0 pipe(0x7faab800e970 sd=5 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab800fc30).fault
Apr 18 19:59:55 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 19:59:55.841242 7faac8302700  0 -- 172.17.3.221:0/1290131804 >> 172.17.3.202:6789/0 pipe(0x7faab800e950 sd=4 :0 s=1 pgs=0 cs=0 l=1 c=0x7faab800fc10).fault
Apr 18 20:00:01 lab-ceph01 ceph-osd-run.sh[2308]: create-or-move updated item name 'osd.1' weight 0.04 at location {host=lab-ceph01,root=default} to crush map
Apr 18 20:00:01 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 20:00:01  /entrypoint.sh: SUCCESS
Apr 18 20:00:01 lab-ceph01 ceph-osd-run.sh[2308]: starting osd.1 at :/0 osd_data /var/lib/ceph/osd/ceph-1 /var/lib/ceph/osd/ceph-1/journal
Apr 18 20:00:03 lab-ceph01 ceph-osd-run.sh[2308]: 2018-04-18 20:00:03.813532 7f82948dba40 -1 osd.1 22 log_to_monitors {default=true}

[heat-admin@lab-ceph01 ~]$ cat /etc/systemd/system/ceph-osd@.service
# Please do not change this file directly since it is managed by Ansible and will be overwritten
[Unit]
Description=Ceph OSD
After=docker.service

[Service]
EnvironmentFile=-/etc/environment
ExecStartPre=-/usr/bin/docker stop ceph-osd-lab-ceph01-%i
ExecStartPre=-/usr/bin/docker rm -f ceph-osd-lab-ceph01-%i
ExecStart=/usr/share/ceph-osd-run.sh %i
ExecStop=-/usr/bin/docker stop ceph-osd-lab-ceph01-%i
Restart=always
RestartSec=10s
TimeoutStartSec=120
TimeoutStopSec=15

[Install]
WantedBy=multi-user.target

[heat-admin@lab-ceph01 ~]$ less /usr/share/ceph-osd-run.sh
(...)
########
# MAIN #
########

/usr/bin/docker run \
  --rm \
  --net=host \
  --privileged=true \
  --pid=host \
  --memory=3g \
  --cpu-quota=100000 \
  -v /dev:/dev \
  -v /etc/localtime:/etc/localtime:ro \
  -v /var/lib/ceph:/var/lib/ceph \
  -v /etc/ceph:/etc/ceph \
  $DOCKER_ENV \
  -e OSD_FILESTORE=1 \
  -e OSD_DMCRYPT=0 \
  -e CLUSTER=ceph \
  -e OSD_DEVICE=/dev/${1} \
  -e CEPH_DAEMON=OSD_CEPH_DISK_ACTIVATE \
   \
  --name=ceph-osd-lab-ceph01-${1} \
  172.16.0.1:8787/ceph/rhceph-2-rhel7:latest
(END)
```

This concludes our whirlwind tour of the deployed overcloud.  Log out of the
Ceph storage node, in preparation for the next lab.

```
[heat-admin@lab-ceph01 ~]$ exit
logout
Connection to 172.16.0.31 closed.
```

## Lab 3: Updating the Overcloud

In this lab, we'll start to explore how the containerization of OSP 12 affects
day-to-day operation of the environment.  Specifically, this lab will focus on
the process used to update the overcloud.

Note that Red Hat makes a strong distinction between **minor** updates (updating
software to the latest packages, maintenance level, minor version, etc.) and
**major** upgrades &mdash; such as upgrades from Red Hat Enterprise Linux 6 to
Red Hat Enterprise Linux 7 or Red Hat OpenStack Platform 11 to Red Hat OpenStack
Platform 12.

### Step 1: Update the Undercloud

This procedure has not changed significantly from previous releases.

#### Prepare the Undercloud

First, make a copy of ``/etc/sysconfig/docker``.  This is required because the
undercloud update process may overwrite this file, and it has been modified to
allow pulling container images from our ``bastion`` host, rather than directly
from Red Hat.

```
(undercloud) [stack@undercloud ~]$ sudo cp /etc/sysconfig/docker /etc/sysconfig/docker.bak
```

Next, stop the the OpenStack services on the undercloud (many of which run as
WSGI applications under ``httpd``).

```
(undercloud) [stack@undercloud ~]$ sudo systemctl stop 'openstack-*' 'neutron-*' httpd
```

Now, update the ``python-tripleoclient`` package.

```
(undercloud) [stack@undercloud ~]$ sudo yum -y update python-tripleoclient
Loaded plugins: search-disabled-repos
rhelosp-12.0-puddle                                                                               | 2.9 kB  00:00:00     
rhelosp-ceph-2.0-mon                                                                              | 2.9 kB  00:00:00     
rhelosp-ceph-2.0-osd                                                                              | 2.9 kB  00:00:00     
rhelosp-ceph-2.0-tools                                                                            | 2.9 kB  00:00:00     
rhelosp-rhel-7.4-extras                                                                           | 2.9 kB  00:00:00     
rhelosp-rhel-7.4-ha                                                                               | 2.9 kB  00:00:00     
rhelosp-rhel-7.4-server                                                                           | 2.9 kB  00:00:00     
(1/7): rhelosp-ceph-2.0-osd/primary_db                                                            | 153 kB  00:00:00     
(2/7): rhelosp-12.0-puddle/primary_db                                                             | 371 kB  00:00:00     
(3/7): rhelosp-ceph-2.0-tools/primary_db                                                          | 171 kB  00:00:00     
(4/7): rhelosp-rhel-7.4-ha/primary_db                                                             | 220 kB  00:00:00     
(5/7): rhelosp-ceph-2.0-mon/primary_db                                                            | 170 kB  00:00:00     
(6/7): rhelosp-rhel-7.4-extras/primary_db                                                         | 310 kB  00:00:00     
(7/7): rhelosp-rhel-7.4-server/primary_db                                                         |  37 MB  00:00:00     
Resolving Dependencies
--> Running transaction check
---> Package python-tripleoclient.noarch 0:7.3.3-7.el7ost will be updated
---> Package python-tripleoclient.noarch 0:7.3.8-1.el7ost will be an update
--> Finished Dependency Resolution

Dependencies Resolved

=========================================================================================================================
 Package                          Arch               Version                       Repository                       Size
=========================================================================================================================
Updating:
 python-tripleoclient             noarch             7.3.8-1.el7ost                rhelosp-12.0-puddle             259 k

Transaction Summary
=========================================================================================================================
Upgrade  1 Package

Total download size: 259 k
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
python-tripleoclient-7.3.8-1.el7ost.noarch.rpm                                                    | 259 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : python-tripleoclient-7.3.8-1.el7ost.noarch                                                            1/2 
  Cleanup    : python-tripleoclient-7.3.3-7.el7ost.noarch                                                            2/2 
  Verifying  : python-tripleoclient-7.3.8-1.el7ost.noarch                                                            1/2 
  Verifying  : python-tripleoclient-7.3.3-7.el7ost.noarch                                                            2/2 

Updated:
  python-tripleoclient.noarch 0:7.3.8-1.el7ost                                                                           

Complete!
```

#### Start the Update

Next, we need to run the ``openstack undercloud upgrade`` command (even though
we're actually performing a minor **update**).  This will update/install other
packages and make any other required changes to the undercloud.

This command will take approximately 15 minutes to complete, so we'll run it in
the background.

```
(undercloud) [stack@undercloud ~]$ openstack undercloud upgrade &> /tmp/undercloud-upgrade.out &
[1] 11606
```

#### Update Container Images

While that command is running, let's take our first look at OSP 12's update (and
deployment) mechanism for container images.

During deployment and updates, TripleO parameters are used to specify the image
used for each overcloud container.  In our deployment, these parameters are set
in ``templates/docker-registry.yaml``.

```
(undercloud) [stack@undercloud ~]$ grep -v '^#' templates/docker-registry.yaml

parameter_defaults:
  DockerAodhApiImage: 172.16.0.1:8787/rhosp12/openstack-aodh-api:12.0-20180309.1
  DockerAodhConfigImage: 172.16.0.1:8787/rhosp12/openstack-aodh-api:12.0-20180309.1
  DockerAodhEvaluatorImage: 172.16.0.1:8787/rhosp12/openstack-aodh-evaluator:12.0-20180309.1
  DockerAodhListenerImage: 172.16.0.1:8787/rhosp12/openstack-aodh-listener:12.0-20180309.1
  DockerAodhNotifierImage: 172.16.0.1:8787/rhosp12/openstack-aodh-notifier:12.0-20180309.1
  DockerCeilometerCentralImage: 172.16.0.1:8787/rhosp12/openstack-ceilometer-central:12.0-20180309.1
  DockerCeilometerComputeImage: 172.16.0.1:8787/rhosp12/openstack-ceilometer-compute:12.0-20180309.1
  DockerCeilometerConfigImage: 172.16.0.1:8787/rhosp12/openstack-ceilometer-central:12.0-20180309.1
  DockerCeilometerNotificationImage: 172.16.0.1:8787/rhosp12/openstack-ceilometer-notification:12.0-20180309.1
  DockerCephDaemonImage: 172.16.0.1:8787/ceph/rhceph-2-rhel7:latest
  DockerClustercheckConfigImage: 172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1
  DockerClustercheckImage: 172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1
  DockerCrondConfigImage: 172.16.0.1:8787/rhosp12/openstack-cron:12.0-20180309.1
  DockerCrondImage: 172.16.0.1:8787/rhosp12/openstack-cron:12.0-20180309.1
  DockerGlanceApiConfigImage: 172.16.0.1:8787/rhosp12/openstack-glance-api:12.0-20180309.1
  DockerGlanceApiImage: 172.16.0.1:8787/rhosp12/openstack-glance-api:12.0-20180309.1
  DockerGnocchiApiImage: 172.16.0.1:8787/rhosp12/openstack-gnocchi-api:12.0-20180309.1
  DockerGnocchiConfigImage: 172.16.0.1:8787/rhosp12/openstack-gnocchi-api:12.0-20180309.1
  DockerGnocchiMetricdImage: 172.16.0.1:8787/rhosp12/openstack-gnocchi-metricd:12.0-20180309.1
  DockerGnocchiStatsdImage: 172.16.0.1:8787/rhosp12/openstack-gnocchi-statsd:12.0-20180309.1
  DockerHAProxyConfigImage: 172.16.0.1:8787/rhosp12/openstack-haproxy:12.0-20180309.1
  DockerHAProxyImage: 172.16.0.1:8787/rhosp12/openstack-haproxy:12.0-20180309.1
  DockerHeatApiCfnConfigImage: 172.16.0.1:8787/rhosp12/openstack-heat-api-cfn:12.0-20180309.1
  DockerHeatApiCfnImage: 172.16.0.1:8787/rhosp12/openstack-heat-api-cfn:12.0-20180309.1
  DockerHeatApiConfigImage: 172.16.0.1:8787/rhosp12/openstack-heat-api:12.0-20180309.1
  DockerHeatApiImage: 172.16.0.1:8787/rhosp12/openstack-heat-api:12.0-20180309.1
  DockerHeatConfigImage: 172.16.0.1:8787/rhosp12/openstack-heat-api:12.0-20180309.1
  DockerHeatEngineImage: 172.16.0.1:8787/rhosp12/openstack-heat-engine:12.0-20180309.1
  DockerHorizonConfigImage: 172.16.0.1:8787/rhosp12/openstack-horizon:12.0-20180309.1
  DockerHorizonImage: 172.16.0.1:8787/rhosp12/openstack-horizon:12.0-20180309.1
  DockerInsecureRegistryAddress:
  - 172.16.0.1:8787
  DockerKeystoneConfigImage: 172.16.0.1:8787/rhosp12/openstack-keystone:12.0-20180309.1
  DockerKeystoneImage: 172.16.0.1:8787/rhosp12/openstack-keystone:12.0-20180309.1
  DockerMemcachedConfigImage: 172.16.0.1:8787/rhosp12/openstack-memcached:12.0-20180309.1
  DockerMemcachedImage: 172.16.0.1:8787/rhosp12/openstack-memcached:12.0-20180309.1
  DockerMysqlClientConfigImage: 172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1
  DockerMysqlConfigImage: 172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1
  DockerMysqlImage: 172.16.0.1:8787/rhosp12/openstack-mariadb:12.0-20180309.1
  DockerNovaApiImage: 172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1
  DockerNovaComputeImage: 172.16.0.1:8787/rhosp12/openstack-nova-compute:12.0-20180309.1
  DockerNovaConductorImage: 172.16.0.1:8787/rhosp12/openstack-nova-conductor:12.0-20180309.1
  DockerNovaConfigImage: 172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1
  DockerNovaConsoleauthImage: 172.16.0.1:8787/rhosp12/openstack-nova-consoleauth:12.0-20180309.1
  DockerNovaLibvirtConfigImage: 172.16.0.1:8787/rhosp12/openstack-nova-compute:12.0-20180309.1
  DockerNovaLibvirtImage: 172.16.0.1:8787/rhosp12/openstack-nova-libvirt:12.0-20180309.1
  DockerNovaMetadataImage: 172.16.0.1:8787/rhosp12/openstack-nova-api:12.0-20180309.1
  DockerNovaPlacementConfigImage: 172.16.0.1:8787/rhosp12/openstack-nova-placement-api:12.0-20180309.1
  DockerNovaPlacementImage: 172.16.0.1:8787/rhosp12/openstack-nova-placement-api:12.0-20180309.1
  DockerNovaSchedulerImage: 172.16.0.1:8787/rhosp12/openstack-nova-scheduler:12.0-20180309.1
  DockerNovaVncProxyImage: 172.16.0.1:8787/rhosp12/openstack-nova-novncproxy:12.0-20180309.1
  DockerPankoApiImage: 172.16.0.1:8787/rhosp12/openstack-panko-api:12.0-20180309.1
  DockerPankoConfigImage: 172.16.0.1:8787/rhosp12/openstack-panko-api:12.0-20180309.1
  DockerRabbitmqConfigImage: 172.16.0.1:8787/rhosp12/openstack-rabbitmq:12.0-20180309.1
  DockerRabbitmqImage: 172.16.0.1:8787/rhosp12/openstack-rabbitmq:12.0-20180309.1
  DockerRedisConfigImage: 172.16.0.1:8787/rhosp12/openstack-redis:12.0-20180309.1
  DockerRedisImage: 172.16.0.1:8787/rhosp12/openstack-redis:12.0-20180309.1
  DockerSwiftAccountImage: 172.16.0.1:8787/rhosp12/openstack-swift-account:12.0-20180309.1
  DockerSwiftConfigImage: 172.16.0.1:8787/rhosp12/openstack-swift-proxy-server:12.0-20180309.1
  DockerSwiftContainerImage: 172.16.0.1:8787/rhosp12/openstack-swift-container:12.0-20180309.1
  DockerSwiftObjectImage: 172.16.0.1:8787/rhosp12/openstack-swift-object:12.0-20180309.1
  DockerSwiftProxyImage: 172.16.0.1:8787/rhosp12/openstack-swift-proxy-server:12.0-20180309.1
```

We see that the overcloud nodes will use images from ``172.16.0.1:8787``, which
is the undercloud's IP address.  In OSP 12, the undercloud automatically runs a
container image registry.

```
(undercloud) [stack@undercloud ~]$ systemctl status docker-distribution
● docker-distribution.service - v2 Registry server for Docker
   Loaded: loaded (/usr/lib/systemd/system/docker-distribution.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2018-04-20 14:40:49 EDT; 1h 6min ago
 Main PID: 531 (registry)
   CGroup: /system.slice/docker-distribution.service
           └─531 /usr/bin/registry serve /etc/docker-distribution/registry/config.yml
```

We can also see that ``DockerInsecureRegistryAddress`` is set to
``172.16.0.1:8787``.  This will cause our overcloud nodes to use HTTP when
pulling images from the undercloud &mdash; avoiding the need to generate and
manage TLS certificates.

The container registry on the undercloud is provided as a convenience; there is
no requirement that it be used.  It is entirely possible to use a different
container image registry, such as Red Hat Satellite, Red Hat OpenShift Container
Platform, etc.

The other thing to note about ``docker-registry.yaml`` is the date-based tags (``12.0-20180309.1``)
that are used to specify the exact container image to use (other than the Ceph
container).  This tag is important to TripleO, as it will only update the
images used by the overcloud when the tag changes.  Thus, we do not follow the
common practice of simply using ``latest``.

So how did those images get into the registry running our undercloud?

It turns out that Red Hat OpenStack Platform 12 provides a number of utility
commands that help in this area.

* ``openstack overcloud container image tag discover`` can be used to discover
  the latest version of the OpenStack container images in the Red Hat Container
  Catalog.  (Unfortunately, it won't work with the image registry on our
  ``bastion`` host.)

* ``openstack overcloud container image prepare`` performs two functions.
  1. It creates a file that specifies the images to pulled from an upstream
     image registry, such as the Red Hat Container Catalog.  This file already
     exists on our undercloud as ``container-images.yaml``.
  1. It creates the ``docker-registry.yaml`` file used for overcloud deployment
     and updates.

* ``openstack overcloud container image upload`` pulls the images and pushes
  them into the undercloud registry.

Because we already have the ``container-images.yaml`` and
``docker-registry.yaml`` files from our deployment, we can simply use ``sed`` to
update the tags in those files.  The latest version of the container images in
the registry on our bastion host is ``20180319.1``.

```
(undercloud) [stack@undercloud ~]$ sed -i s/20180309.1/20180319.1/ container-images.yaml
(undercloud) [stack@undercloud ~]$ sed -i s/20180309.1/20180319.1/ templates/docker-registry.yaml
```

Before running ``openstack overcloud container image upload``, we need to check
the status of the ``docker`` service.  The ``openstack undercloud upgrade``
process that is running in the background will modify ``/etc/sysconfig/docker``
and restart the service; we need wait until it has performed these steps and
then revert the change.

First, check on the configuration file.

```
(undercloud) [stack@undercloud ~]$ diff -u /etc/sysconfig/docker.bak /etc/sysconfig/docker
--- /etc/sysconfig/docker.bak   2018-04-20 18:08:13.687203918 -0400
+++ /etc/sysconfig/docker       2018-04-20 18:09:27.782825832 -0400
@@ -29,5 +29,4 @@
 #DOCKERDBINARY=/usr/bin/dockerd-latest
 #DOCKER_CONTAINERD_BINARY=/usr/bin/docker-containerd-latest
 #DOCKER_CONTAINERD_SHIM_BINARY=/usr/bin/docker-containerd-shim-latest
-INSECURE_REGISTRY="--insecure-registry 172.16.0.1:8787 --insecure-registry 172.16.0.11:8787 --insecure-registry 192.168.1.10:5000"
-
+INSECURE_REGISTRY="--insecure-registry 172.16.0.1:8787 --insecure-registry 172.16.0.11:8787"
```

If the ``diff`` command produces no output, the background process has not yet
modified the file.  You can use the ``watch`` command to wait until it does so.

```
(undercloud) [stack@undercloud ~]$ watch -n 5 diff -u /etc/sysconfig/docker.bak /etc/sysconfig/docker
```

Wait until the ``diff`` command produces output and terminate ``watch`` with
**Ctrl-C**.

``openstack undercloud upgrade`` restarts the ``docker`` daemon shortly after
it modifies ``/etc/sydconfig/docker``.  Verify that the service has recently
been restarted.

```
(undercloud) [stack@undercloud ~]$ systemctl status docker | grep Active
   Active: active (running) since Fri 2018-04-20 18:09:30 EDT; 1min ago
```

Now revert the change to the configuration file and restart (re-restart?) the
daemon.

```
(undercloud) [stack@undercloud ~]$ sudo cp -f /etc/sysconfig/docker.bak /etc/sysconfig/docker

(undercloud) [stack@undercloud ~]$ sudo systemctl restart docker
```

Pull the new container images from the bastion host and push them into the
image registry on the undercloud.

```
(test@overcloud) [stack@undercloud ~]$ openstack overcloud container image upload --verbose --config-file container-images.yaml
START with options: [u'overcloud', u'container', u'image', u'upload', u'--verbose', u'--config-file', u'container-images.yaml']
command: overcloud container image upload -> tripleoclient.v1.container_image.UploadImage (auth=False)
Using config files: [u'container-images.yaml']
imagename: 192.168.1.10:5000/rhosp12/openstack-aodh-api:12.0-20180319.1
Completed upload for docker image 192.168.1.10:5000/rhosp12/openstack-aodh-api:12.0-20180319.1
(...)
imagename: 192.168.1.10:5000/rhosp12/openstack-swift-proxy-server:12.0-20180319.1
Completed upload for docker image 192.168.1.10:5000/rhosp12/openstack-swift-proxy-server:12.0-20180319.1
imagename: 192.168.1.10:5000/ceph/rhceph-2-rhel7:latest
Completed upload for docker image 192.168.1.10:5000/ceph/rhceph-2-rhel7:latest
END return value: 0
```

You can view a list of all images known to the local ``docker`` daemon with the
``docker images`` command.  For each OpenStack image, it should now show both
the old (``12.0-20180309.1``) and new (``12.0-20180319.1``) tags for both the
bastion host (``192.168.1.10:5000``) and local (``172.16.0.1:8787``) registries.
For example:

```
(test@overcloud) [stack@undercloud ~]$  docker images | grep openstack-nova-api
172.16.0.1:8787/rhosp12/openstack-nova-api                    12.0-20180319.1     bc157a405e32        4 weeks ago         758 MB
192.168.1.10:5000/rhosp12/openstack-nova-api                  12.0-20180319.1     bc157a405e32        4 weeks ago         758 MB
172.16.0.1:8787/rhosp12/openstack-nova-api                    12.0-20180309.1     7f9d9a559a00        6 weeks ago         757 MB
192.168.1.10:5000/rhosp12/openstack-nova-api                  12.0-20180309.1     7f9d9a559a00        6 weeks ago         757 MB
```

#### Verify Undercloud Update Completion

Now let's check on the ``openstack undercloud upgrade`` command that we ran as
a background process.

```
(test@overcloud) [stack@undercloud ~]$ jobs
[1]+  Running                 openstack undercloud upgrade &>/tmp/undercloud-upgrade.out &
```

This indicates that the process is still running.  It's progress can be
monitored with:

```
(test@overcloud) [stack@undercloud ~]$  tailf /tmp/undercloud-upgrade.out
(...)
```

If ``jobs`` produces no output, it indicates that the update process has
finished.  Check the output file, to verify that it was successful.

```
(test@overcloud) [stack@undercloud ~]$ tail -n 20 /tmp/undercloud-upgrade.out
2018-04-23 14:09:03,962 INFO: Configuring Mistral workbooks
2018-04-23 14:09:31,972 INFO: Mistral workbooks configured successfully
2018-04-23 14:09:32,537 INFO: Not creating default plan "overcloud" because it already exists.
2018-04-23 14:09:32,537 INFO: Configuring an hourly cron trigger for tripleo-ui logging
2018-04-23 14:09:35,585 INFO: Added _member_ role to admin user
2018-04-23 14:09:36,244 INFO: Starting and waiting for validation groups ['post-upgrade'] 
2018-04-23 14:10:27,207 INFO: 
#############################################################################
Undercloud upgrade complete.

The file containing this installation's passwords is at
/home/stack/undercloud-passwords.conf.

There is also a stackrc file at /home/stack/stackrc.

These files are needed to interact with the OpenStack services, and should be
secured.

#############################################################################
```

Reboot the undercloud.

```
(test@overcloud) [stack@undercloud ~]$ sudo reboot
PolicyKit daemon disconnected from the bus.
We are no longer a registered authentication agent.
Connection to undercloud.example.com closed by remote host.
Connection to undercloud.example.com closed.
```

#### Update Overcloud Images

Log back into the undercloud and source ``stackrc``.  (The ``stack`` user's
password is ``redhat``.)

```
[lab-user@bastion-3998 ~]$ ssh stack@undercloud.example.com
stack@undercloud.example.com's password: 
Last login: Mon Apr 23 13:08:56 2018 from bastion.example.com
[stack@undercloud ~]$

[stack@undercloud ~]$ . stackrc
(undercloud) [stack@undercloud ~]$
```

> **NOTE:** The remainder of this section is optional in this lab.  We will not
> be deploying any additional nodes (scaling out), so we do not strictly need
> to update the image used for our overcloud nodes.

Determine whether the undercloud update installed updated image packages.

```
(undercloud) [stack@undercloud ~]$ sudo grep images /var/log/yum.log
Mar 15 20:37:56 Installed: rhosp-director-images-ipa-12.0-20180309.2.el7ost.noarch
Mar 15 20:38:47 Installed: rhosp-director-images-12.0-20180309.2.el7ost.noarch
Apr 23 14:01:05 Installed: rhosp-director-images-ipa-12.0-20180402.1.el7ost.noarch
Apr 23 14:01:55 Installed: rhosp-director-images-12.0-20180402.1.el7ost.noarch
```

It did, so let's update the images used to deploy new nodes.  First, uninstall
the old RPMs.  (This is not absolutely necessary, but the old packages do use a
fair bit of disk space.)

```
(undercloud) [stack@undercloud ~]$ rpm -qa rhosp-director-images*
rhosp-director-images-ipa-12.0-20180309.2.el7ost.noarch
rhosp-director-images-12.0-20180402.1.el7ost.noarch
rhosp-director-images-12.0-20180309.2.el7ost.noarch
rhosp-director-images-ipa-12.0-20180402.1.el7ost.noarch

(undercloud) [stack@undercloud ~]$ sudo yum -y remove rhosp-director-images-ipa-12.0-20180309.2.el7ost.noarch rhosp-director-images-12.0-20180309.2.el7ost.noarch
Loaded plugins: search-disabled-repos
Resolving Dependencies
--> Running transaction check
---> Package rhosp-director-images.noarch 0:12.0-20180309.2.el7ost will be erased
---> Package rhosp-director-images-ipa.noarch 0:12.0-20180309.2.el7ost will be erased
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================
 Package                                     Arch                     Version                                     Repository                              Size
===============================================================================================================================================================
Removing:
 rhosp-director-images                       noarch                   12.0-20180309.2.el7ost                      @rhelosp-12.0-puddle                   1.3 G
 rhosp-director-images-ipa                   noarch                   12.0-20180309.2.el7ost                      @rhelosp-12.0-puddle                   369 M

Transaction Summary
===============================================================================================================================================================
Remove  2 Packages

Installed size: 1.6 G
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Erasing    : rhosp-director-images-12.0-20180309.2.el7ost.noarch                                                                                         1/2 
  Erasing    : rhosp-director-images-ipa-12.0-20180309.2.el7ost.noarch                                                                                     2/2 
  Verifying  : rhosp-director-images-ipa-12.0-20180309.2.el7ost.noarch                                                                                     1/2 
  Verifying  : rhosp-director-images-12.0-20180309.2.el7ost.noarch                                                                                         2/2 

Removed:
  rhosp-director-images.noarch 0:12.0-20180309.2.el7ost                        rhosp-director-images-ipa.noarch 0:12.0-20180309.2.el7ost                       

Complete!
```

Change to the ``images`` directory and extract the new images.

```
(undercloud) [stack@undercloud ~]$ cd images

(undercloud) [stack@undercloud images]$ rpm -qa rhosp-director-images*
rhosp-director-images-12.0-20180402.1.el7ost.noarch
rhosp-director-images-ipa-12.0-20180402.1.el7ost.noarch

(undercloud) [stack@undercloud images]$ rpm -ql rhosp-director-images-12.0-20180402.1.el7ost.noarch rhosp-director-images-ipa-12.0-20180402.1.el7ost.noarch
/usr/share/rhosp-director-images/overcloud-full-12.0-20180402.1.el7ost.tar
/usr/share/rhosp-director-images/version-12.0-20180402.1.el7ost.txt
/usr/share/rhosp-director-images/ironic-python-agent-12.0-20180402.1.el7ost.tar

(undercloud) [stack@undercloud images]$ tar xvf /usr/share/rhosp-director-images/overcloud-full-12.0-20180402.1.el7ost.tar
overcloud-full.qcow2
overcloud-full.initrd
overcloud-full.vmlinuz
overcloud-full-rpm.manifest
overcloud-full-signature.manifest

(undercloud) [stack@undercloud images]$ tar xvf /usr/share/rhosp-director-images/ironic-python-agent-12.0-20180402.1.el7ost.tar
ironic-python-agent.initramfs
ironic-python-agent.kernel
```

Customize the overcloud image, so it will use the bastion host as its ``yum``
repository.

```
(undercloud) [stack@undercloud images]$ virt-customize -a overcloud-full.qcow2 --copy-in /etc/yum.repos.d/lab.repo:/etc/yum.repos.d/ --selinux-relabel
[   0.0] Examining the guest ...
[   3.4] Setting a random seed
[   3.5] Copying: /etc/yum.repos.d/lab.repo to /etc/yum.repos.d/
[   3.5] SELinux relabelling
[  57.3] Finishing off
```

Upload the new images.

```
(undercloud) [stack@undercloud images]$ openstack overcloud image upload --update-existing
Image "overcloud-full-vmlinuz" was uploaded.
+--------------------------------------+------------------------+-------------+---------+--------+
|                  ID                  |          Name          | Disk Format |   Size  | Status |
+--------------------------------------+------------------------+-------------+---------+--------+
| 37aa394f-e583-4bed-9947-d67b200d5b03 | overcloud-full-vmlinuz |     aki     | 6381872 | active |
+--------------------------------------+------------------------+-------------+---------+--------+
Image "overcloud-full-initrd" was uploaded.
+--------------------------------------+-----------------------+-------------+----------+--------+
|                  ID                  |          Name         | Disk Format |   Size   | Status |
+--------------------------------------+-----------------------+-------------+----------+--------+
| 0d830007-5bd3-41b6-97ce-8dab1035fb60 | overcloud-full-initrd |     ari     | 62806684 | active |
+--------------------------------------+-----------------------+-------------+----------+--------+
Image "overcloud-full" was uploaded.
+--------------------------------------+----------------+-------------+------------+--------+
|                  ID                  |      Name      | Disk Format |    Size    | Status |
+--------------------------------------+----------------+-------------+------------+--------+
| c22fe1c8-0f6e-4dac-8e58-16fae39ac738 | overcloud-full |    qcow2    | 1322778624 | active |
+--------------------------------------+----------------+-------------+------------+--------+
Image "bm-deploy-kernel" was uploaded.
+--------------------------------------+------------------+-------------+---------+--------+
|                  ID                  |       Name       | Disk Format |   Size  | Status |
+--------------------------------------+------------------+-------------+---------+--------+
| 8c641282-2db0-4aa5-9461-2fc354b69e79 | bm-deploy-kernel |     aki     | 6381872 | active |
+--------------------------------------+------------------+-------------+---------+--------+
Image "bm-deploy-ramdisk" was uploaded.
+--------------------------------------+-------------------+-------------+-----------+--------+
|                  ID                  |        Name       | Disk Format |    Size   | Status |
+--------------------------------------+-------------------+-------------+-----------+--------+
| f7fcad54-801c-45ef-b692-ccb284095cc9 | bm-deploy-ramdisk |     ari     | 426133936 | active |
+--------------------------------------+-------------------+-------------+-----------+--------+
```

Update the Ironic node configuration to use the new deployment kernal and
RAM disk.

```
(undercloud) [stack@undercloud images]$ NODES=`openstack baremetal node list -f csv -c Name --quote none | sed 1d | paste -s -d ' '`

(undercloud) [stack@undercloud images]$ openstack overcloud node configure $NODES
Started Mistral Workflow tripleo.baremetal.v1.configure. Execution ID: d987630f-389f-4a44-898c-a7a3f45d7b6e
Waiting for messages on queue 'f83ccfc6-8425-4d4e-b730-1741c62a9247' with no timeout.
Successfully configured the nodes.
```

Finally, delete the old images from the undercloud's Glance service.

```
(undercloud) [stack@undercloud images]$  openstack image list
+--------------------------------------+-----------------------------------------+--------+
| ID                                   | Name                                    | Status |
+--------------------------------------+-----------------------------------------+--------+
| 8c641282-2db0-4aa5-9461-2fc354b69e79 | bm-deploy-kernel                        | active |
| 3ac4bb8b-1197-4893-9450-854de210a35e | bm-deploy-kernel_20180316T123554Z       | active |
| f7fcad54-801c-45ef-b692-ccb284095cc9 | bm-deploy-ramdisk                       | active |
| 25c9caec-6344-4ece-bd39-09e6d776d0ae | bm-deploy-ramdisk_20180316T123556Z      | active |
| c22fe1c8-0f6e-4dac-8e58-16fae39ac738 | overcloud-full                          | active |
| 0d830007-5bd3-41b6-97ce-8dab1035fb60 | overcloud-full-initrd                   | active |
| a6410509-a8c2-470e-9875-a1cfd078668d | overcloud-full-initrd_20180316T123533Z  | active |
| 37aa394f-e583-4bed-9947-d67b200d5b03 | overcloud-full-vmlinuz                  | active |
| 72d6708b-6706-488c-94ef-2c3e54eb2848 | overcloud-full-vmlinuz_20180316T123531Z | active |
| 37bdc1c0-3b85-4c2e-a37f-0638faf442a8 | overcloud-full_20180412T194559Z         | active |
+--------------------------------------+-----------------------------------------+--------+

(undercloud) [stack@undercloud images]$ IMAGES=`openstack image list -f csv -c Name --quote none | grep 2018`

(undercloud) [stack@undercloud images]$ for IMAGE in $IMAGES ; do openstack image delete $IMAGE ; done

(undercloud) [stack@undercloud images]$ openstack image list
+--------------------------------------+------------------------+--------+
| ID                                   | Name                   | Status |
+--------------------------------------+------------------------+--------+
| 8c641282-2db0-4aa5-9461-2fc354b69e79 | bm-deploy-kernel       | active |
| f7fcad54-801c-45ef-b692-ccb284095cc9 | bm-deploy-ramdisk      | active |
| c22fe1c8-0f6e-4dac-8e58-16fae39ac738 | overcloud-full         | active |
| 0d830007-5bd3-41b6-97ce-8dab1035fb60 | overcloud-full-initrd  | active |
| 37aa394f-e583-4bed-9947-d67b200d5b03 | overcloud-full-vmlinuz | active |
+--------------------------------------+------------------------+--------+
```

And return to our home directory.

```
(undercloud) [stack@undercloud images]$ cd
(undercloud) [stack@undercloud ~]$
```

### Step 2: Update the Overcloud

Like painting, most of the work involved in an overcloud update actually occurs
during the preparation.  The steps that actually update the overcloud are fairly
simple.

#### Update the Plan

## Lab 4: Troubleshooting and Testing

## Lab 5: Deploying a New Overcloud

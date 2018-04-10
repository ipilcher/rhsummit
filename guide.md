![Red Hat logo](/images/redhat-33.png)

# Red Hat Summit 2018 - San Francisco, California

**Title:** Understanding Containerized Red Hat OpenStack Platform (L1018)  
**Date:** May 8, 2017  
**Authors:**
* Ian Pilcher <<ipilcher@redhat.com>>
* Greg Charot <<gcharot@redhat.com>>
* Jacob Liberman <<jliberma@redhat.com>>
* Rhyx Oxenham <<roxenham@redhat.com>>

## Lab Contents

* **Lab 1:** Introduction
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

**We are not talking about running containerized applications _ON_ OpenStack.**
OpenStack is a fantastic platform for running containerized applications and
container orchestration engines, such as Red Hat OpenShift Container Platform,
but that is not the subject of this lab.
 
Instead, this lab is concerned with running the services that manage an
OpenStack environment within containers.  Why is Red Hat doing this?

Running OpenStack (and other) services within containers provides a number of
benefits:

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
* **Flexibility** - Red Hat OpenStack Platform 10 introduced 

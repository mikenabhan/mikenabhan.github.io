---
layout: post
title:  "The Solar Powered Kubernetes Cluster"
---

# The Solar Powered Kubernetes Cluster

This post is the first part in a series.
The purpose of this series  is to show that on-prem/private cloud/self-hosted kubernetes doesn't have to be hard and you don't have to make too many compromises.

## Things that are hard in on-prem kubernetes

* Storage
* Networking
    * LoadBalancers
    * and by extension: Ingress
    * Control Plane Access
* Cluster Autoscaling
* Bootstrapping new nodes

## The Golden Rule of High Availability

`One is none and 3 is one`

This is an oversimplification, but for most high availability platforms, algorithms, and tools the minimum number of members you might want in a cluster is 3, as this allows the failure of one node while still allowing quorum.  You will see this reflected in much of the architecture that follows.

## The hardware stack

At its core, the hardware stack is comprised of [three intel NUCs running Proxmox](/assets/images/nuc-cluster.png).  Over the years, this NUC cluster has taken a few evolutions, such as a separate corosync network, but I've since simplified it.


## The solar setup

Right now, my house has solar panels on its roof. I don't currently have whole home batteries, so part of the cluster runs on batteries, and the other part will power off when I am not producing solar in excess of demand.  This is by design, to inject chaos into the cluster.  Think of it like Spot instances on your cloud provider.

## Cluster Design Philosophy

This cluster follows a gitops philosophy.  This means that the application state will be completely driven by what you see in git.  There may also be some gitops-style terraform run from argo workflows.  
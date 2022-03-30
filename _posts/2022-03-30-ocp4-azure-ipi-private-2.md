---
layout: post
title: Secure your Private OpenShift clusters with Azure Firewall and Hub & Spoke architectures
date: 2022-03-30
type: post
published: false
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How can we generate a

This is the second blog post of the series of OpenShift deployments in Azure. Check the first part in [Deep dive in Private OpenShift 4 clusters deployments in Azure](https://rcarrata.com/openshift/ocp4-azure-ipi-private/).

Let's dig in!

## 1. Overview

As we checked in the previous blog post, we have several options to deploy our Private OpenShift clusters, depending of their Outbound / User-Defined Routing modes that we want to use.

We analyzed a couple of this User Defined Routing (UDR), including Azure Load Balancer (default), and also the use of the Proxy Configuration for the OpenShift cluster installations.

Now we're analyzing two of the most complex but flexible and interesting options too:

* Private cluster with network address translation (NAT Gateway)

* Private cluster with Azure Firewall

Both are super useful and try to solve the same issue: manage and control the egress routing traffic from the OCP cluster to Internet, but the point of view of the implementation are quite different.

In this blog post we will focus in both, but we will deep dive in the Azure Firewall mode, because it's a bit more flexible and can control and manage in a fine grain the traffic generated.

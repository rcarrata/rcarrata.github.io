---
layout: post
title: Secure your Private OpenShift clusters with Azure Firewall and Hub-Spoke architectures
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

How can we generate Private clusters OpenShift in Azure, secured by Azure Firewall? How we can ensure that we have controlled and managed all the traffic of outbound in our cluster, having the ability to

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

[![](/images/azurefw0.png "azurefw0.png")]({{site.url}}/images/azurefw0.png)

## 2. Egress / Outbound Mode with NAT Gateway

Let's analyze the third option (remember that we've checked in the first blog post the two initial ones): using Azure VNET network address translation (NAT Gateway) to provide the outbound Internet routing for the subnets of the OCP cluster.

[![](/images/azureipi16.png "azureipi16.png")]({{site.url}}/images/azureipi16.png)

As we can see we have our VNet setup with an [Azure NAT Gateway](https://docs.microsoft.com/en-us/azure/virtual-network/nat-gateway/nat-overview) and the user-defined routing configured, and there are NOT any public endpoint / IPs.

VNet NAT Gateway is a fully managed and highly resilient Network Address Translation (NAT) service. Virtual Network NAT simplifies outbound Internet connectivity for virtual networks. When configured on a subnet, all outbound connectivity uses the Virtual Network NAT's static public IP addresses.

So in a nutshell, when using the default route table for subnets, with 0.0.0.0/0 populated automatically by Azure, all Azure API requests are routed over Azure’s internal network, though the Azure NAT Gateway.

But if there is a NAT, why don't use the Azure Load Balancer SNAT method as we depicted in the first part of the blog post? It's only by the public IPs limitation?

Well, Virtual Network NAT is the recommended method for outbound connectivity. A NAT gateway doesn't have the same limitations of SNAT port exhaustion as does default outbound access and outbound rules of a load balancer.

Other benefit it's the security: with a NAT gateway, individual VMs or other compute resources, don't need public IP addresses and can remain private. Resources without a public IP address can still reach external sources outside the virtual network.

Finally NAT Gateway is a fully is a fully managed and distributed service, providing resiliency and scalability on demand, scaling when it's needed.

## 3. Egress / Outbound Mode with Azure Firewall

Let's analyze the last option, that it's use an Azure Firewall to provide outbound Internet routing for the VNet for the OCP cluster installation and usage.

Let's view a possible architecture of an Private OpenShift cluster with Azure Firewall:

[![](/images/azurefw0.png "azurefw0.png")]({{site.url}}/images/azurefw0.png)

Let's dig a bit deeper on this architecture, because have a lot of details that it's can be worth it to be explained more carefully.

We can divide this architecture in several parts:

* Azure Hub and Spoke Architecture
* Azure Firewall
* User Define Routing and Azure Firewall
* OpenShift Private Cluster outbound routing

### 3.1 Azure Hub & Spoke Architecture

The [Hub-Spoke topology in Azure](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke?tabs=cli) is a reference architecture provided by Microsoft:

[![](/images/azurefw0_1.png "azurefw0_1.png")]({{site.url}}/images/azurefw0_1.png)

NOTE: this diagram is extracted from the MSFT official documentation where this architecture is detailed.

* In this architecture the **hub virtual network** acts as a central point of connectivity to many spoke virtual networks. The hub can also be used as the connectivity point to your on-premises networks.

* The **Spoke virtual networks** are used to isolate workloads in their own virtual networks, managed separately from other spokes. Each workload might include multiple tiers, with multiple subnets connected through Azure load balancers.

* The **Virtual network peering** is also present allowing that two virtual networks can be connected using a peering connection. Peering connections are non-transitive, low latency connections between virtual networks. Once peered, the virtual networks exchange traffic by using the Azure backbone without the need for a router.

The Hub-Spoke topology in Azure is a recommended best practice architecture that have several benefits such as cost savings, overcoming limits and the most important workload isolation.

In our case, we've used the Hub-Spoke architecture because we can control all the outbound traffic (and also inbound), and filter in fine grain which URLs, IPs are white or blacklisted or even filtered by certain origins from our OpenShift cluster. 

That adds much more flexibility to the Private OpenShift cluster deployment, adding more security and control to the ingress/egress traffic that comes from/to our OCP cluster.

### 3.2 Azure Firewall and OCP

Azure Firewall is a cloud-native and intelligent network firewall security service that provides the best of breed threat protection for your cloud workloads running in Azure.

It's a fully stateful, firewall as a service with built-in high availability and unrestricted cloud scalability. It provides both east-west and north-south traffic inspection.

[![](/images/azurefw0_2.png "azurefw0_2.png")]({{site.url}}/images/azurefw0_2.png)

We will benefit from this Azure Firewall, that will act as a WAF, filtering and controlling all the traffic that comes from our cluster (or goes into).

Another feature that we can benefit is the URL filtering, blacklisting certain URLs/IP ranges, and also is capable even of act as IDPS and/or adding TLS inspection. Cool isn't?

The Azure Firewall lives in their own Subnet, defined as Hub in the Hub-Spoke architecture that is detailed in the step before.

This allows that ALL the traffic to/from our cluster (or clusters) will go through our VNET - Azure Firewall because of the Route Tables that will be explained below.

And on the other hand, imagine that instead of 1 cluster of OpenShift, you will deploy N OpenShift clusters that are private. With this architecture, you will have the possibility to deploy them as a Spokes in their own VNET / Subnets, having a proper architecture and control of your resources in Azure.

### 3.3 User Define Routing and Azure Firewall

Imagine that we deployed 

Outbound type of UDR requires there is a route for 0.0.0.0/0 and next hop destination of NVA (Network Virtual Appliance) in the route table. 

The route table already has a default 0.0.0.0/0 to Internet, without a Public IP to SNAT just adding this route will not provide you egress. AKS will validate that you don't create a 0.0.0.0/0 route pointing to the Internet but instead to NVA or gateway, etc. When using an outbound type of UDR, a load balancer public IP address for inbound requests is not created unless a service of type loadbalancer is configured. A public IP address for outbound requests is never created by AKS if an outbound type of UDR is set.

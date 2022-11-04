---
layout: post
title: Monitoring and Analysis of Network Traffic in K8s with Network Observability Operator  
date: 2022-08-22
type: post
published: true
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How can you monitor and make an deep dive analysis of your Network Traffic and Flows in your K8s/OCP clusters in a simple and easy way? How to have some dashboards in (nearly) real time around what's happening in your Kubernetes SDN networks? 

This is the third blog post of the series of Monitoring Network Flow Traffic in OpenShift/Kubernetes. 

Check the earlier posts in:

* [I - Monitoring and analysis of Network Flow Traffic in OpenShift (Part I)](https://rcarrata.com/openshift/traffic-flow-ovn)
* [II - Monitoring and analysis of Network Flow Traffic in OpenShift (Part II)](https://rcarrata.com/openshift/traffic-flow-ovn2/)

Let's dig in!

## 1. Overview

As we explained in the [first blog post of this series](https://rcarrata.com/openshift/traffic-flow-ovn), the network traffic analysis / monitoring within the OpenShift SDN is hard to perform. 

If you wanted to know the network flows of your traffic of your workloads in the SDN you needed to analyze node to node the Openflow (protocol designed to allow network controllers to determine the path) and the Flow tables from the OVS pods, you need to use utilities like ovs-ofctl to extract the flow entries in the flow table.

Putting all of this together in order to analyze and monitor our traffic was not easy, neither straightforward. We need to know which table is used for in each case and try to put together all pieces for debug and an

The previous posts talked about Network Flow Analysis configuring OpenShift to Tracking Network Flows using the Network Flow Collectors such as:

- NetFlow: feature that was introduced on Cisco routers that provides the ability to collect IP network traffic as it enters or exits an interface.

- sFlow: or sampled flow is an industry standard for packet export at Layer 2 of the OSI model. It provides a means for exporting truncated packets, together with interface counters for the purpose of network monitoring.

- IPFIX: is an IETF protocol that defines how IP flow information is to be formatted and transferred from an exporter to a collector.

The problem was that it was hard to extract information of all the logs collected, and process this info was hard and very time-consuming. For this reason we used [ELK Stack with ElastiFlow](https://rcarrata.github.io/openshift/traffic-flow-ovn2/) to provide some useful dashboards with all of this information showed in a nicer and comprehensive way.

But recently, another superb tool is developing to offer native alternatives of the options described before.

Entering [Network Observability Operator](https://github.com/netobserv/network-observability-operator)!

## 2. Network Observability Operator

NetObserv Operator is a Kubernetes / OpenShift operator for network observability. 

Deploys a monitoring pipeline to collect and enrich the Network Flows for our Kubernetes / OpenShift networking clusters.

These flows can be produced in two several ways, by the NetObserv eBPF agent, or by any device or CNI able to export flows in IPFIX format, for example OVN-Kubernetes.

The operator provides dashboards, metrics, and keeps flows accessible in a queryable log store, Grafana Loki. 

And a super cool feature is when is used in OpenShift, new dashboards are available in the Console!



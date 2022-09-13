---
layout: post
title: Securing CICD pipelines with StackRox / RHACS and Sigstore 
date: 2022-07-19
type: post
published: false
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How can we

## Overview

## Ingress Controller

The Ingress Operator implements the IngressController API and is the component responsible for enabling external access to ROSA cluster services alongside with the Custom Domain Operator.

The Ingress Operator makes it possible for external clients to access our service within the ROSA cluster by deploying and managing one or more HAProxy-based Ingress Controllers to handle routing.

You can use the Ingress Operator to route traffic by specifying OpenShift Container Platform Route and Kubernetes Ingress resources.

Let’s analyze the Networking a little bit deeper.

## Default ROSA IngressController  

By default in ROSA there is a IngressController that is generated during the ROSA cluster installation and manages and generates several components [3], including the Route53 domain *.apps.<cluster_name>.<cluster_id>.p1.openshiftapps.com, the LoadBalancer provided by the Kubernetes Service type LoadBalancer, and also the OpenShift Ingresses that are the HAProxies that, which will route the traffic to the pods within our cluster.

Let’s check this within a ROSA cluster.

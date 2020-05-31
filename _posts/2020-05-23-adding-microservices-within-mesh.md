---
layout: post
title: Including microservices in a Service Mesh
date: 2020-05-23
type: post
published: true
status: publish
categories:
- Istio
tags: []
author: rcarrata
comments: true
---

How we can include within Service Mesh our microservices deployed easily and automatically? How to control and
istiofying them? And what are the main components involved?

Let's Mesh in!!

This is the third blog post of the Service Mesh in Openshift series. Check the earlier post:
* [Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)

## Overview

In this blog post, we will istiofying our four microservices and we will explain terms like
autoinjection, and the main differences between RH Service mesh based in Maistra and with Istio
upstream.

## 0. Prerequisites

* Openshift 4.x cluster (tested in a 4.3+ cluster)
* Openshift Service Mesh Operators installed
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)

## 1. Enable sidecar autoinjection

Istio data plane is build as a sidecar container, which is living together with the application container running at the same pod.

So, in order to take advantage of all of Istio’s features, pods in the mesh must be running an Istio sidecar proxy.

Because of that, your pods will have 2 containers (app container + istio-proxy container), but we don't need to add this container definition manually.

### 1.1 Comparison between RH Service Mesh/Maistra and Istio Upstream

Enabling automatic injection for your deployments differs between the upstream Istio releases and
the Maistra releases.

The upstream sidecar injector injects all deployments within labeled projects
whereas the Maistra version relies on presence of the sidecar.istio.io/inject annotation and the
project being listed in the ServiceMeshMemberRoll.

### 1.2 RH Service Mesh Automatic Injection

As we defined before, to enable the sidecar autoinjection in our pods, we will have to annotate them with:

```
sidecar.istio.io/inject: true
```

The annotation needs to be reflected in the Pod, but instead we're to define this annotation in the
controller that in this case is the DeploymentConfig (or Deployments, etc).

It's important to set properly the annotation above the correct structure:

```
spec.template.metadata.annotations
```

If it's defined into the metadata.annotations have not any effect and the autoinjection its not
correctly applied.

Remember that this is one of the biggest difference between Istio upstream, where you annotate the
namespace istead.

Again focusing in our specific case, we'll patch our current DeploymentConfig of our microservices
deployed in the previous lab, in order to enable autoinjection:

```
oc patch dc/customer -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/preference -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/recommendation -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/partner -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS
```

Check how the rolling process has been triggered, how new pods are created and finally, the current
number of containers per pod:

```
$ oc get pod
NAME                      READY   STATUS      RESTARTS   AGE
customer-1-deploy         0/1     Completed   0          43m
customer-2-4gkzf          2/2     Running     0          42m
customer-2-deploy         0/1     Completed   0          42m
partner-1-deploy          0/1     Completed   0          40m
partner-2-4q8zl           2/2     Running     0          39m
partner-2-deploy          0/1     Completed   0          39m
preference-1-deploy       0/1     Completed   0          40m
preference-2-deploy       0/1     Completed   0          39m
preference-2-nxh2v        2/2     Running     0          39m
recommendation-1-deploy   0/1     Completed   0          40m
recommendation-2-cj25h    2/2     Running     0          39m
recommendation-2-deploy   0/1     Completed   0          39m
```

For more information check out the [Istio Autoinject Guide](https://istio.io/docs/setup/additional-setup/sidecar-injection/#injection) and the [Maistra Autoinject Guide](https://maistra.io/docs/installation/automatic-injection/)

Check out the part four of this blog series in [Ingress Routing in Service Mesh](https://rcarrata.com/istio/ingress-routing-service-mesh/)

Happy Service Meshing!



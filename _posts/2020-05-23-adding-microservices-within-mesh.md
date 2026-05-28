---
layout: single
title: Including microservices in OpenShift Service Mesh
date: 2020-05-23
type: post
published: true
status: publish
categories:
- Istio
tags: ["Kubernetes", "security", "Administration", "OpenShift", "Networking", "Istio"]
author: rcarrata
comments: true
---

How can we include our microservices within Service Mesh easily and automatically? How to control and
istiofy them? And what are the main components involved?

Let's Mesh in!!

This is the third blog post of the Service Mesh in OpenShift series. Check the earlier post:
* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [II - Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)

## Overview

In this blog post, we will istiofy our four microservices and we will explain terms like
autoinjection, and the main differences between RH Service Mesh based on Maistra and Istio
upstream.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 0. Prerequisites

* OpenShift 4.x cluster (tested in a 4.3+ cluster)
* OpenShift Service Mesh Operators installed (v1.1.1 in these blog posts)
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)

## 1. Enable sidecar autoinjection

The Istio data plane is built as a sidecar container, which lives together with the application container running in the same pod.

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

The annotation needs to be reflected in the Pod, but instead we need to define this annotation in the
controller, which in this case is the DeploymentConfig (or Deployment, etc.).

It is important to properly set the annotation in the correct structure:

```
spec.template.metadata.annotations
```

If it is defined in the metadata.annotations, it will not have any effect and the autoinjection is not
correctly applied.

Remember that this is one of the biggest differences from Istio upstream, where you annotate the
namespace instead.

Again focusing on our specific case, we'll patch our current DeploymentConfig of our microservices
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

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy Service Meshing!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
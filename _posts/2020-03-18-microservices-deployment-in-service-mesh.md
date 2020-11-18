---
layout: post
title: Microservices Deployment in Service Mesh
date: 2020-03-18
type: post
published: true
status: publish
categories:
- Istio
tags: []
author: rcarrata
comments: true
---

How we can deploy microservices within Service Mesh easily and automatically? How to control and
istiofying them? And what are the main components involved?

Let's Mesh in!!

This is the second blog post of the Service Mesh in Openshift series. Check the earlier post:
* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)

## Overview

In this blog post, we will deploy four microservices that will help us for show the capabilities of Istio.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 0. Prerequisites

* Openshift 4.x cluster (tested in a 4.3+ cluster)
* Openshift Service Mesh Operators installed (v1.1.1 in these blog posts)
* Service Mesh Control Plane deployed
* Free time to have fun with Service Mesh!

Export the useful variables that will be use for customize the Namespace where our microservices
will be installed:

```
export OCP_NS=istio-tutorial
```

## 1. Deploy the microservices of our application

The microservices used in this tutorial are composed of customer / partner, preference and
recommendation. The workflow between them is as following:

```
( customer | partner ) ⇒ preference ⇒ recommendation
```

So as we can see, if we request the customer/partner microservice route, internally these apps will
call the services of preference, and therefore the will be recommendation finally.

* Deploy the customer v1 microservice using oc new-app, that will deploy a service, a route, a
  deploymentconfig and the pods of customer:

```
oc new-app -l app=customer,version=v1 --name=customer --docker-image=quay.io/rcarrata/customer:quarkus -e VERSION=v1 -e  JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -n $OCP_NS

oc expose svc customer -n $OCP_NS
```

Pay attention in the labels of this microservices: app=customer and version=v1, that are very
important for our purposes.

* Deploy the partner v1 app:

```
oc new-app -l app=partner,version=v1 --name=partner --docker-image=quay.io/rcarrata/partner:sb -e JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -n $OCP_NS

oc expose svc partner -n $OCP_NS
```

As you can see, the partner and the customer will expose a route of Openshift. Nothing wierd until
here right?

* Deploy the preference v1 app:

```
oc new-app -l app=preference,version=v1 --name=preference --docker-image=quay.io/rcarrata/preference:quarkus -e JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true'  -n $OCP_NS
```

This microservice app, don't need any route because only need customer or partner to be reached
externally, and when the request reach this two microservices internally and using the svc of
Openshift/Kubernetes will be used to communicate through preference and recommendation micros.

* Deploy the recommendation v1 app:

```
oc new-app -l app=recommendation,version=v1 --name=recommendation --docker-image=quay.io/dsanchor/recommendation:vertx -e JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -e VERSION=v1 -n $OCP_NS
```

## 2. Test our microservices (outside of the Mesh)

Let's test it our microservices routes:

```
$ oc get route | egrep -i 'customer|partner'
customer   customer-istio-tutorial.apps.ocp4.rglab.com          customer   8080-tcp None
partner    partner-istio-tutorial.apps.ocp4.rglab.com           partner    8080-tcp None
```

As we can see the customer and partner have the same "behaviour":

```
$ curl partner-meshtutorial.apps.ocp4.rglab.com
partner => preference => recommendation v1 from 'recommendation-1-9d977': 7
$ curl customer-meshtutorial.apps.ocp4.rglab.com
customer v1 => preference => recommendation v1 from 'recommendation-1-9d977': 8
```

## 3. Including our microservices in our Service Mesh

Until now, this deployment of this apps is a regular deployment with all of the "traditional"
components of Service Mesh, with Services, Routes, and DeploymentConfigs among others.

But for enable the Service Mesh capabilities, we need to include these microservices into our
Service Mesh. How can we accomplished that? With sidecar autoinjection.

But first of all, two important things that need to discuss, sidecar containers and istio data
plane.

Istio data plane is built as a sidecar container, which is deployed together with our applications
running in the same pod (yes, same pod).

So in overview, in order to take advantage of all of Istio’s features, pods in the mesh must be
running an Istio sidecar proxy.

### 3.1 Differences between Istio upstream and RH Service Mesh / Maistra

Enabling automatic injection for your deployments differs between the upstream Istio releases and
the Maistra releases. The upstream sidecar injector injects all deployments within labeled projects
whereas the RH Service Mesh / Maistra version relies on presence of the sidecar.istio.io/inject annotation and the
project being listed in the ServiceMeshMemberRoll.

More information could be found in:

* [Maistra Automatic Injection](https://maistra.io/docs/installation/automatic-injection/)
* [Istio Upstream Automatic Injection](https://istio.io/docs/setup/additional-setup/sidecar-injection/)

### 3.2 Enabling the sidecar autoinjector

To enable the sidecar autoinjector in our Pod, we will have to annotate them with the
following annotation:

```
sidecar.istio.io/inject: true
```

But usually, we don't annotate the Pods directly, instead we will annotate the controller such as
DeploymentConfigs or Deployments (among others).

The annotation need to go under the "spec.template.metadata.annotations". Remember that you need to
be strict on this, is not valid to put it on the metadata.annotations level, where usually goes the
annotations.

As we discussed in the section before, one of the differences with Istio upstream is that here we
annotate the deploymentconfigs, and not annotate the .

For enable the autoinjection of our DeploymentConfigs run the following patch:

```
oc patch dc/customer -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/preference -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/recommendation -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS

oc patch dc/partner -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS
```

Then a rolling process will triggered, and a new pods will be created finally when the current of
containers per pod (2/2):

```
$ oc get pod | grep -i Running
NAME                      READY   STATUS      RESTARTS   AGE
customer-2-4gkzf          2/2     Running     0          42s
partner-2-4q8zl           2/2     Running     0          39s
preference-2-nxh2v        2/2     Running     0          39s
recommendation-2-cj25h    2/2     Running     0          39s
```

As we can see we have two containers running in the same pod: istio-proxy and our container app.

### 3.3 Controlling the namespaces allowed to be in our Mesh

Last but not least an interesting topic is how to control the namespaces what are allowed to be including apps in
the mesh. This is controlled by the ServiceMeshMemberRoll.

The ServiceMeshMemberRoll resource configures which projects belong to a control plane. Only
projects listed in the ServiceMeshMemberRoll will be affected by the control plane. Any number of
projects can be added, but a project may not exist in more than one control plane.

This resource must be created in the same project as the ServiceMeshControlPlane resource.

```
$ oc get ServiceMeshMemberRoll -n istio-system -o yaml | grep spec -A3
 spec:
   members:
   - bookinfo
   - istio-tutorial
```

For more information refer to the [Configuring Members](https://maistra.io/docs/installation/configuring-members/) in Maistra documentation.

And that's it! Hope that helps!

Check out the part three of this blog series in [Including microservices in ServiceMesh](https://rcarrata.com/istio/adding-microservices-within-mesh/)

Happy ServiceMeshing!


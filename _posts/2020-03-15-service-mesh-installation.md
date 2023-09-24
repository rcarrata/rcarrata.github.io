---
layout: single
title: Service Mesh Installation
date: 2020-03-15
type: post
published: true
status: publish
categories:
- Istio
tags: ["Kubernetes", "security", "Administration", "OpenShift", "Networking", "Istio"]
author: rcarrata
comments: true
---

How to install Service Mesh easy, straight forward and with a basic but fully operational installation on top of OpenShift 4? And how are the advantages and caveats of each configuration parameter?

I been playing with Service Mesh more than 1 year in this moments. I tweaked, broke it a lot of times, tried with
different configurations and after a bit off playing with other things, I wanted to write some blog
post with all the things that I discovered.

This is a blog post series (I hope :D), that involves all of the configurations, tools, components,
pros/cons about this wonderful world of the Service Mesh.

All of the blog post will be about Service Mesh on top of OpenShift, but as you know you can deploy
very easily the exact examples in Kubernetes.

So, let's start!

## Overview

Installing the Service Mesh involves :

* Installing Elasticsearch, Jaeger, Kiali

* Installing the Service Mesh Operator

* Creating and managing a ServiceMeshControlPlane resource to deploy the Service Mesh control plane

* Creating a ServiceMeshMemberRoll resource to specify the namespaces associated with the Service Mesh.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 1. Installing the Service Mesh Operators from OperatorHub

The Red Hat OpenShift Service Mesh Operator has dependencies Elasticsearch, Jaeger and Kiali operators.

Check the [Service Mesh Installation in OpenShift4](https://docs.openshift.com/container-platform/4.3/service_mesh/service_mesh_install/installing-ossm.html#ossm-operatorhub-install_installing-ossm) of the official documentation of OpenShift and install

### 1.1 Installing the Elasticsearch Operator

* In the OperatorHub catalog of your OCP Web Console, type Elasticsearch into the filter box to locate the Elasticsearch Operator.

[![](/images/istio-install1.png "ElasticSearch Operator")]({{site.url}}/images/istio-install1.png)

* Check the Elasticsearch operators installation

```
$ oc get subscription -n openshift-operators | grep "^elastic"
elasticsearch-operator         elasticsearch-operator         redhat-operators      4.3
```

```
$ oc get clusterserviceversion -n openshift-operators | grep "^elastic"
elasticsearch-operator.4.3.14-202004200457   Elasticsearch Operator           4.3.14-202004200457                                         Succeeded
```

### 1.2 Installing Jaeger Operator

* In the OperatorHub catalog of your OCP Web Console, type Jaeger into the filter box to locate the Elasticsearch Operator.

[![](/images/istio-install2.png "Jaeger Operator")]({{site.url}}/images/istio-install2.png)

* Check the Jaeger operators installation

```
$ oc get subscription -n openshift-operators | grep "^jaeger"
jaeger-product                 jaeger-product                 redhat-operators      stable
```

```
$ oc get clusterserviceversion -n openshift-operators | grep "^elastic"
jaeger-operator.v1.17.2                      Red Hat OpenShift Jaeger         1.17.2                                                      Succeeded
```

### 1.3 Installing the Kiali Operator

* In the OperatorHub catalog of your OCP Web Console, type Kiali Operator into the filter
box to locate the Elasticsearch Operator.

[![](/images/istio-install3.png "Kiali Operator")]({{site.url}}/images/istio-install3.png)

* Check the Kiali operators installation

### 1.4 Installing Service Mesh Operators

* In the OperatorHub catalog of your OCP Web Console, type ServiceMesh Operator into the filter
box to locate the Elasticsearch Operator.

[![](/images/istio-install4.png "Service Mesh Operator")]({{site.url}}/images/istio-install4.png)

* Check the Service mesh operators installation

```
$ oc get subscription -n openshift-operators | grep "^servicemesh"
servicemeshoperator            servicemeshoperator            redhat-operators      stable
```

```
$ oc get clusterserviceversion -n openshift-operators | grep "^servicemesh"
servicemeshoperator.v1.1.1                   Red Hat OpenShift Service Mesh   1.1.1                 servicemeshoperator.v1.1.0            Succeeded
```

### 1.5 Check that all the mesh operators are installed

Finally, check that the operators involved in the Service Mesh installation are installed properly:

```
$ oc get subscription -n openshift-operators
NAMESPACE             NAME                           PACKAGE                        SOURCE                CHANNEL
openshift-operators   elasticsearch-operator         elasticsearch-operator         redhat-operators      4.3
openshift-operators   jaeger-product                 jaeger-product                 redhat-operators      stable
openshift-operators   kiali-ossm                     kiali-ossm                     redhat-operators      stable
openshift-operators   openshift-pipelines-operator   openshift-pipelines-operator   community-operators   dev-preview
```

On the other hand, you can check the installation with the ClusterServiceVersion custom resource:

```
$ oc get ClusterServiceVersion
NAME                                         DISPLAY                          VERSION               REPLACES                              PHASE
elasticsearch-operator.4.3.14-202004200457   Elasticsearch Operator           4.3.14-202004200457                                         Succeeded
jaeger-operator.v1.17.2                      Red Hat OpenShift Jaeger         1.17.2                                                      Succeeded
kiali-operator.v1.12.11                      Kiali Operator                   1.12.11               kiali-operator.v1.12.7                Succeeded
servicemeshoperator.v1.1.1                   Red Hat OpenShift Service Mesh   1.1.1                 servicemeshoperator.v1.1.0            Succeeded
```

## 2. Installing ServiceMesh Control Plane and Service Mesh Member Role

The previously installed Service Mesh operator watches for a ServiceMeshControlPlane resource in all namespaces. Based on the configurations defined in that ServiceMeshControlPlane, the operator creates the Service Mesh control plane.

### 2.1 Create istio-system namespace

Create a namespace called istio-system where the Service Mesh control plane will be installed.

```
$ cat istio-files/mesh-install/servicemesh-namespace.yml
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: istio-system
  spec:
    finalizers:
      - kubernetes
$ oc apply -f istio-files/mesh-install/servicemesh-namespace.yml
```

### 2.2 Deploy Service Mesh Control Plane

Let's dig in in the installation of a Service Mesh Control Plane, once we have the operators in
place, and the namespace for the installation of the mesh deployed:

The file of the basic-istio-install.yaml contains the description of a basic mesh installation with
the configuration of all of the components involved (a bit tweaked):

```
$ cat istio-files/mesh-install/basic-istio-install.yml
apiVersion: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: basic-install
  namespace: istio-system-$NAMESPACE
spec:
  istio:
    global:
      controlPlaneSecurityEnabled: true
      disablePolicyChecks: false
      mtls:
        enabled: true
      proxy:
        accessLogFile: /dev/stdout
    sidecarInjectorWebhook:
      rewriteAppHTTPProbe: true
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
      istio-ingressgateway:
        autoscaleEnabled: false
    mixer:
      policy:
        autoscaleEnabled: false
      telemetry:
        autoscaleEnabled: false
    pilot:
      autoscaleEnabled: false
      traceSampling: 100
    kiali:
      enabled: true
    grafana:
      enabled: true
    tracing:
      enabled: true
      jaeger:
        template: all-in-one
---
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system-$NAMESPACE
spec:
  members:
  - $NAMESPACE
```

NOTES:

* Mutual TLS is enabled by setting mtls to true.
* Kiali and grafana are enabled
* Mixer (policy and telemetry) have autoscaleEnabled disabled
* Gateway (ingress and egress) have autoscaleEnabled disabled
* [RewriteAppHTTProbe](https://istio.io/docs/ops/configuration/mesh/app-health-check/) set to true
* Jaeger is enabled with the [template of all-in-one](https://github.com/jaegertracing/jaeger-openshift/blob/master/all-in-one/jaeger-all-in-one-template.yml)


Apply the basic Service Mesh Control Plane:

```
$ cat istio-files/mesh-install/basic-istio-install.yml | NAMESPACE=$OCP_NS envsubst | oc apply -f -
servicemeshcontrolplane.maistra.io/basic-install created
servicemeshmemberroll.maistra.io/default created
```

After a bit, check that the control plane is installed properly:

```
$ oc get pod -n istio-system
NAME                                      READY   STATUS    RESTARTS   AGE
grafana-67c58f9f9-2czv8                   2/2     Running   0          3m33s
istio-citadel-6784798885-2hmwt            1/1     Running   0          5m21s
istio-egressgateway-659bb7c7db-xtmj9      1/1     Running   0          4m8s
istio-galley-5988bb6f9c-88tzf             1/1     Running   0          4m24s
istio-ingressgateway-569c9555db-ptpq6     1/1     Running   0          4m8s
istio-pilot-5fff54b8cc-dt54j              2/2     Running   0          4m8s
istio-policy-7f4dbcc979-cjfhh             2/2     Running   0          4m13s
istio-sidecar-injector-866fccd4d9-24t9b   1/1     Running   0          3m49s
istio-telemetry-7dc77c4d8b-9r7p9          2/2     Running   0          4m13s
jaeger-6cfd8f88bf-5ffmc                   2/2     Running   0          4m24s
kiali-599499888b-cxz9w                    1/1     Running   0          2m33s
prometheus-6886c768dc-886s9               2/2     Running   0          5m10s
```

The Service Mesh operator has installed a control plane configured for multitenancy. This installation reduces the scope of the control plane to only those projects/namespaces listed in a ServiceMeshMemberRoll.

```
$ oc get smmr -o yaml
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: istio-system
spec:
  members:
  - istio-tutorial
```

NOTE: if you want to add more namespaces inside of the mesh, add the namespaces in the members list
inside of the smmr.

## Links

Check the [Service Mesh Installation in OpenShift4](https://docs.openshift.com/container-platform/4.3/service_mesh/service_mesh_install/installing-ossm.html#ossm-operatorhub-install_installing-ossm) of the official documentation of OpenShift for more information.

Check out the part two of this blog series in [Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy ServiceMeshing!!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
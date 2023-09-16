---
layout: single
title: Analysis of Operator Lifecycle Manager in OpenShift 4
date: 2020-06-16
type: post
published: true
status: publish
categories:
- OpenShift
- Operators
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How can I use the Operators in my OpenShift4 cluster? And how are updated and managed by the
Operator Lifecycle Manager? What is the components of their architecture?

Let's dig in!

## 1. Overview

In OpenShift 4 the Operators are key for main operations, deployments and upgrades within our
cluster.

The Operator Lifecycle Manager (OLM in short name), helps users install, update, and manage the
lifecycle of all Operators and their associated services running across their clusters.

This OLM is part of the Operator Framework, defined by CoreOS and widely spread into the Kubernetes
Community among these years.

The Operator Framework is an open source toolkit designed to manage Kubernetes native apps (also
called Operators) in a automated, time-effective and scalable way.

## 2. Operator Lifecycle Manager Architecture

The Operator Lifecycle Manager is composed of two Operators: the OLM Operator and the Catalog Operator.

* ClusterServiceVersion (OLM): Application metadata: name, version, icon, required resources, installation, etc.
* InstallPlan (Catalog): Calculated list of resources to be created in order to automatically install or upgrade a CSV.
* CatalogSource (Catalog): A repository of CSVs, CRDs, and packages that define an application.
* Subscription (Catalog): Used to keep CSVs up to date by tracking a channel in a package.
* OperatorGroup (OLM): Used to group multiple namespaces and prepare them for use by an Operator.

### 2.1 OLM Operator

The OLM Operator is responsible for deploying applications defined by CSV resources after the
required resources specified in the CSV are present in the cluster.

```
$ oc get pod -n openshift-operator-lifecycle-manager | grep olm-operator
olm-operator-7c89775499-hxzqv       1/1     Running   0          45h
```

### 2.2 Catalog Operator

The Catalog Operator is responsible for resolving and installing CSVs and the required resources
they specify. It is also responsible for watching CatalogSources for updates to packages in channels
and upgrading them (optionally automatically) to the latest available versions.

```
$ oc get pod -n openshift-operator-lifecycle-manager | grep olm-operator
NAME                                READY   STATUS    RESTARTS   AGE
catalog-operator-74694fb4d4-g4fsk   1/1     Running   0          45h
```

A user who wishes to track a package in a channel creates a Subscription resource configuring the
desired package, channel, and the CatalogSource from which to pull updates. When updates are found,
an appropriate InstallPlan is written into the namespace on behalf of the user.

```
$ oc get installplan -n openshift-operators | grep elasticsearch
install-b4628   elasticsearch-operator.4.3.20-202005121847   Automatic   true
install-fngg7   elasticsearch-operator.4.3.14-202004200457   Automatic   true
install-glfb4   elasticsearch-operator.4.3.22-202005201238   Automatic   true
install-hdrdv   elasticsearch-operator.4.3.19-202005041055   Automatic   true
install-k8jbq   elasticsearch-operator.4.2.29-202004140532   Automatic   true
install-l927k   elasticsearch-operator.4.4.0-202004261927    Automatic   true
install-px7tr   elasticsearch-operator.4.3.23-202005270305   Automatic   true
```

### 2.3 Resources created by OLM and Catalog Operators

The resources created by the OLM and Catalog Operators are the following:

* OLM: Deployments, ServiceAccounts, (Cluster)Roles and (Cluster)RoleBindings
* Catalog: Custom Resource Definitions (CRDs), ClusterServiceVersions (CSVs)

## 3. Operator Lifecycle Manager ecosystem

So, now that we already explained the components of the OLM, we need to describe the ecosystem in
order to deploy, upgrade and maintain our operators within the OLM.

In the Operator Lifecycle Manager (OLM) ecosystem, the following resources are used to resolve Operator installations and upgrades:

* ClusterServiceVersion (CSV)
* CatalogSource
* Subscription

### Cluster Service Version (CSV)

A ClusterServiceVersion (CSV) is a YAML manifest created from Operator metadata that assists the
Operator Lifecycle Manager (OLM) in running the Operator in a cluster.

```
$ oc get csv
NAME                                         DISPLAY                          VERSION
REPLACES                                     PHASE
elasticsearch-operator.4.3.23-202005270305   Elasticsearch Operator           4.3.23-202005270305
elasticsearch-operator.4.3.22-202005201238   Succeeded
jaeger-operator.v1.17.2                      Red Hat OpenShift Jaeger         1.17.2
Succeeded
kiali-operator.v1.12.12                      Kiali Operator                   1.12.12
kiali-operator.v1.12.11                      Succeeded
openshift-pipelines-operator.v0.10.7         OpenShift Pipelines Operator     0.10.7
openshift-pipelines-operator.v0.8.2          Succeeded
packageserver                                Package Server                   0.14.2
Succeeded
servicemeshoperator.v1.1.2.3                 Red Hat OpenShift Service Mesh   1.1.2+3
servicemeshoperator.v1.1.2.2                 Succeeded
```

### Catalog Sources

Operator metadata, defined in CSVs, can be stored in a collection called a CatalogSource.
OLM uses CatalogSources, which use the Operator Registry API, to query for available Operators as well as upgrades for installed Operators.

```
$ oc get CatalogSources -n openshift-marketplace
NAME                  DISPLAY               TYPE   PUBLISHER   AGE
certified-operators   Certified Operators   grpc   Red Hat     76d
community-operators   Community Operators   grpc   Red Hat     76d
redhat-marketplace    Red Hat Marketplace   grpc   Red Hat     18d
redhat-operators      Red Hat Operators     grpc   Red Hat     76d
```

Within a CatalogSource, Operators are organized into packages and streams of updates called channels, which should be a familiar update pattern from OpenShift Container Platform or other software on a continuous release cycle like web browsers.

```
$ oc get CatalogSources -n openshift-marketplace community-operators -o yaml --export
Flag --export has been deprecated, This flag is deprecated and will be removed in future.
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  generation: 1
  labels:
    olm-visibility: hidden
    openshift-marketplace: "true"
    opsrc-datastore: "true"
    opsrc-owner-name: community-operators
    opsrc-owner-namespace: openshift-marketplace
    opsrc-provider: community
  name: community-operators
  selfLink: /apis/operators.coreos.com/v1alpha1/namespaces/openshift-marketplace/catalogsources/community-operators
spec:
  address: community-operators.openshift-marketplace.svc:50051
  displayName: Community Operators
  icon:
    base64data: ""
    mediatype: ""
  publisher: Red Hat
  sourceType: grpc
```

### Subscriptions

A user indicates a particular package and channel in a particular CatalogSource in a Subscription.

```
$ oc get subscription -n openshift-operators
NAME                           PACKAGE                        SOURCE                CHANNEL
elasticsearch-operator         elasticsearch-operator         redhat-operators      4.3
jaeger-product                 jaeger-product                 redhat-operators      stable
kiali-ossm                     kiali-ossm                     redhat-operators      stable
openshift-pipelines-operator   openshift-pipelines-operator   community-operators   dev-preview
servicemeshoperator            servicemeshoperator            redhat-operators      stable
```

If a Subscription is made to a package that has not yet been installed in the namespace, the latest Operator for that package is installed.

For example the Kiali subscription:

```
$ oc get subscription -n openshift-operators kiali-ossm -o yaml --export
Flag --export has been deprecated, This flag is deprecated and will be removed in future.
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  generation: 1
  name: kiali-ossm
  selfLink: /apis/operators.coreos.com/v1alpha1/namespaces/openshift-operators/subscriptions/kiali-ossm
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kiali-ossm
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: kiali-operator.v1.12.11
```

And that's its for this blog post!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
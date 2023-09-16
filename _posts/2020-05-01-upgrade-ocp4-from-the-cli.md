---
layout: single
title: Upgrade OpenShift 4 clusters with the CLI
date: 2020-05-01
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How to perform an OCP4 upgrade from the CLI safe, securely and fully CLI executed?

Let's take a look!

## Overview

The OpenShift Container Platform update service is the hosted service that provides over-the-air
updates to both OpenShift Container Platform and Red Hat Enterprise Linux CoreOS (RHCOS). It
provides a graph, or diagram that contain vertices and the edges that connect them, of component
Operators.

The Cluster Version Operator (CVO) in your cluster checks with the OpenShift Container Platform
update service to see the valid updates and update paths based on current component versions and
information in the graph. When you request an update, the OpenShift Container Platform CVO uses the
release image for that update to upgrade your cluster. The release artifacts are hosted in Quay as
container images.

Upgrade channels are tied to a minor version of OpenShift Container Platform. For instance,
OpenShift Container Platform 4.2 upgrade channels will never include an upgrade to a 4.3 release.
This strategy ensures that administrators explicitly decide to upgrade to the next minor version of
OpenShift Container Platform.

OpenShift Container Platform 4.2 offers the following upgrade channels:

* Stable: This channel contains releases as soon as their errata are published, releases are added
to the stable channel after a delay of several hours to a day.

* Fast: This channel is updated with new versions as soon as Red Hat declares the given version as
a general availability release.

* Candidate: Release candidates contain all the features of the product but are not supported. Use
release candidate versions to test feature acceptance and assist in qualifying the next version
of OpenShift Container Platform.

## Planning the upgrade - Graph Upgrade

For take a view in the upgrade plan before to start to upgrade process, there is a useful tool to
visualize this path in a SVG format:


```
curl https://raw.githubusercontent.com/pamoedom/ocp4upc/master/ocp4upc.sh > ocp4-upgrade-checker.sh
chmod +x ocp4-upgrade-checker.sh
```

Generate the graph for your current OpenShift 4 version.

```
./ocp4-upgrade-checker.sh 4.2.2
[INFO] Checking prerequisites... [OK]
[INFO] Checking if '4.2.2' (amd64) is a valid release... [OK]
sed: -e expression #1, char 8: unterminated `s' command
sed: -e expression #1, char 8: unterminated `s' command
[INFO] Result exported as 'stable-4.3.svg'
[INFO] Result exported as 'fast-4.3.svg'
```

Open the graph with an SVG visualizer (e.g. firefox).

```
firefox stable-4.3.svg
```

The edges in the graph show which versions you can safely update to, and the vertices are update
payloads that specify the intended state of the managed cluster components. For the example graph
before, the upgrade will be performed to 4.2.29 and then to 4.3.13.

* Minor target version: 4.2.29
* Major target version: 4.3.13

## Check your cluster health

Before to start your upgrade, verify that everything is ok in your cluster:

First of all, check that all nodes are in a Ready status.

```
$ oc get nodes
NAME                     STATUS   ROLES          AGE   VERSION
master0.ocp4.rglab.com   Ready    master         31d   v1.14.6.+7e13ab9a7
master1.ocp4.rglab.com   Ready    master         31d   v1.14.6.+7e13ab9a7
master2.ocp4.rglab.com   Ready    master         31d   v1.14.6.+7e13ab9a7
worker0.ocp4.rglab.com   Ready    infra,worker   31d   v1.14.6.+7e13ab9a7
worker1.ocp4.rglab.com   Ready    infra,worker   31d   v1.14.6.+7e13ab9a7
worker2.ocp4.rglab.com   Ready    infra,worker   31d   v1.14.6.+7e13ab9a7
worker3.ocp4.rglab.com   Ready    worker         9d    v1.14.6.+7e13ab9a7
worker4.ocp4.rglab.com   Ready    worker         9d    v1.14.6.+7e13ab9a7
worker5.ocp4.rglab.com   Ready    worker         9d    v1.14.6.+7e13ab9a7
worker6.ocp4.rglab.com   Ready    worker         9d    v1.14.6.+7e13ab9a7
worker7.ocp4.rglab.com   Ready    worker         9d    v1.14.6.+7e13ab9a7
```

Verify the current status version is available and there is no upgrade in progress.

```
$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.2.2    True        False         10d     Cluster version is 4.3.12
```

Ensure the cluster is healthy and the upgrade can be performed checking that all operators are
available, none of them should be degraded.

```
$ oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.2.2    True        False         False      31d
cloud-credential                           4.2.2    True        False         False      31d
cluster-autoscaler                         4.2.2    True        False         False      31d
console                                    4.2.2    True        False         False      10d
dns                                        4.2.2    True        False         False      31d
image-registry                             4.2.2    True        False         False      10d
ingress                                    4.2.2    True        False         False      31d
insights                                   4.2.2    True        False         False      31d
kube-apiserver                             4.2.2    True        False         False      31d
kube-controller-manager                    4.2.2    True        False         False      31d
kube-scheduler                             4.2.2    True        False         False      31d
machine-api                                4.2.2    True        False         False      31d
machine-config                             4.2.2    True        False         False      10d
marketplace                                4.2.2    True        False         False      10d
monitoring                                 4.2.2    True        False         False      9h
network                                    4.2.2    True        False         False      31d
node-tuning                                4.2.2    True        False         False      10d
openshift-apiserver                        4.2.2    True        False         False      9h
openshift-controller-manager               4.2.2    True        False         False      31d
openshift-samples                          4.2.2    True        False         False      10d
operator-lifecycle-manager                 4.2.2    True        False         False      31d
operator-lifecycle-manager-catalog         4.2.2    True        False         False      31d
operator-lifecycle-manager-packageserver   4.2.2    True        False         False      10d
service-ca                                 4.2.2    True        False         False      31d
service-catalog-apiserver                  4.2.2    True        False         False      31d
service-catalog-controller-manager         4.2.2    True        False         False      31d
storage                                    4.2.2    True        False         False      10d
```

Check the latest updates available to the cluster.

```
$ oc adm upgrade
Cluster version is 4.2.2

Updates:

VERSION IMAGE
4.2.4
quay.io/openshift-release-dev/ocp-release@sha256:cebce35c054f1fb066a4dc0a518064945087ac1f3637fe23d2ee2b0c433d6ba8
4.2.7
quay.io/openshift-release-dev/ocp-release@sha256:bac62983757570b9b8f8bc84c740782984a255c16372b3e30cfc8b52c0a187b9
4.2.8
quay.io/openshift-release-dev/ocp-release@sha256:4bf307b98beba4d42da3316464013eac120c6e5a398646863ef92b0e2c621230
4.2.9
quay.io/openshift-release-dev/ocp-release@sha256:f28cbabd1227352fe704a00df796a4511880174042dece96233036a10ac61639
4.2.10
quay.io/openshift-release-dev/ocp-release@sha256:dc2e38fb00085d6b7f722475f8b7b758a0cb3a02ba42d9acf8a8298a6d510d9c
4.2.12
quay.io/openshift-release-dev/ocp-release@sha256:77ade34c373062c6a6c869e0e56ef93b2faaa373adadaac1430b29484a24d843
4.2.13
quay.io/openshift-release-dev/ocp-release@sha256:782b41750f3284f3c8ee2c1f8cb896896da074e362cf8a472846356d1617752d
4.2.14
quay.io/openshift-release-dev/ocp-release@sha256:3fabe939da31f9a31f509251b9f73d321e367aba2d09ff392c2f452f6433a95a
```

## Upgrade to latest version in a channel

If the minor target version (4.2.29) is not available, upgrade to the latest available version until it is available.

```
$ oc adm upgrade --to-latest
Updating to latest version 4.2.14
```

Monitor the upgrade each 10 seconds until it is 100% complete.

```
$ watch -n 10 "oc adm upgrade && oc get co && oc get nodes -o wide"
Every 10.0s: oc adm upgrade && oc get co && oc get nodes -o wide

 info: An upgrade is in progress. Working towards 4.2.14: 13% complete
 ...
```

Once the upgrade is completed, check again if the latest updates available contain the minor target version (4.2.29).

```
$ oc adm upgrade
Cluster version is 4.2.14

Updates:

VERSION IMAGE
4.2.16
quay.io/openshift-release-dev/ocp-release@sha256:e5a6e348721c38a78d9299284fbb5c60fb340135a86b674b038500bf190ad514
4.2.18
quay.io/openshift-release-dev/ocp-release@sha256:283a1625e18e0b6d7f354b1b022a0aeaab5598f2144ec484faf89e1ecb5c7498
4.2.19
quay.io/openshift-release-dev/ocp-release@sha256:b51a0c316bb0c11686e6b038ec7c9f7ff96763f47a53c3443ac82e8c054bc035
4.2.20
quay.io/openshift-release-dev/ocp-release@sha256:bd8aa8e0ce08002d4f8e73d6a2f9de5ae535a6a961ff6b8fdf2c52e4a14cc787
4.2.21
quay.io/openshift-release-dev/ocp-release@sha256:6c57b48ec03382d9d63c529d4665d133969573966400515777f36dd592ad834a
4.2.25
quay.io/openshift-release-dev/ocp-release@sha256:dfbe59ca5dcc017475a0e1c703f51750c1bde63f12c725fbe4b7a599e36eb725
4.2.26
quay.io/openshift-release-dev/ocp-release@sha256:af0a6384e787c820279a0954c804d9c27b13458df2de366b3cd5ec3a7cdaa4b2
```

Because it does not contain the minor target version, the upgrade must be done to a version that
allows an upgrade to the minor target version according to the upgrade plan graph (e.g. 4.2.26).

```
$ oc adm upgrade --to="4.2.26"
Updating to 4.2.26
```

Finally, upgrade the cluster to the minor target version.

```
$ oc adm upgrade --to="4.2.29"
Updating to 4.2.29
```

## Upgrade to next channel version

The clusterversion CR handles the upgrades for a certain channel based on the .spec.channel field.

Update this field to use the stable-4.3 channel.

```
$ oc patch clusterversion version --type="merge" -p '{"spec":{"channel":"stable-4.3"}}'
```

Check the updates that are available for this channel.

```
$ oc adm upgrade
Cluster version is 4.2.29

Updates:

VERSION IMAGE
4.3.13
quay.io/openshift-release-dev/ocp-release@sha256:e1ebc7295248a8394afb8d8d918060a7cc3de12c491283b317b80b26deedfe61
```

Upgrade to the major target version.

```
$ oc adm upgrade --to="4.3.13"
Updating to latest version 4.3.13
```

And that it's, after that you have your OCP4 cluster in the latest version available in this
moments.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

---
layout: single
title: Disabling the self provisioning in OpenShift 4
date: 2019-11-11
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How to disable the Self Provisioning in OpenShift 4 clusters to gain more control for the projects created? How to apply a fancy new message when someone with not enough privileges try to create a new project?

### Overview

One of the coolest things about OpenShift is the Role Based Access Control, that allows the
administrators, SREs, etc to control and manage who/when/how the users can create/manage the
different objects inside of OpenShift cluster.

In this particular case, we want to control the possibility that developers creates their own
projects. This have many use cases, such as controlling the projects in a namespace of Production,
avoid the starvation of the resources in a cluster, among others.

Let's deep dive!

### Check the self provisioning objects

The self provisioner cluster role binding is a role binding as a cluster wide (that apply to the
entire cluster not only namespace wide) that will allow the cluster to self provisioning any
new projects.

* Describe the self-provisioners clusterrolebindings:

```
oc describe clusterrolebinding.rbac self-provisioners

Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
```

As we can see, the group that is allowed is the system:authenticated:oauth, that is every user that
is authenticated in the cluster (that have proper login in the cluster).

### Removing Self Provisioning Projects

Deleting the self-provisioners cluster role binding will deny permissions for self-provisioning any
new projects.

* To remove the `self-provisioner` cluster role from the group
`system:authenticated:oauth` you need to remove that group from the role binding.

```
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
```

* Automatic updates reset the cluster roles to a default state. In order to
disable this, you need to set the annotation
`rbac.authorization.kubernetes.io/autoupdate` to `false` by running:

```
oc patch clusterrolebinding.rbac self-provisioners -p '{ "metadata": { "annotations": { "rbac.authorization.kubernetes.io/autoupdate": "false" } } }'
```

* Check that the clusterrolebinding have not the group system:authenticated:oauth among the
allowed groups

```
# oc get clusterrolebinding.rbac self-provisioners -o yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "false"
  creationTimestamp: "2019-10-30T22:08:21Z"
  name: self-provisioners
  resourceVersion: "4408134"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/self-provisioners
  uid: c9117fdf-fb61-11e9-96cb-00505693eda8
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: self-provisioner
```

* Let's check out the result of the operation. Login with a normal user and try to create a project:

```
oc login -u fancyuser1 -p openshift
oc new-project fancyuserproject

Error from server (Forbidden): You may not request a new project via this API.
```

It works!

### Customizing the request message

Now any time a user tries to create a project they will be greated with the
same message `You may not request a new project via this API`. You can
customize this message to give a more meaningful response

```
oc patch --type=merge project.config.openshift.io cluster -p '{"spec":{"projectRequestMessage":"Please visit https://myticket.rober.com to request a project"}}'
```

```
oc login -u fancyuser1 -p openshift
oc new-project fancyuserproject

You should see the following message:

Error from server (Forbidden): Please visit https://myticket.rober.com to request a project
```

Hope that helps!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting







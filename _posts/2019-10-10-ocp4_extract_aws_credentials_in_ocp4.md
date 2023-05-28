---
layout: post
title: Extract AWS Credentials in a cluster of OpenShift 4
date: 2019-10-10
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How obtain the AWS Credentials once the cluster of OCP4 is deployed? Where are they stored in the
cluster?

The AWS Creds are used (among others) by the Machine Config Operator for manage the OpenShift nodes
(worker and nodes) within the cluster as MachineSet and Machines.

This credentials are stored into a Secret into the namespace of "openshift-cloud-credential-operator":

```
$ oc get secret -n openshift-cloud-credential-operator cloud-credential-operator-iam-ro-creds -o yaml
apiVersion: v1
data:
  aws_access_key_id: xxxx
  aws_secret_access_key: yyyy
kind: Secret
metadata:
  annotations:
    cloudcredential.openshift.io/aws-policy-last-applied: '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":["iam:GetUser","iam:GetUserPolicy","iam:ListAccessKeys"],"Resource":"*"},{"Effect":"Allow","Action":["iam:GetUser"],"Resource":"arn:aws:iam::041887290372:user/ocp4-6m565-cloud-credential-operator-iam-ro-2j8m9"}]}'
    cloudcredential.openshift.io/credentials-request: openshift-cloud-credential-operator/cloud-credential-operator-iam-ro
  creationTimestamp: "2019-06-28T14:13:13Z"
  name: cloud-credential-operator-iam-ro-creds
  namespace: openshift-cloud-credential-operator
  resourceVersion: "5044"
  selfLink: /api/v1/namespaces/openshift-cloud-credential-operator/secrets/cloud-credential-operator-iam-ro-creds
  uid: ddb3651a-99ae-11e9-a986-02079fc11896
type: Opaque
```

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!


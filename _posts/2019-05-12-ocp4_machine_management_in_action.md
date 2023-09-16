---
layout: single
title: Machine Management in action into OpenShift 4
date: 2019-05-12
type: post
published: true
status: publish
categories:
- OpenShift
- Networking
- Kubernetes
tags: []
author: rcarrata
comments: true
---

When you deploy a cluster of OpenShift 4.1 using the UPI (User-Provisioned Infrastructure) AWS installation, the deployment can be performed using the AWS Cloudformation templates provided, for create the infrastructure required and afterwards deploy the OCP cluster on top.

By default the cloudformation templates provided, deploys 3 masters but only one worker (in IPI installations 2 workers are deployed instead using the Machine Config Operator).

On the other hand and also by default, when the OpenShift cluster is deployed, two routers are deployed (for give the proper HA to the OCP routes) into the worker nodes of our cluster. For avoid that this two OCP routers are running into the same worker node, two worker nodes minimum are needed to host this routers, but only one is deployed with the Cloudformation templates and only one router is running (the other is in Pending state, as we will see above).

The solution for this problem is to use the Machine Api Operator, for deploy and scale to 2 (or 3) the workers deployed in our cluster. With the proper number of workers, the OCP routers will run perfectly (one into each worker node) and will have the proper HA required for production environments.

## Overview

First of all, we need our OCP4  deployed into AWS using the UPI installation. The result of this installation something will be like this:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get nodes
NAME                                         STATUS   ROLES    AGE   VERSION
ip-10-0-143-54.eu-west-1.compute.internal    Ready    master   23h   v1.13.4+cb455d664
ip-10-0-150-130.eu-west-1.compute.internal   Ready    master   23h   v1.13.4+cb455d664
ip-10-0-158-99.eu-west-1.compute.internal    Ready    worker   22h   v1.13.4+cb455d664
ip-10-0-165-234.eu-west-1.compute.internal   Ready    master   23h   v1.13.4+cb455d664
```

As we see, only one worker node is deployed in our cluster.

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get nodes | grep worker
ip-10-0-158-99.eu-west-1.compute.internal    Ready    worker   23h   v1.13.4+cb455d664
```

For this reason, only one of the two OCP routers in our cluster is running properly in the openshift-ingress namespace:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get pod -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE                                        NOMINATED NODE   READINESS GATES
router-default-76f869f9dc-s48rw   1/1     Running   0          23h   10.131.0.6   ip-10-0-158-99.eu-west-1.compute.internal   <none>           <none>
router-default-76f869f9dc-xh7w4   0/1     Pending   0          98s   <none>       <none>                                      <none>
```

As we can observe, the second router are not running ok, because there isn't a second worker node for host them, and is waiting in "Pending" state.


## Machine Management - Deploy additional worker nodes

So, for fix this and have two routers fully available, we can deploy a new worker node using a MachineSet and the Machine API Operator.

First of all, we can check that are not any MachineSet and Machine present/available:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get machinesets -n openshift-machine-api
No resources found.

[root@clientvm 0 ~/new-ocp4-cf]# oc get machine
No resources found.
```

NOTE: the only worker node present in our cluster was deployed with AWS Cloudformation:

```
[root@clientvm 0 ~/new-ocp4-cf]# aws ec2 describe-instances | jq -r '.Reservations[].Instances[] | select(.Tags[].Value|test(".*rcarrata-cf.*worker.*"))? | select(.State.Name=="running") | .InstanceId'
i-008f1d37b01200ffb

[root@clientvm 0 ~/new-ocp4-cf]# oc get nodes | grep worker
ip-10-0-158-99.eu-west-1.compute.internal    Ready    worker   23h   v1.13.4+cb455d664
```

We can create a MachineSet for deploy a new worker node in eu-west-1a (AZ1 of AWS Ireland region):

```
[root@clientvm 0 ~/new-ocp4-cf]# cat machineset_worker.yml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  labels:
    machine.openshift.io/cluster-api-cluster: rcarrata-cf-7lk9g
  name: rcarrata-cf-7lk9g-worker-2-eu-west-1a
  namespace: openshift-machine-api
spec:
  replicas: 1
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: rcarrata-cf-7lk9g
      machine.openshift.io/cluster-api-machine-role: worker
      machine.openshift.io/cluster-api-machine-type: worker
      machine.openshift.io/cluster-api-machineset: rcarrata-cf-7lk9g-worker-eu-west-1a
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: rcarrata-cf-7lk9g
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: rcarrata-cf-7lk9g-worker-eu-west-1a
    spec:
      metadata:
        labels:
          node-role.kubernetes.io/worker: ""
      providerSpec:
        value:
          ami:
            id: ami-00d18cbff03587a41
          apiVersion: awsproviderconfig.openshift.io/v1beta1
          blockDevices:
            - ebs:
                iops: 0
                volumeSize: 120
                volumeType: gp2
          credentialsSecret:
            name: aws-cloud-credentials
          deviceIndex: 0
          iamInstanceProfile:
            id: clustersecurity-WorkerInstanceProfile-HGYXROC3XT5E
          instanceType: m4.large
          kind: AWSMachineProviderConfig
          placement:
            availabilityZone: eu-west-1a
            region: eu-west-1
          securityGroups:
            - filters:
                - name: "tag:aws:cloudformation:logical-id"
                  values:
                    - WorkerSecurityGroup
          subnet:
            filters:
              - name: tag:Name
                values:
                  - rcarrata-xbyo-m2d2l-private-eu-west-1a
          tags:
            - name: kubernetes.io/cluster/rcarrata-cf-7lk9g
              value: owned
          userDataSecret:
            name: worker-user-data
status:
  replicas: 0
```

Apply the machineset worker yaml with the definitions of the worker node:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc apply -f machineset_worker.yml
machineset.machine.openshift.io/rcarrata-cf-7lk9g-worker-2-eu-west-1a created
```

And check the machinesets and the machines created:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get machineset
NAME                                    DESIRED   CURRENT   READY   AVAILABLE   AGE
rcarrata-cf-7lk9g-worker-2-eu-west-1a   1         1                             4s
```

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get machine
NAME                                          INSTANCE              STATE     TYPE       REGION      ZONE         AGE
rcarrata-cf-7lk9g-worker-2-eu-west-1a-r4sqk   i-0a07c73b9d8b438bc   pending   m4.large   eu-west-1   eu-west-1a   7s
```

Check with the aws-cli tool the brand new instance that will be our new brand worker node:

```
[root@clientvm 0 ~/new-ocp4-cf]# aws ec2 describe-instances | jq -r '.Reservations[].Instances[] \
| select(.Tags[].Value|test(".*rcarrata-cf.*worker.*"))? \
| select(.State.Name=="pending") | .InstanceId'
i-0a07c73b9d8b438bc
```

Once the worker node is in state "Running", the machineset reflects that the Desired State and the Ready / Available States have matching resources (1 worker in this case):

```
[root@clientvm 0 ~/new-ocp4-cf]# aws ec2 describe-instances | jq -r '.Reservations[].Instances[] | \
select(.Tags[].Value|test(".*rcarrata-cf.*worker.*"))? \
| select(.State.Name=="running") | .InstanceId'
i-008f1d37b01200ffb
i-0a07c73b9d8b438bc
```

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get machine
NAME                                          INSTANCE              STATE     TYPE       REGION      ZONE         AGE
rcarrata-cf-7lk9g-worker-2-eu-west-1a-r4sqk   i-0a07c73b9d8b438bc   running   m4.large   eu-west-1   eu-west-1a   42s

[root@clientvm 0 ~/new-ocp4-cf]# oc get machineset
NAME                                    DESIRED   CURRENT   READY   AVAILABLE   AGE
rcarrata-cf-7lk9g-worker-2-eu-west-1a   1         1         1       1           6m3s
```

## Conclusion - Check OpenShift Cluster and Routers status

So now, the new worker node is up && running in our cluster of OpenShift:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get nodes
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-0-137-15.eu-west-1.compute.internal    Ready    worker   2m26s   v1.13.4+cb455d664
ip-10-0-143-54.eu-west-1.compute.internal    Ready    master   23h     v1.13.4+cb455d664
ip-10-0-150-130.eu-west-1.compute.internal   Ready    master   23h     v1.13.4+cb455d664
ip-10-0-158-99.eu-west-1.compute.internal    Ready    worker   23h     v1.13.4+cb455d664
ip-10-0-165-234.eu-west-1.compute.internal   Ready    master   23h     v1.13.4+cb455d664
```

Because of we have an additional worker node in our cluster, the second router can run in this brand new worker:

```
[root@clientvm 0 ~/new-ocp4-cf]# oc get pod -n openshift-ingress
NAME                              READY   STATUS    RESTARTS   AGE
router-default-76f869f9dc-4dqhj   1/1     Running   0          18m
router-default-76f869f9dc-s48rw   1/1     Running   0          23h
```

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
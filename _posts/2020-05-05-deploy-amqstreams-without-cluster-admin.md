---
layout: post
title: Deploying AMQStreams Kafka without Cluster Admin
date: 2020-05-05
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How to deploy fully automated a Kafka cluster using AMQStreams / Strimzi? And how to deploy Kafka
clusters without being cluster-admin, using always regular users or local admins to the namespace?

Let's dig in!

## Overview of AMQStreams / Strimzi / Kafka

AMQ Streams provides Operators for managing a Kafka cluster running within an OpenShift cluster.

* Cluster Operator: Deploys and manages Apache Kafka clusters, Kafka Connect, Kafka MirrorMaker, Kafka Bridge, Kafka Exporter, and the Entity Operator

* Entity Operator: Comprises the Topic Operator and User Operator

* Topic Operator: Manages Kafka topics

* User Operator: Manages Kafka users

The Cluster Operator can deploy the Topic Operator and User Operator as part of an Entity Operator configuration at the same time as a Kafka cluster.

## Description of this scenario

The goal in this blog post is to deploy a Kafka cluster in OpenShift, using the AMQStreams 1.4
operators (based in Strimzi version 0.17.x and Kafka 2.4.0) deployed as a cluster-admin, and giving
strimzi-admin to a regular user to deploy Kafka clusters without being cluster-admin.

Tested and working with 3.11 and 4.3.x.

## Installation AMQStreams operator

In the first part of this blog, the AMQStreams Kafka cluster operator, needs to be set deployed as a
cluster-admin.

The namespaces used for this installation will be:

* Namespace for deploy the AMQStreams Cluster Operator: **kafkaproject**
* Namespace for deploy the Kafka Cluster / Kafka Topics: **test-kafka**

### Installation of AMQStreams 1.4 Cluster Kafka Operators

This steps need to be implemented with a user with cluster-admin role, such as system:admin.

1. Download the bits of AMQStreams 1.4 in [AMQStreams download
site](https://access.redhat.com/jbossnetwork/restricted/listSoftware.html?downloadType=distributions&product=jboss.amq.streams)

2. Extract the files needed for the installation:

```
$ mkdir $HOME/amq_1.4_install
$ unzip amq-streams-1.4.0-ocp-install-examples.zip -d $HOME/amq_1.4_install
$ export AMQ_STREAMS_HOME=$HOME/amq_1.4_install
```

3. Login as an cluster-admin user, in our case with the name admin:

```
$ oc login -u admin -p xxxxxx
$ oc get users admin
NAME      UID                                    FULL NAME   IDENTITIES
admin     5ac66c87-9376-11ea-ac11-02ec1020384e               htpasswd_auth:admin
```

4. With this user cluster admin (only used for deploying the cluster-operator), create the namespace/ project for the AMQStreams Kafka Cluster Operator

```
$ oc new-project kafkaproject
$ cd $AMQ_STREAMS_HOME
```

5. Modify the installation files (rolebindings among others) to reference the new kafkaproject namespace where you will install the AMQ Streams Kafka Cluster Operator.

```
$ sed -i 's/namespace: .*/namespace: kafkaproject/' install/cluster-operator/*RoleBinding*.yaml
```

6. Deploy the CRDs and role-based access control (RBAC) resources to manage the CRDs.

```
$ oc project kafkaproject
$ oc apply -f install/cluster-operator/
```

* Check that the cluster operator is deployed properly.

```
$ oc get pod
NAME                                        READY     STATUS    RESTARTS   AGE
strimzi-cluster-operator-66f9b9c68d-jk6w2   0/1       Running   0          1m
```

## Prepare and adjust the roles and bindings to allow users interact to the kafka cluster operator/crd

For allow to deploy and manage by a regular user (non cluster-admin), the Kafka CRDs created in the installation of the AMQStreams operators, some adjusts need to be done.

6. Create a new test-kafka namespace where you will deploy your Kafka cluster.

```
$ oc new-project test-kafka
```

7. Give access to my-kafka-project to a non-admin user andrew.

Our developer Andrew, wanted to create kafka clusters but without have the cluster-admin privileges.

Andrew is a regular user with non privileged within the group of "system:authenticated:oauth".

This user have no privileges for read/modify the resources in the namespace that will be used for
deploy the kafka cluster:

```
# oc get pvc --as=andrew
No resources found.
Error from server (Forbidden): persistentvolumeclaims is forbidden: User "andrew" cannot list
persistentvolumeclaims in the namespace "test-kafka": no RBAC policy matched
```

NOTE: For determine the full list of verbs that can the user perform use:

```
oc policy can-i --list --as=andrew
```


8. give to Andrew user local admin privileges (namespace-wide):

```
oc adm policy add-role-to-user admin andrew -n test-kafka
role "admin" added: "andrew"
```

Check that the user andrew only have capabilities of admin in the namespace test-kafka:

```
# oc get node --as=andrew
No resources found.
Error from server (Forbidden): nodes is forbidden: User "andrew" cannot list nodes at the cluster
scope: no RBAC policy matched

$ oc get pvc --as=andrew -n test-kafka
 No resources found.

$ oc get secret --as=andrew -n test-kafka
NAME                       TYPE                                  DATA      AGE
builder-dockercfg-j9b4m    kubernetes.io/dockercfg               1         2h
builder-token-7fxmp        kubernetes.io/service-account-token   4         2h
builder-token-kx8zm        kubernetes.io/service-account-token   4         2h
default-dockercfg-pgs8j    kubernetes.io/dockercfg               1         2h
```

As we can see, Andrew can list PVCs and secrets into the namespace test-kafka, but not list nodes.

9. Give permission to the Cluster Operator to watch the my-kafka-project namespace.

```
$ oc set env deploy/strimzi-cluster-operator STRIMZI_NAMESPACE=kafkaproject,test-kafka -n
 kafkaproject
 deployment.extensions/strimzi-cluster-operator updated
```

```
$ oc apply -f install/cluster-operator/020-RoleBinding-strimzi-cluster-operator.yaml -n test-kafka
 rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator created

$ oc apply -f install/cluster-operator/032-RoleBinding-strimzi-cluster-operator-topic-operator-delegation.yaml -n test-kafka
 rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-topic-operator-delegation created

$ oc apply -f install/cluster-operator/031-RoleBinding-strimzi-cluster-operator-entity-operator-delegation.yaml -n test-kafka
 rolebinding.rbac.authorization.k8s.io/strimzi-cluster-operator-entity-operator-delegation created
```

The commands create role bindings that grant permission for the Cluster Operator to access the Kafka cluster.

10. Create a new cluster role strimzi-admin.

```
# oc apply -f install/strimzi-admin
clusterrole.rbac.authorization.k8s.io/strimzi-admin created
```

This cluster role defines the strimzi-admin, a role that allow the interaction with the CRDs and the
kafka cluster operator:

```
$ cat install/strimzi-admin/010-ClusterRole-strimzi-admin.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: strimzi-admin
  labels:
    app: strimzi
rules:
- apiGroups:
  - "kafka.strimzi.io"
  resources:
  - kafkas
  - kafkaconnects
  - kafkaconnects2is
  - kafkamirrormakers
  - kafkausers
  - kafkatopics
  - kafkabridges
  - kafkaconnectors
  - kafkamirrormaker2s
  verbs:
  - get
  - list
  - watch
  - create
  - delete
  - patch
  - update
```

11. Check that the local admin user andrew can't list any crd of kafka:

```
$ oc get kafkausers --as=andrew
Error from server (Forbidden): kafkausers.kafka.strimzi.io is forbidden: User "andrew" cannot list kafkausers.kafka.strimzi.io in the namespace "test-kafka": no RBAC policy matched
```

```
# oc get kafkaconnects --as=andrew
No resources found.
Error from server (Forbidden): kafkaconnects.kafka.strimzi.io is forbidden: User "andrew" cannot list kafkaconnects.kafka.strimzi.io in the namespace "test-kafka": no RBAC policy matched
```

12. Apply the clusterroles to the user andrew to allow them to manage objects from the kafka crds (and also interaction with the kafka cluster operator):

```
$ oc adm policy add-cluster-role-to-user strimzi-admin andrew
cluster role "strimzi-admin" added: "andrew"
```

Check that the andrew user can now to manage kafka crds objects:

```
$ oc get kafkaconnects --as=andrew
No resources found.
```

## Deploy Kafka Clusters as a regular user

13. Login as andrew and switch to the namespace test-kafka

```
$ oc login -u andrew
$ oc project test-kafka
$ oc whoami
andrew
```

14. Create a new my-cluster Kafka cluster with 3 Zookeeper and 3 broker nodes.

```
$ cat << EOF | oc create -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    listeners:
      plain: {}
      tls: {}
      external:
        type: route
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
EOF
kafka.kafka.strimzi.io/my-cluster created
```

```
# oc get pod -n test-kafka
NAME                                         READY     STATUS    RESTARTS   AGE
my-cluster-entity-operator-556b956cf-zm4ls   2/2       Running   0          1m
my-cluster-kafka-0                           2/2       Running   0          2m
my-cluster-kafka-1                           2/2       Running   0          2m
my-cluster-kafka-2                           2/2       Running   0          2m
my-cluster-zookeeper-0                       2/2       Running   0          4m
my-cluster-zookeeper-1                       2/2       Running   0          4m
my-cluster-zookeeper-2                       2/2       Running   0          4m

$ oc get kafka my-cluster
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS
my-cluster   3                        3
```

## Deploy a Topic in your Kafka cluster

15. When your cluster is ready, create a topic to publish and subscribe from your external client.

```
cat << EOF | oc create -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: rober-topic
  labels:
    strimzi.io/cluster: "my-cluster"
spec:
  partitions: 3
  replicas: 3
EOF
kafkatopic.kafka.strimzi.io/rober-topic created
```

```
$ oc get kafkatopic
NAME          PARTITIONS   REPLICATION FACTOR
rober-topic   3            3
```

## Links

* [Requirements of Admin Privileges for install AQMStreams 1.4](https://access.redhat.com/documentation/en-us/red_hat_amq/7.6/html-single/using_amq_streams_on_openshift/index#co-faq-admin-privileges_str)
* [Designating Strimzi administrators](https://access.redhat.com/documentation/en-us/red_hat_amq/7.6/html-single/using_amq_streams_on_openshift/index#proc-deploying-the-user-operator-using-the-cluster-operator-str)
* [Deploying Strimzi Administrators](https://strimzi.io/docs/operators/master/deploying.html#adding-users-the-strimzi-admin-role-str)

And that's it! Hope that helps!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy AMQStreaming!

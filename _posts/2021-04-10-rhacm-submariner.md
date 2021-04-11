---
layout: post
title: Deep Dive with RHACM and Submariner - Connecting multicluster overlay networks
date: 2021-04-10
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How to connect your overlay networks of different Kubernetes clusters? How can you deploy stateful applications spanning in a multicluster cluster environments? 

Let's dig in! 

### Overview

Deploy our workloads in multiclustering environment is hard. And if you want to span your applications and connect them between them, to deploy it in several datacenters / clouds, it's even more difficult. 

But with the help of RHACM and Submariner this is a bit easier.

RHACM can help deploying apps, managing multiple clusters, and enforcing policies across multiple clusters at scale. 

But how to connect our pods and services between two different clusters, enabling the communication between different microservices deployed into different clusters? 

The first thought is to use the Ingress Controllers or Openshift Routers, right? But this have several setbacks and it's not quite flexible, because all of your traffic needs to egress through one of your clusters to another, and ingress in the destination cluster using the specific routes / ingresses exposing our apps. 

But, if I want to connect or extend my Pod / Service overlays networks from one cluster to another? 

Let's introduce Submariner that helps with our problem.

### Submariner - Connecting Kubernetes clusters networks 

Submariner is an open source tool that can be used with RHACM to provide direct networking between two or more Kubernetes clusters in your environment, either on-premises or in the cloud. Submariner connects multiple Kubernetes clusters in a way that is secure and performant. 

For doing that, Submariner flattens the networks between the connected clusters, and enables IP reachability between Pods and Services. Submariner also provides, via Lighthouse, service discovery capabilities. 

The diagram below illustrates the basic architecture of Submariner:

[![](/images/submariner2.svg "Submariner Diagram")]({{site.url}}/images/submariner2.svg)

For more information about the architecture of Submariner check the [official documentation of Submariner](https://submariner.io/getting-started/architecture/)

### Prerequisites and Scenario

This blog post is tested with the following components:

* RHACM Hub version 2.2.1 on top of OCP 4.7
* OCP 4.7.5 in AWS (aws-sub1) in region eu-west-1
* OCP 4.7.5 in AWS (aws-sub2) in region eu-west-1

[![](/images/submariner1.png "ACM Diagram")]({{site.url}}/images/submariner1.png)

Important! The two cluster CIDRs (ServiceCIDR and ClusterCIDR) cannot overlap. 
The ClusterNetworks and ServiceNetworks for our clusters are the following:

* Cluster1 (aws-sub1): **ClusterNetwork** 10.132.0.0/14 and **ServiceNetwork** 172.31.0.0/16

* Cluster2 (aws-sub2): **ClusterNetwork** 10.128.0.0/14 and **ServiceNetwork** 172.30.0.0/16

NOTE: check that in your region is available the type of instance m5n.large. This will be used within the installation of the submariner addon.

### Set up the context of our clusters

Let's create several contexts for switching from one to another in an easy way.

* Define a new kubeconfig:

```
$ rm -rf /var/tmp/acm-lab-kubeconfig
$ touch /var/tmp/acm-lab-kubeconfig
$ export KUBECONFIG=/var/tmp/acm-lab-kubeconfig
$ export CLUSTER1=aws-sub1
$ export CLUSTER2=aws-sub2
```

* Log in in the hub cluster (where ACM is deployed), and set the context:

```
$ oc login -u ocp-admin -p xxxx --insecure-skip-tls-verify --server=https://api.cluster-xxxx.xxxx.example.rcarrata.com:6443
$ oc config rename-context $(oc config current-context) hubcluster
```

* Log in in the cluster1 (aws-sub1), and set the context:

```
$ oc login -u kubeadmin --server=https://api.aws-sub1.xxxx.example.rcarrata.com:6443 -p xxxx
$ oc config rename-context $(oc config current-context) cluster1
```

* Log in in the cluster2 (aws-sub1), and set the context:

```
$ oc login -u kubeadmin -p xxxx https://api.aws-sub2.xxxx.example.rcarrata.com:6443
$ oc config rename-context $(oc config current-context) cluster2
```

### Preparing the OCP hosts for deploy Submariner

Before deploying Submariner with RHACM, you must prepare the clusters on the hosting environment for the connection. 

There are two ways that you can configure the OpenShift Container Platform cluster that is hosted on Amazon Web Services to integrate with a Submariner deployment.

Let's use the automated way because both clusters are in AWS (in this moment is the only supported way to connect automatically). If you are in other environments check the [RHACM Documentation for the 2.2 version of ACM](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.2/html-single/manage_cluster/index#preparing-gcp)

* Switch back to the Hub Cluster (ACM):

```
$ oc config use-context hubcluster
```

* Extract the names of the secrets of the aws-creds of the managed clusters to connect (aws-sub1, and aws-sub2)

```
$ SECRET_NAME_1=$(oc get secret aws-sub1-aws-creds -n $CLUSTER1 -o jsonpath='{.metadata.name}')
$ SECRET_NAME_2=$(oc get secret aws-sub2-aws-creds -n $CLUSTER2 -o jsonpath='{.metadata.name}')
```

You can use the SubmarinerConfig API to build the cluster environment. With this method, the submariner-addon configures the environment, so you use your configurations and cloud provider credentials in your SubmarinerConfig definition. 

* Define the SubmarinerConfig for the first managed cluster:

```
$ cat << EOF | oc apply -f -
apiVersion: submarineraddon.open-cluster-management.io/v1alpha1
kind: SubmarinerConfig
metadata:
  name: subconfig
  namespace: $CLUSTER1
spec:
  credentialsSecret:
     name: $SECRET_NAME_1
EOF
```

* Check that the SubmarinerConfig is generated properly:

```
$ oc get SubmarinerConfig -n $CLUSTER1
NAME        AGE
subconfig   14s
```

* Define the SubmarinerConfig for the second managed cluster:

```
$ cat << EOF | oc apply -f -
apiVersion: submarineraddon.open-cluster-management.io/v1alpha1
kind: SubmarinerConfig
metadata:
  name: subconfig
  namespace: $CLUSTER2
spec:
  credentialsSecret:
     name: $SECRET_NAME_2
EOF

$ oc get SubmarinerConfig -n $CLUSTER2
NAME        AGE
subconfig   26s
```

* Let's check the logs of the Submariner Addon Pod in the ACM Hub and figure out what's behind the hood:

```
$ SUBMARINER_POD=$(oc get pod -n open-cluster-management | grep submari | awk '{ print $1 }')
$ oc logs -f --tail=5 $SUBMARINER_POD  -n open-cluster-management
I0409 22:12:52.085687       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"open-cluster-management", Name:"application-chart-0e047-applicationui", UID:"xxxx", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'SubmarinerRoutePortOpened' the submariner route port is opened on aws
I0409 22:12:52.715964       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"open-cluster-management", Name:"application-chart-0e047-applicationui", UID:"xxxx", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'SubmarinerMetricsPortOpened' the submariner metrics port is opened on aws
I0409 22:12:52.975213       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"open-cluster-management", Name:"application-chart-0e047-applicationui", UID:"xxxx", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'SubmarinerIPsecPortsOpened' the submariner IPsec ports are opened on aws
I0409 22:12:53.119162       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"open-cluster-management", Name:"application-chart-0e047-applicationui", UID:"xxxx", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'SubmarinerSubnetTagged' the subnet subnet-0597bd670b9e542e9 is tagged on aws
I0409 22:12:53.123697       1 event.go:282] Event(v1.ObjectReference{Kind:"Deployment", Namespace:"open-cluster-management", Name:"application-chart-0e047-applicationui", UID:"xxxx", APIVersion:"apps/v1", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'SubmarinerGatewayManifestworkCreated' the submariner gateway manifestwork aws-submariner-gateway-machineset is created
```

### Deep Dive on the SubmarinerConfig configurations 

#### IPsec and IKE in Submariner

After the generation and configuration of SubmarinerConfig for our both managed clusters that wanted to connect. Let's deep dive a bit in what's generated in our OCP cluster and in AWS environments.

Submariner uses IPsec tunnels and IKE Protocols for the communications of their clusters:

* Internet Key Exchange is the protocol used to set up a security association (SA) in the IPsec protocol suite. 

* Internet Protocol Security (IPsec) is a secure network protocol suite that authenticates and encrypts the packets of data to provide secure encrypted communication between two computers over an Internet Protocol network. It is usually used in virtual private networks (VPNs).

For that purpose, Submariner Gateway nodes need to be able to accept traffic over UDP ports (4500 and 500 by default) when using IPsec. 

Submariner also uses UDP port 4800 to encapsulate traffic from the worker and master nodes to the Gateway nodes, and TCP port 8080 to retrieve metrics from the Gateway nodes. 

So both needs to be opened at the AWS level, for this reason these ports are opened into the security groups during the SubmarinerConfig automatic configuration:

[![](/images/submariner3.png "ACM Diagram 3")]({{site.url}}/images/submariner3.png)

Also into the Security Group of the workers of OCP, ports 4800 and 8080 are opened too, beside the 4800, 4500 and 500 UDP needed for IKE and IPsec.

#### Specific nodes for Submariner

Additionally, the default OpenShift deployment does not allow assigning an elastic public IP to existing worker nodes, which may be necessary on one end of the IPsec connection.

On the other hand, the SubmarinerConfig deploys an m5n.large EC2 instance type by default, optimized for improved network throughput and packet rate performance, for the Submariner gateway node.

Let check the MachineSet generated for this purpose:

```
$ oc config use-context cluster1

$ oc get machineset -n openshift-machine-api | grep submariner
aws-sub1-xxx-submariner-gw-eu-west-1b   1         1         1       1           14h

$ oc get machines -n openshift-machine-api | grep submariner
aws-sub1-xxx-submariner-gw-eu-west-1b-xx52q   Running   m5n.large   eu-west-1   eu-west-1b   4m
```

In the machineset we found some interesting points, as for example have labels of submariner.io/gateway, that labels the node as the Submariner Gateway node:

```
$ oc get machineset aws-sub1-xxx-submariner-gw-eu-west-1b -n openshift-machine-api -o jsonpath={'.spec.template'}  | jq -r .spec.metadata
{
  "labels": {
    "submariner.io/gateway": "true"
  }
}
```

As we commented the type of the Instance is [m5n.large](https://aws.amazon.com/blogs/aws/new-m5n-and-r5n-instances-with-up-to-100-gbps-networking/), AWS EC2 instances with up to 100Gbps networking, and significant improvements in packet processing performance. : 

```
$ oc get machineset aws-sub1-xxx-submariner-gw-eu-west-1b -n openshift-machine-api -o jsonpath={'.spec.template'}  | jq -r .spec.providerSpec.value.instanceType
m5n.large
```

On the other hand also have attached the two security groups with the ports for IPsec and IKE opened for connecting the overlay networks of both clusters:

```
$ oc get machineset aws-sub1-xxx-submariner-gw-eu-west-1b -n openshift-machine-api -o jsonpath={'.spec.template'}  | jq -r .spec.providerSpec.value.securityGroups
[
  {
    "filters": [
      {
        "name": "tag:Name",
        "values": [
          "aws-sub1-xxx-worker-sg",
          "aws-sub1-xxx-submariner-gw-sg"
        ]
      }
    ]
  }
]
```

Finally we have two interesting values additionally, the subnet id where the node is deployed and the value tags, defining that this node is the submariner gateway node used for this purpose: 

```
$ oc get machineset aws-sub1-xxx-submariner-gw-eu-west-1b -n openshift-machine-api -o jsonpath={'.spec.template'}  | jq -r .spec.providerSpec.value.subnet
{
  "filters": [
    {
      "name": "tag:Name",
      "values": [
        "aws-sub1-xxx-public-eu-west-1b"
      ]
    }
  ]
}
```

```
$ oc get machineset aws-sub1-xxx-submariner-gw-eu-west-1b -n openshift-machine-api -o jsonpath={'.spec.template'}  | jq -r .spec.providerSpec.value.tags
[
  {
    "name": "kubernetes.io/cluster/aws-sub1-xxx",
    "value": "owned"
  },
  {
    "name": "submariner.io",
    "value": "gateway"
  }
]
```

If we check in the second managed cluster (aws-sub2) we discover the same type of node:

```
$ oc config use-context cluster2

$ oc get machines -n openshift-machine-api | grep submariner
aws-sub2-xxx-submariner-gw-eu-west-1a-xxx   Running   m5n.large   eu-west-1   eu-west-1a   5m46s
```

NOTE: These steps are performed automatically because the environment is AWS. For other clouds it's a manual process that needs to be implemented to open the ports, sgs, tag the subnets, etc.

### Deploying Submariner

Create a ManagedClusterSet on the hub cluster by using the instructions provided in ManagedClusterSets.

A ManagedClusterSet is a group of managed clusters. With a ManagedClusterSet, you can manage access to all of the managed clusters in the group together. You can also create a ManagedClusterSetBinding resource to bind a ManagedClusterSet resource to a namespace. 

* Create a ManagedClusterSet on the hub cluster:

```
$ oc config use-context hubcluster

$ cat << EOF | kubectl apply -f -
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterSet
metadata:
  name: submariner
EOF

managedclusterset.cluster.open-cluster-management.io/submariner created
```

* The submariner-addon creates a namespace called submariner-clusterset-<clusterset-name>-broker and deploys the Submariner Broker to it:

```
$ oc get ns submariner-clusterset-submariner-broker
NAME                                      STATUS   AGE
submariner-clusterset-submariner-broker   Active   9s
```

* Check the managed clusters status: 

```
$ oc get managedclusters | grep aws
aws-sub1        true                                  True     True        26h
aws-sub2        true                                  True     True        3h56m

$ oc get managedclusterset
NAME         AGE
submariner   4m59s
```

* Enable Submariner to provide communication between managed clusters by entering the following command: 

```
$ oc label managedclusters aws-sub1 "cluster.open-cluster-management.io/submariner-agent=true" --overwrite
managedcluster.cluster.open-cluster-management.io/aws-sub1 labeled

$ oc label managedclusters aws-sub2 "cluster.open-cluster-management.io/submariner-agent=true" --overwrite
managedcluster.cluster.open-cluster-management.io/aws-sub2 labeled
```

* Add the managed clusters to the ManagedClusterSet by entering the following command: 

```
$ oc label managedclusters aws-sub1 "cluster.open-cluster-management.io/clusterset=submariner" --overwrite
managedcluster.cluster.open-cluster-management.io/aws-sub1 labeled

$ oc label managedclusters aws-sub2 "cluster.open-cluster-management.io/clusterset=submariner" --overwrite
managedcluster.cluster.open-cluster-management.io/aws-sub2 labeled
```

* Check that the managed clusters have the labels properly defined:

```
$ oc get managedcluster aws-sub1 --show-labels
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE   LABELS
aws-sub1   true                                  True     True        27h   cloud=Amazon,cluster.open-cluster-management.io/clusterset=submariner,cluster.open-cluster-management.io/submariner-agent=true,clusterID=xxx,environment=qa,name=aws-sub1,region=eu-west-1,vendor=OpenShift

$ oc get managedcluster aws-sub2 --show-labels
NAME       HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE     LABELS
aws-sub2   true                                  True     True        4h13m   cloud=Amazon,cluster.open-cluster-management.io/clusterset=submariner,cluster.open-cluster-management.io/submariner-agent=true,clusterID=xxx,environment=qa,name=aws-sub2,region=eu-west-1,vendor=OpenShift
```

And that's it for this blog post! In the [next blog post](https://rcarrata.com/openshift/rhacm-submariner-2/) we will deploy the Service discovery for Submariner and we will deploy an stateful application spanning their microservices among our two connected clusters with Submariner.

Happy Submarining! 
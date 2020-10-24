---
layout: post
title: Multiple Route Shardings in the same node in a On-Premise Openshift 4
date: 2020-10-23
type: post
published: true
status: publish
categories:
- Openshift
tags: []
author: rcarrata
comments: true
---

How to run a Route Sharding in the same Openshift nodes in a On-premise installation (UPI or IPI)? What are the steps for configuring it?

Let's dig in!

### Overview & Example Scenario

This example is applicable in all UPI deployments in On-premise (Baremetal, VMWare UPI, RHV UPI,
OSP UPI, etc) and also the RHV IPI, VMWARE IPI and Baremetal IPI (with some differences that will be
analyzed in next blog posts) of Openshift 4.

In this example a Openshift 4.5 UPI on top of Openstack is used (same as the previous blog post).

### Why can't I run two Route Sharding in the same node?

As we checked in the previous blog post about the [Deep Dive of Openshift 4 Routers in On-Premise Deployments](https://rcarrata.com/openshift/ocp4_upi_routers/) the routers in an installation non-cloud are a bit different. That is because the object of Kubernetes [Loadbalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) is not available, and the Openshift Routers are deployed with a Service ClusterIP and with the endpointPublishingStrategy of **HostNetwork: true**.

This is because HostNetwork allowed the Openshift Router Pods can directly see the network
interfaces of the host machine (Openshift Node) where the host is started and exposes the ports of
the Routers (HTTP and HTTPS, and metrics port) for Load Balancing for an external physical or virtual LB.

But with by default behaviour of the IngressController and the Openshift Routers you CAN'T have a
Route Sharding in the same worker/infra node, because it will overlap the Ports into the Openshift
Node (remember that the OCP Router is listening at the Node interface not in the Pod Network).

For this reason, for each Route Sharding you need to create another separate node for each Route
Sharding. And because we will always HA of hour Openshift Routers, is 4 worker or infras for a 2
route sharding (2 Nodes for the default node and 2 more nodes for the Shard).

Back in Openshift 3.11 was allowed to [have several shards in the same router](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#routers), changing the ports were the route sharding routers were listening to not collision to the default routers:

```
ROUTER_SERVICE_HTTP_PORT
ROUTER_SERVICE_HTTPS_PORT
```

But in Openshift 4, we had the Openshift Routers controlled and managed by the Openshift Ingress
Controllers.

* Before the Openshift4.4 there was no way to use the Ingress Operator to deploy more than a router
per node.

* But from Openshift4.4+ there's a way to run more than a router using the IngressOperator
NodePortService endpoint publishing strategy.

### Endpoint Publishing Comparison - NodePort vs HostNetwork

* NodePortService

The NodePortService strategy publishes an ingress controller using a Kubernetes NodePort Service.
With this strategy, the administrator is responsible for configuring any external DNS or load balancer.

[![](/images/endpoint-publishing-hostnetwork.png "Endpoint Publishing HostNetwork")]({{site.url}}/images/endpoint-publishing-hostnetwork.png)

* HostNetwork

The HostNetwork strategy uses host networking to publish the ingress controller directly on the node
host where the ingress controller is deployed.

[![](/images/endpoint-publishing-nodeportservice.png "Endpoint Publishing Service")]({{site.url}}/images/endpoint-publishing-nodeportservice.png)

NOTE: This diagrams are extracted from the original repo of [cluster-ingress-operator](https://github.com/openshift/cluster-ingress-operator#hostnetwork) in github.

### Creating new Route Shardings in the Same Openshift 4 Node

* Define a new IngressController resource in openshift-ingress-operator namespace that will be our brand new Route Sharding:

```
$ cat ingress-custom.yml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: custom
  namespace: openshift-ingress-operator
spec:
  replicas: 1
  endpointPublishingStrategy:
    type: NodePortService
  domain: apps2.sharedocp4upi45.lab.upshift.rdu2.redhat.com
```

The most important part in this IngressController definition is the enpointPublishingStrategy, where need to be defined as [type:NodePortService](https://github.com/openshift/api/blob/master/operator/v1/types_ingress.go#L225)

In the IngressController Operator, the EndpointPublishingStrategyType is a way to publish ingress controller endpoints.
At this specific case, the NodePortService publishes the ingress controller using a Kubernetes NodePort Service, instead of the by default endpointPublishingStrategy type:HostNetwork (that publishes the ingress controller on node ports where the ingress controller is deployed).

* Create the IngressController resource in openshift-ingress-operator namespace:

```
$ oc create -f ingress-custom.yml
ingresscontroller.operator.openshift.io/default configure
```

* Check that the pods of the Openshift Routers custom are creating:

```
$ oc get pod -n openshift-ingress
NAME                              READY   STATUS              RESTARTS   AGE
router-custom-568c9c455-p47q2     0/1     ContainerCreating   0          23s
router-default-754bf5f974-rpzq5   1/1     Running             0          4m53s
router-default-754bf5f974-wtrnb   1/1     Running             2          4m53s
```

* Check the difference between the router custom and the router default:

```
$ oc get pod -n openshift-ingress router-custom-568c9c455-p47q2 -o json | jq -r '.spec.hostNetwork'
null
```

```
$ oc get pod -n openshift-ingress router-default-754bf5f974-rpzq5 -o json | jq -r '.spec.hostNetwork'
true
```

As we can see the main difference is that the custom router is not using the hostNetwork, and the
default router it is. This is because we configured the endpointPublishingStrategy in the custom
router as **type:HostNetwork**.

* Check the location and the IP of the Openshift Router pods:

```
$ oc get pod -o wide -n openshift-ingress
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE NOMINATED NODE   READINESS GATES
router-custom-568c9c455-p47q2     1/1     Running   0          2m3s    10.131.0.120 worker-1.sharedocp4upi45.lab.upshift.rdu2.redhat.com   <none>           <none>
router-default-754bf5f974-rpzq5   1/1     Running   0          6m33s   10.0.92.117 worker-0.sharedocp4upi45.lab.upshift.rdu2.redhat.com   <none>           <none>
router-default-754bf5f974-wtrnb   1/1     Running   0          6m33s   10.0.93.214 worker-1.sharedocp4upi45.lab.upshift.rdu2.redhat.com   <none>           <none>
```

NOTE: If you noticed the pods for the Router Custom and Default Router Pods are located in the same
Openshift Node (worker1 in this case). If you create another route sharding using other Ingress Controller you will also can allocate and run the pods in the same Nodes that we checked before, coexisting several Route Sharding router pods in the same Nodes.

* Check the Openshift ingress Services that are created the IngressControllers Default and Custom:

```
oc get svc -n openshift-ingress
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
router-internal-custom    ClusterIP   172.30.71.122   <none>        80/TCP,443/TCP,1936/TCP      7m11s
router-internal-default   ClusterIP   172.30.108.18   <none>        80/TCP,443/TCP,1936/TCP      11m
router-nodeport-custom    NodePort    172.30.1.17     <none>        80:32596/TCP,443:32121/TCP   7m11s
```

As you can notice the Service of the router-internal-default that only has the ClusterIP service.
ClusterIP Service exposes the Service on a cluster-internal IP. This value makes the Service only reachable from within the cluster.

On the other hand the Route Sharding Custom Ingress Controlled created two Services, a ClusterIP and a NodePort, because the hostNetwork is enabled, and the Openshift Routers needsto be accessible in away (through the NodePort and not with HostNetwork).

Remember that the NodePort is used to make the service accessible from outside of the cluster
opening a static port on each Node's.

* If we check the router-internal-custom ClusterIP service in detail we can see the endpoints of the pods,
  that are in the pod Network instead of the hostNetwork

```
$ oc describe -n openshift-ingress service router-internal-custom
Name:              router-internal-custom
Namespace:         openshift-ingress
Labels:            ingresscontroller.operator.openshift.io/owning-ingresscontroller=custom
Annotations:       service.alpha.openshift.io/serving-cert-secret-name: router-metrics-certs-custom
                   service.alpha.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1602996798
                   service.beta.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1602996798
Selector:          ingresscontroller.operator.openshift.io/deployment-ingresscontroller=custom
Type:              ClusterIP
IP:                172.30.71.122
Port:              http  80/TCP
TargetPort:        http/TCP
Endpoints:         10.131.0.120:80
Port:              https  443/TCP
TargetPort:        https/TCP
Endpoints:         10.131.0.120:443
Port:              metrics  1936/TCP
TargetPort:        1936/TCP
Endpoints:         10.131.0.120:1936
Session Affinity:  None
```

If we check the host subnets of the workers we noticed that the Pod Network is the 10.131.0.0 that
is the worker1 where the router-internal-custom pod is.

```
$ oc get hostsubnet | grep worker
worker-0.sharedocp4upi43.lab.pnq2.cee.redhat.com   worker-0.sharedocp4upi45.lab.pnq2.cee.redhat.com 10.74.178.241   10.130.0.0/23
worker-1.sharedocp4upi43.lab.pnq2.cee.redhat.com   worker-1.sharedocp4upi43.lab.pnq2.cee.redhat.com 10.74.181.41    10.131.0.0/23
```

* On the other hand if we check the NodePort Service generated for the custom IngressController:

```
$ oc get svc -n openshift-ingress router-nodeport-custom -o yaml | yq .spec
{
  "clusterIP": "172.30.1.17",
  "externalTrafficPolicy": "Local",
  "ports": [
    {
      "name": "http",
      "nodePort": 32596,
      "port": 80,
      "protocol": "TCP",
      "targetPort": "http"
    },
    {
      "name": "https",
      "nodePort": 32121,
      "port": 443,
      "protocol": "TCP",
      "targetPort": "https"
    }
  ],
  "selector": {
    "ingresscontroller.operator.openshift.io/deployment-ingresscontroller": "custom"
  },
  "sessionAffinity": "None",
  "type": "NodePort"
}
```

We can check that the nodeport has opened a random port (from the 30000+) in each Node, and bind it
his the port of HTTP and HTTPS.

And with that you can create several Route Shardings in the same Openshift Nodes, avoiding to have
specific nodes only for your IngressController (if you want obviously :D), spending less resources.

Stay tuned to the new blog post!

Happy Openshifting!

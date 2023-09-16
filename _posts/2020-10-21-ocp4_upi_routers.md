---
layout: single
title: Deep Dive of OpenShift 4 Routers in On-Premise deployments
date: 2020-10-21
type: post
published: true
status: publish
categories:
- OpenShift
- Networking
- Administration
- Kubernetes
tags: []
author: rcarrata
comments: true
---

What's the difference between the Routers / IngressController of OpenShift 4 in cloud environments and the deployments in On-premise? What are the main components involved?

Let's take a look!

### Overview and Example Scenario

This example is applicable in all UPI deployments in On-premise (Baremetal, VMWare UPI, RHV UPI,
OSP UPI, etc) and also the RHV IPI, VMWARE IPI and Baremetal IPI (with some differences that will be
analyzed in next blog posts) of OpenShift 4.

In this example a OpenShift 4.5 UPI on top of Openstack is used.

### Ingresscontrollers in On-premise deployments

By default when a UPI OpenShift 4 is installed a default ingresscontroller is configured. Let's take
a look:

```
$ oc get ingresscontroller -n openshift-ingress-operator
NAME      AGE
default   3d16h
```

If we check the endpointPublishingStrategy in the definition of the ingresscontroller we can see:

```
$ oc get ingresscontrollers.operator.openshift.io -n openshift-ingress-operator default -o yaml | yq .status.endpointPublishingStrategy
{
  "type": "HostNetwork"
}
```

As you noticed, the endpoint publishing strategy is configured as type: HostNetwork.

The HostNetwork endpoint publishing strategy publishes the Ingress Controller on node ports where the Ingress Controller is deployed.

An Ingress controller with the HostNetwork endpoint publishing strategy can have only one Pod replica per node.

If you want n replicas, you must use at least n nodes where those replicas can be scheduled. Because each Pod replica requests ports 80 and 443 on the node host where it is scheduled, a replica cannot be scheduled to a node if another Pod on the same node is using those ports.

* HostNetwork spec

Interesting right? But what is the HostNetwork and to what applies?

The hostNetwork setting applies to the Kubernetes pods. When a pod is configured with hostNetwork: true, the applications running in such a pod can directly see the network interfaces of the host machine where the pod was started. An application that is configured to listen on all network interfaces will in turn be accessible on all network interfaces of the host machine.

Creating a pod with ```hostNetwork: true``` on OpenShift is a privileged operation. For these reasons, the host networking is not a good way to make your applications accessible from outside of the cluster.

What is the host networking good for? For cases where a direct access to the host networking is required. As we have in our UPI installation of OpenShift, where the pods of the HAproxy need to access to the host machine interfaces for expose the HAproxy ports in order to LoadBalance them.

Let's check the OpenShift Routers created by the ingresscontroller then.

### OpenShift Routers in UPI deployment

First check the pods of the routers deployed:

```
$ oc get pod -n openshift-ingress -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE                                                   NOMINATED NODE   READINESS GATES
router-default-754bf5f974-62ntm   1/1     Running   0          41h   10.0.92.117   worker-0.sharedocp4upi45.lab.upshift.rdu2.redhat.com   <none>           <none>
router-default-754bf5f974-qqmx5   1/1     Running   0          41h   10.0.93.214   worker-1.sharedocp4upi45.lab.upshift.rdu2.redhat.com   <none>           <none>
```

Also, look at that the IP is from the hostsubnet and not from the pod network as by default is:

```
$ oc get clusternetworks
NAME      CLUSTER NETWORK   SERVICE NETWORK   PLUGIN NAME
default   10.128.0.0/14     172.30.0.0/16     redhat/openshift-ovs-networkpolicy

$ oc get hostsubnets worker-0.sharedocp4upi45.lab.upshift.rdu2.redhat.com
NAME                                                   HOST                                                   HOST IP       SUBNET          EGRESS CIDRS   EGRESS IPS
worker-0.sharedocp4upi45.lab.upshift.rdu2.redhat.com   worker-0.sharedocp4upi45.lab.upshift.rdu2.redhat.com   10.0.92.117   10.128.2.0/23
```

Let's inspect in detail one of the instances of the OpenShift routers:

```
$ oc get pod router-default-754bf5f974-62ntm -n openshift-ingress -o json | jq -r '.spec.hostNetwork'
true
```

As we can see the hostNetwork is enabled as we expected.

Also more interesting things in the routers yaml definition are:

```
$ oc get pod router-default-754bf5f974-62ntm -n openshift-ingress -o json | jq -r '.spec.containers[].ports'
[
  {
    "containerPort": 80,
    "hostPort": 80,
    "name": "http",
    "protocol": "TCP"
  },
  {
    "containerPort": 443,
    "hostPort": 443,
    "name": "https",
    "protocol": "TCP"
  },
  {
    "containerPort": 1936,
    "hostPort": 1936,
    "name": "metrics",
    "protocol": "TCP"
  }
]
```

The hostPort setting applies to the Kubernetes containers. The container port will be exposed to the external network at <hostIP>:<hostPort>, where the hostIP is the IP address of the Kubernetes node where the container is running and the hostPort is the port requested by the user.
So, the hostPort feature allows to expose a single container port on the host IP.

* HostPort spec

What is the hostPort used for? As we checked the OpenShift routers are deployed as a set of containers running on top of our OpenShift cluster. These containers are configured to use hostPorts 80 and 443 to allow the inbound traffic on these ports from the outside of the OpenShift cluster (from an external Loadbalancer, physical as f5 or a virtual as Nginx)

For this reason each OpenShift router pods are located in different OpenShift nodes, because it can not be overlapped due to the hostPort as we mentioned before.

If we check inside the pod of the OpenShift Routers (haproxy):

```
$ oc exec -ti router-default-754bf5f974-62ntm -n openshift-ingress bash
bash-4.2$ ss -laputen | grep LISTEN | grep 443
tcp    LISTEN     0      128       *:443                   *:*                   uid:1000560000 ino:53555 sk:14 <->

bash-4.2$ ip ad | grep veth
11: vethb4826b01@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group default
14: veth5dc748f8@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group default
81: veth2190afa3@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group default
82: veth8b6082e1@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group default
84: vethf61d7613@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master ovs-system state UP group default
...
```

As we can see they can see every interface because is a privileged pod with the hostnetwork: true

Furthermore, as we check the services we have a ClusterIP svc that have as backends the OpenShift router pods:

```
$ oc get svc -n openshift-ingress
NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                   AGE
router-internal-default   ClusterIP   172.30.109.20   <none>        80/TCP,443/TCP,1936/TCP   3d14h
```

As we can see the ports are 80/443/1936 for http, https, and metrics:

```
$ oc get svc router-internal-default -o json | jq .spec.ports
[
  {
    "name": "http",
    "port": 80,
    "protocol": "TCP",
    "targetPort": "http"
  },
  {
    "name": "https",
    "port": 443,
    "protocol": "TCP",
    "targetPort": "https"
  },
  {
    "name": "metrics",
    "port": 1936,
    "protocol": "TCP",
    "targetPort": 1936
  }
]
```

And the backends are the two replicas of the pods of the routers:

```
$ oc describe svc router-internal-default
Name:              router-internal-default
Namespace:         openshift-ingress
Labels:            ingresscontroller.operator.openshift.io/owning-ingresscontroller=default
Annotations:       service.alpha.openshift.io/serving-cert-secret-name: router-metrics-certs-default
                   service.alpha.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1602996798
                   service.beta.openshift.io/serving-cert-signed-by: openshift-service-serving-signer@1602996798
Selector:          ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
Type:              ClusterIP
IP:                172.30.109.20
Port:              http  80/TCP
TargetPort:        http/TCP
Endpoints:         10.0.92.117:80,10.0.93.214:80
Port:              https  443/TCP
TargetPort:        https/TCP
Endpoints:         10.0.92.117:443,10.0.93.214:443
Port:              metrics  1936/TCP
TargetPort:        1936/TCP
Endpoints:         10.0.92.117:1936,10.0.93.214:1936
Session Affinity:  None
Events:            <none>
```

And that's it! In the next blog post we will tweak the ingresscontroller to deploy route sharding in
the same hosts, changing the hostNetwork spec.

Stay tuned!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
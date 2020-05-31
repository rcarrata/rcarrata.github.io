---
layout: post
title: Ingress Routing & Traffic Management in Service Mesh
date: 2020-05-26
type: post
published: true
status: publish
categories:
- Istio
tags: []
author: rcarrata
comments: true
---

What is the ingress routing in Service Mesh and what components are involved? And which are the main
differences between Openshift Routes and Ingress routing in Service Mesh?

Let's Mesh in!!

This is the third blog post of the Service Mesh in Openshift series. Check the earlier post:
* [Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)

## Overview

In this blog post, we will deep dive in the traffic management, ingress routing of Service Mesh and
the components involved for getting traffic into our applications deployed within our Service Mesh.

## 0. Prerequisites

* Openshift 4.x cluster (tested in a 4.3+ cluster)
* Openshift Service Mesh Operators installed
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)
* Microservices added within Service Mesh (follow the third blog post)

## 1. Adapt Existing K8S Services

In Openshift, when a oc new-app is used, several kubernetes resources are created within this
command. One of this service is the Service.

Check out the service of the customer microservice deployed

```
 oc get svc/customer -n $OCP_NS -o json | jq -r '[.spec.selector]'
[
  {
    "app": "customer",
    "deploymentconfig": "customer",
    "version": "v1"
  }
]
```

as we know, kubernetes services match with pods based on the selector we have defined on it. In the
previous example the app, deploymentconfig, and version labels define the match with this pods.

So, this means that only pods matching those labels will be using the loadbalancing of this specific
service (be 'behind' the service).

But in Service Mesh, we need to define the Service in a bit wider selector, because we will
differenciate the specific versions with the Destination Rules that will describe in next sections.

An example that of a new selector is the following:

```
cat istio-files/customer/kubernetes/Service.yml | grep selector -A5
 selector:
   app: customer
```

this selector will match any pod with the labelled with app: customer, and this possibility includes
several versions of the same application.

Run the specific commands to delete the existing oc new-app services and apply the service adapted
to istio:

```
$ oc delete svc/customer -n $OCP_NS
service "customer" deleted

$ oc delete svc/preference -n $OCP_NS
service "preference" deleted

$ oc delete svc/recommendation -n $OCP_NS
service "recommendation" deleted

$ oc delete svc/partner -n $OCP_NS
service "partner" deleted
```

```
$ oc apply -f istio-files/customer/kubernetes/Service.yml -n $OCP_NS
service/customer created

$ oc apply -f istio-files/preference/kub
ernetes/Service.yml -n $OCP_NS service/preference created

$ oc apply -f istio-files/recommendation/kubernetes/Service.yml -n $OCP_NS
service/recommendation created

$ oc apply -f istio-files/partner/kubernetes/Service.yml -n $OCP_NS
service/partner created
```

---

## 2. Enabling Ingress Routing

Now that we have the proper Services patches, we need to enable the ingress routing in order to
reach and consume our microservices outside of the Mesh and the Openshift cluster.

### 2.1 Openshift Routes vs Ingress Service Mesh

There is a main difference between the ingress of Openshift Routes and the Ingress Routing with
Service Mesh.

The Openshift Route uses the Openshift Ingresscontroller/Router (Haproxy) for getting the traffic
into the cluster, the router points to the specific service with the labels, and through that
service reaches the specific pods of our applications:

* Openshift Route Ingress by Default:

```
Route (HAProxy/Router) => SVC1 (ip-pod-1, ip-pod-2, ...)
```

So far so good, right? But within the Service Mesh, using this Route will
miss the features and capabilities of Service Mesh in those services which are externally exposed.
This is because some policies and rules are applied on the client proxy.

So in order to take profit to the capabilities of Istio in our ingress routing, we will use of the
following Istio resources:

* Ingress Gateway
* VirtualService
* DestinationRule

We'll explain in a bit this components but for now, in a high level the ingress routing within Mesh
will be the following:

* Ingress Routing within Service Mesh

```
Route (HAProxy/Router) => istio-ingressgateway  => SVC1 (ip-pod-1, ip-pod-2, ...)
```

So gor getting the traffic into the cluster and into the Service Mesh a ingress gateway will be
used, but instead of accessing through the same ingress gateway (and using several paths as
/customer or /partner) we're using a Route that points to the istio ingress gateway.

After the definition of the ingress gateway and using the VirtualService, the requests will be
routed through this ingress gateway to the services of k8s and therefore will be routed finally to
our Pods.

Seems not quite easy, but in fact it's more straightforward that this looks like. Let's take a look
of the components used in this process.

### 2.2 Components of Service Mesh for Traffic Management

* A **virtual service** lets you configure how requests are routed to a service within an Istio service mesh, building on the basic connectivity and discovery provided by Istio and your platform.

* Example1 of VirtualService using a gateway (customer-gw in this case):

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customer
  namespace: ${NAMESPACE}
spec:
  gateways:
  - customer-gw
  hosts:
  - customer-${NAMESPACE}-istio-system.${APP_SUBDOMAIN}
  http:
  - route:
    - destination:
        host: customer
        subset: version-v1
      weight: 100
```

* Example2 of VirtualService using User headers to segregate traffic:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

* **Destination Rule** are applied after virtual service routing rules are evaluated, so they apply to the traffic’s “real” destination.

You use destination rules to specify named service subsets, such as grouping all a given service’s instances by version. You can then use these service subsets in the routing rules of virtual services to control the traffic to different instances of your services.

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

In fact, **think of virtual services as how you route your traffic to a given destination**, and then you use **destination rules to configure what happens to traffic for that destination**.

* Gateway: You use Gateways to manage inbound and outbound traffic for your mesh, letting you specify which traffic you want to enter or leave the mesh.

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

### 2.3 Creating service mesh objects for our microservices

To expose our microservices with an ingress routing, we need to create several objects, some of them reviewed in the previous section.

Be clear that after to apply this istio objects, the old routes does not work anymore, and the new
route will be no longer in the namespace of our microservices, every route and ingress gateway will
be located in the istio-system namespace.

So in our case the objects that will be applied will be the following:

```
Route (HAProxy/Router) => istio-ingressgateway  => SVC1 (ip-pod-1, ip-pod-2, ...)
```

* Route (ns => istio-system)
* Gateway (ns => istio-tutorial)
* VirtualService (ns => istio-tutorial)
* DestinationRule (ns => istio-tutorial) - Enable MTLS

Render and apply the objects for enable the ingress routing:

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $APP_SUBDOMAIN
apps.ocp4.rglab.com
export OCP_NS="istio-tutorial"
```

```
$ cat istio-files/customer-ingress_mtls.yml | NAMESPACE=$(echo $OCP_NS) envsubst | oc apply -f -
route.route.openshift.io/customer created
gateway.networking.istio.io/customer-gw created
virtualservice.networking.istio.io/customer created
destinationrule.networking.istio.io/customer created
```

### 2.4 Let's dig in in the ingress routing objects:

First the route:

```
$ oc get route -n istio-system customer -o yaml --export
...
spec:
  host: customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
  port:
    targetPort: http2
  to:
    kind: Service
    name: istio-ingressgateway
    weight: 100
  wildcardPolicy: None
status:
  ingress: null
```

as we can see the route points to the service of the istio-ingressgateway using the host of the
<app><app-ns><istio-ns>.apps and the port for http2.

For the Gateway, the customer-gw is defined into the namespace of our apps:

```
$ oc get gateway -n istio-tutorial customer-gw -o yaml | yq .spec
{
  "selector": {
    "istio": "ingressgateway"
  },
  "servers": [
    {
      "hosts": [
        "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com"
      ],
      "port": {
        "name": "http2",
        "number": 80,
        "protocol": "HTTP2"
      }
    }
  ]
}
```

as we can see the selector have istio: ingressgateway, defining that is linked with the istio ingressgateway, and have the host and the port defined also in the route.

```
$ oc get virtualservice -n istio-tutorial customer -o yaml | yq .spec
{
  "gateways": [
    "customer-gw"
  ],
  "hosts": [
    "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com"
  ],
  "http": [
      "route": [
        {
          "destination": {
            "host": "customer",
            "subset": "version-v1"
          }
        }
      ]
    }
  },
}
```

The virtualservice uses points to the gateway of customer-gw, and sets the route request to the subset of the host of customer host and with the subset of version-v1.

Finally the DestinationRule is defined as:

```
$ oc get destinationrule -n istio-tutorial customer -o yaml | yq .spec
{
  "host": "customer",
  "subsets": [
    {
      "labels": {
        "version": "v1"
      },
      "name": "version-v1"
    },
  "trafficPolicy": {
    "tls": {
      "mode": "ISTIO_MUTUAL"
    }
  }
}
```

the destination rule sets the subset of versions (actually version-v1 but in the next labs will be expanding) that we have in place. Also the host that belongs this destinationrule, and an important feature: enable the Mutual TLS.

Another blog post will be dedicated to analyse in deep the mutual TLS, but for know check that with this traffic policy the MTLS is enabled.

## 3. Testing the ingress routing of our microservices in Service Mesh

Check the routes generated in the istio-system namespace:

```
$ oc get route -n istio-system customer
NAME       HOST/PORT                                                  PATH   SERVICES               PORT    TERMINATION   WILDCARD
customer   customer-istio-tutorial-istio-system.apps.ocp4.rglab.com          istio-ingressgateway   http2                 None
```

Curl them to test that it's everything is ok:

```
$ curl customer-istio-tutorial-istio-system.apps.ocp4.rglab.com -I
HTTP/1.1 200 OK
content-type: text/plain;charset=UTF-8
content-length: 0
```

It works!

IMPORTANT: Old routes does not work anymore, use all new routes under istio-sytem:

```
$ oc get route -n istio-tutorial customer
NAME       HOST/PORT                                     PATH   SERVICES   PORT       TERMINATION   WILDCARD
customer   customer-istio-tutorial.apps.ocp4.rglab.com          customer   8080-tcp                 None

$ curl -I customer-istio-tutorial.apps.ocp4.rglab.com
HTTP/1.0 503 Service Unavailable
```

## 4. Apply the istio routing to the partner microservice

Now that we're aware that everything is working with customer microservice ingress routing, let's apply this to the partner microservice:

```
$ cat istio-files/partner-ingress_mtls.yml | NAMESPACE=$(echo $OCP_NS) envsubst | oc apply -f -
route.route.openshift.io/partner created
gateway.networking.istio.io/partner-gw created
virtualservice.networking.istio.io/partner created
destinationrule.networking.istio.io/partner created
```

Test it in order to know if everything is ok:

```
$ curl -I partner-istio-tutorial-istio-system.apps.ocp4.rglab.com
HTTP/1.1 200 OK
x-application-context: application
content-type: text/plain;charset=UTF-8
content-length: 76

$ curl partner-istio-tutorial-istio-system.apps.ocp4.rglab.com
partner => preference => recommendation v1 from 'recommendation-2-kjpsc': 9
```

So now, we have in place the two routes for our ingress routing system:

```
$ oc get routes -n istio-system | egrep "customer|partner"
customer               customer-istio-tutorial-istio-system.apps.ocp4.rglab.com          istio-ingressgateway   http2                        None
partner                partner-istio-tutorial-istio-system.apps.ocp4.rglab.com           istio-ingressgateway   http2                        None
```

Happy Service Meshing!

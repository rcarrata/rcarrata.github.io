---
layout: post
title: Canary deployments in Openshift Service Mesh
date: 2020-06-02
type: post
published: true
status: publish
categories:
- Istio
tags: []
author: rcarrata
comments: true
---

Whats a Canary deployment and how to configure in Service Mesh? What are the benefits of a Canary deployment?
And how we can give more intelligence to our routes and requests inside of our mesh?

Let's Mesh in!!

This is the sixth blog post of the Service Mesh in Openshift series. Check the earlier posts in:

* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [II - Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)
* [III - Including microservices in Service Mesh](https://rcarrata.com/istio/adding-microservices-within-mesh/)
* [IV - Ingress Routing & Traffic Management in Service Mesh](https://rcarrata.com/istio/ingress-routing-service-mesh/)
* [V - Blue Green Deployments in Service Mesh](https://rcarrata.com/istio/blue-green-in-service-mesh/)

## Overview

When introducing new versions of a service, it is often desirable to shift a controlled percentage of user traffic to a newer version of the service in the process of phasing out the older version. This technique is called a canary deployment.

In this blog post we will analyse the Canary Deployments and how could be implemented in our Service Mesh.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 0. Prerequisites

* Openshift 4.x cluster (tested in a 4.3+ cluster)
* Openshift Service Mesh Operators installed
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)
* Microservices added within Service Mesh (follow the third blog post)
* Including microservices in Service Mesh (follow the fourth blog post)

Export this environment variables to identify your cluster and namespace:

```
$ export OCP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $OCP_SUBDOMAIN
apps.ocp4.rglab.com
export OCP_NS="istio-tutorial"
```

## 1. Deploy customer v2 microservice

In this case, we will use one of the external service to set up a Canary to version v2 based on the content of an http header: user

```
oc new-app -l app=customer,version=v2 --name=customer-v2 --docker-image=quay.io/rcarrata/customer:quarkus -e VERSION=v2 -e  JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -n $OCP_NS

oc delete svc/customer-v2 -n $OCP_NS

oc patch dc/customer-v2 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS
```

### 2. Configure Canary rules based in http headers

The canary to version v2 of customer, will be based in a http header of user.

For implementing this, we will use a DestinationRule and a VirtualService:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customer
  namespace: ${NAMESPACE}
  labels:
    group: tutorial
spec:
  gateways:
  - customer-gw
  hosts:
  - customer-${NAMESPACE}-istio-system.${APP_SUBDOMAIN}
  http:
  - match:
    - headers:
        user:
          exact: rober
    route:
    - destination:
        host: customer
        subset: version-v2
  - route:
    - destination:
        host: customer
        subset: version-v1
```

In the VirtualService is the key of this configuration, as you can see the http header of user that
need to match to rober.

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customer
  namespace: ${NAMESPACE}
  labels:
    group: tutorial
spec:
  host: customer
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - labels:
      version: v1
    name: version-v1
  - labels:
      version: v2
    name: version-v2
```

The DestinationRule is the same that we implemented in the exercises before, with two versions in
the subsets to the customer (v1 and v2).

Let's apply and try different headers to our customer microservice:

```
$ cat istio-files/customer-user-header_mtls .yml | NAMESPACE=$OCP_NS envsubst | oc apply -f -
 virtualservice.networking.istio.io/customer configured
 destinationrule.networking.istio.io/customer configured
```

Test the customer route with the user http header that we customized (in our case user:rober):

```
$ curl -v -Huser:rober customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
*   Trying 10.1.8.85:80...
* TCP_NODELAY set
* Connected to customer-istio-tutorial-istio-system.apps.ocp4.rglab.com (10.1.8.85) port 80 (#0)
> GET / HTTP/1.1
> Host: customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
> User-Agent: curl/7.65.3
> Accept: */*
> user:rober
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< content-type: text/plain;charset=UTF-8
< content-length: 67
< date: Sat, 16 May 2020 17:08:19 GMT
< x-envoy-upstream-service-time: 1125
< server: istio-envoy
< Set-Cookie: a16fff5a7437b33e5338c8809926371f=95a3920d95573fbd4abdcc889cf1c1f9; path=/; HttpOnly
< Cache-control: private
<
customer v2 => preference => recommendation v2 from '2-nqq9f': 325
* Connection #0 to host customer-istio-tutorial-istio-system.apps.ocp4.rglab.com left intact
```

As we can see the version that is reached is customer v2, because of the user:rober that is applied
in the virtualservice in the step earlier:

```
$ oc get vs customer -o yaml | grep -m2 -A8 http | tail -n9
 http:
 - match:
   - headers:
       user:
         exact: rober
   route:
   - destination:
       host: customer
       subset: version-v2
```

Now we will execute a request without the user header, or with a different user than we used before:

```
$ curl customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
customer v1 => preference => recommendation v1 from 'recommendation-2-kjpsc': 667

$ curl -Huser:johndoe customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
customer v1 => preference => recommendation v2 from '2-nqq9f': 326
```

as we can see the request is routed to customerv1, because does not match the exact user header that we defined in the VirtualService:

```
$ oc get vs customer -o yaml | grep -m2 -A15 http | tail -n4
  - route:
    - destination:
        host: customer
        subset: version-v1
```

## 3. Configure Canary deployments based in User-Agent Headers

User agents are unique to every visitor on the web. They reveal a catalog of technical data about
the device and software that the visitor is using.

When your browser (or similar device) loads a website, it identifies itself as an agent when it
retrieves the content you’ve requested.

Along with that user-agent identification, the browser sends a host of information about the device
and network that it’s on.

In this case, we want to generate a Canary deployment based in User-Agent Headers, so in the
VirtualService for the recommendation we will have:

```
  http:
  - match:
    - headers:
        baggage-user-agent:
          regex: .*Safari.*
    route:
    - destination:
        host: recommendation
        subset: version-v2
  - route:
    - destination:
        host: recommendation
        subset: version-v1
```

as we see in this example, we will use the baggage-user-agent to define the user agent for Safari
browsers.

Note: the "user-agent" header is added to OpenTracing baggage in the Customer service. From there it
is automatically propagated to all downstream services. To enable automatic baggage propagation all
intermediate services have to be instrumented with OpenTracing. The baggage header for user agent
has following form baggage-user-agent: <value>.

Let's apply the VirtualService and the DestinationRule (a default one with mtls and v1 / v2 subsets):

```
$ oc apply -f istio-files/recommendation-safari-user-header.yml
```

Once is applied, check the recommendation vs in order to check the headers:

```
$ oc get vs recommendation -o yaml | yq .spec.http
[
  {
    "match": [
      {
        "headers": {
          "baggage-user-agent": {
            "regex": ".*Safari.*"
          }
        }
      }
    ],
    "route": [
      {
        "destination": {
          "host": "recommendation",
          "subset": "version-v2"
        }
      }
    ]
  },
  {
    "route": [
      {
        "destination": {
          "host": "recommendation",
          "subset": "version-v1"
        }
      }
    ]
  }
]
```

So with this match headers, when a request is originated from a Safari browser, will be hit the
version-v2 of recommendation, instead if a non-Safari user is requesting our app, the request will be
routed to the version-v1:

```
customer_route=$(oc get route -n istio-system customer | tail -n1 | awk '{ print $2 }')

./test.sh $customer_route
Request Number 0:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 31

Request Number 1:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 32

Request Number 2:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 33
```

as we see if we request a nonSafari request, the request is routed to the recommendation v1.

```
 curl -A Safari $customer_route -v
* Connected to customer-istio-tutorial-istio-system.apps.ocp4.rglab.com (10.1.8.85) port 80 (#0)
> GET / HTTP/1.1
> Host: customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
> User-Agent: Safari
> Accept: */*

< HTTP/1.1 200 OK
< x-envoy-upstream-service-time: 27
< server: istio-envoy
<
customer v1 => preference => recommendation v2 from '2-mzkn6': 25
```

if you request the customer route, with the header Safari (or in a Safari browser :D), the request is routed to the
recommendation v2. As we can check, the User-Agent is passed in the curl command and returns an 200 OK from our app.

Into the Jaeger (Opentracing), we can see the header also of Safari in the request, that is propagated through the different microservices that
are belonging to our app:

[![](/images/istio4.png "Istio Canary Safari")]({{site.url}}/images/istio4.png)

### Mobile User-Agents

With the same User Agent agent (represented as OpenTracing baggage in the customer service), we can
diffentiate the traffic when is requested from a Mobile Phone or a Desktop browser into the VirtualService:

```
$ oc get vs recommendation -o yaml | yq .spec.http
[
  {
    "match": [
      {
        "headers": {
          "baggage-user-agent": {
            "regex": ".*Mobile.*"
          }
        }
      }
    ],
    "route": [
      {
        "destination": {
          "host": "recommendation",
          "subset": "version-v2"
        }
      }
    ]
  },
  {
    "route": [
      {
        "destination": {
          "host": "recommendation",
          "subset": "version-v1"
        }
      }
    ]
  }
]
```

So, when a regular curl is performed this request is routed to the version-v1 of recommendation:

```
./istio-files/test.sh $customer_route
Request Number 0:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 593

Request Number 1:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 594

Request Number 2:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 595

Request Number 3:
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 596
```

But if we force the request simutaling that is from a phone, the request will be routed to the recommendation v2.

For simulate the Mobile phone, the -A header could be used with this header:

```
"Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5"
```

Executing the request to the customer, we see that is routed to the recommendation v2:

```
curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" $customer_route
customer v1 => preference => recommendation v2 from '2-mzkn6': 44
```

As we can see in Kiali (if we execute more curls with the headers of Mobile), the requests hits into the recommendation v2:

[![](/images/istio6.png "Istio Canary Mobile")]({{site.url}}/images/istio6.png)

And that all for this blog post!

Happy Meshing!

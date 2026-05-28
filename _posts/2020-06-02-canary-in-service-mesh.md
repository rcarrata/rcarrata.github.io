---
layout: single
title: Canary deployments in OpenShift Service Mesh
date: 2020-06-02
type: post
published: true
status: publish
categories:
- Istio
tags: ["Kubernetes", "security", "Administration", "OpenShift", "Networking", "Istio"]
author: rcarrata
comments: true
---

What's a Canary deployment and how do you configure it in Service Mesh? What are the benefits of a Canary deployment?
And how can we give more intelligence to our routes and requests inside our mesh?

Let's Mesh in!!

This is the sixth blog post of the Service Mesh in OpenShift series. Check the earlier posts in:

* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [II - Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)
* [III - Including microservices in Service Mesh](https://rcarrata.com/istio/adding-microservices-within-mesh/)
* [IV - Ingress Routing & Traffic Management in Service Mesh](https://rcarrata.com/istio/ingress-routing-service-mesh/)
* [V - Blue Green Deployments in Service Mesh](https://rcarrata.com/istio/blue-green-in-service-mesh/)

## Overview

When introducing new versions of a service, it is often desirable to shift a controlled percentage of user traffic to a newer version of the service in the process of phasing out the older version. This technique is called a canary deployment.

In this blog post we will analyse Canary Deployments and how they can be implemented in our Service Mesh.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 0. Prerequisites

* OpenShift 4.x cluster (tested in a 4.3+ cluster)
* OpenShift Service Mesh Operators installed (v1.1.1 in these blog posts)
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)
* Microservices added within Service Mesh (follow the third blog post)
* Including microservices in Service Mesh (follow the fourth blog post)

Export these environment variables to identify your cluster and namespace:

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $APP_SUBDOMAIN
apps.ocp4.rglab.com
export NAMESPACE="istio-tutorial"
```

## 1. Deploy customer v2 microservice

In this case, we will use one of the external services to set up a Canary to version v2 based on the content of an HTTP header: user

```
oc new-app -l app=customer,version=v2 --name=customer-v2 --docker-image=quay.io/rcarrata/customer:quarkus -e VERSION=v2 -e  JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -n $OCP_NS

oc delete svc/customer-v2 -n $OCP_NS

oc patch dc/customer-v2 -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS
```

## 2. Configure Canary rules based on HTTP headers

The canary to version v2 of customer will be based on an HTTP header of user.

To implement this, we will use a DestinationRule and a VirtualService:

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

The VirtualService is the key of this configuration, as you can see the HTTP header of user that
needs to match to rober.

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

The DestinationRule is the same one that we implemented in the previous exercises, with two versions in
the subsets for the customer (v1 and v2).

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

As we can see, the version that is reached is customer v2, because of the user:rober header that is applied
in the VirtualService from the earlier step:

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

Now we will execute a request without the user header, or with a different user than the one we used before:

```
$ curl customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
customer v1 => preference => recommendation v1 from 'recommendation-2-kjpsc': 667

$ curl -Huser:johndoe customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
customer v1 => preference => recommendation v2 from '2-nqq9f': 326
```

As we can see, the request is routed to customer v1, because it does not match the exact user header that we defined in the VirtualService:

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

### 3.1 Browser Mobile Agent

When your browser (or similar device) loads a website, it identifies itself as an agent when it
retrieves the content you’ve requested.

Along with that user-agent identification, the browser sends a host of information about the device
and network that it’s on.

In this case, we want to generate a Canary deployment based on User-Agent Headers, so in the
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

As we see in this example, we will use the baggage-user-agent to define the user agent for Safari
browsers.

Note: the "user-agent" header is added to OpenTracing baggage in the Customer service. From there it
is automatically propagated to all downstream services. To enable automatic baggage propagation all
intermediate services have to be instrumented with OpenTracing. The baggage header for user agent
has the following form baggage-user-agent: <value>.

Let's apply the VirtualService and the DestinationRule (a default one with mtls and v1 / v2 subsets):

```
$ oc apply -f istio-files/recommendation-safari-user-header.yml
```

Once it is applied, check the recommendation VirtualService to verify the headers:

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

So with these match headers, when a request originates from a Safari browser, it will hit the
version-v2 of recommendation. If a non-Safari user is requesting our app, the request will be
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

As we see, if we send a non-Safari request, it is routed to recommendation v1.

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

If you request the customer route with the Safari header (or in a Safari browser :D), the request is routed to
recommendation v2. As we can see, the User-Agent is passed in the curl command and returns a 200 OK from our app.

In Jaeger (OpenTracing), we can also see the Safari header in the request, which is propagated through the different microservices that
belong to our app:

[![](/images/istio4.png "Istio Canary Safari")]({{site.url}}/images/istio4.png)

### 3.2 Mobile User-Agents

With the same User Agent (represented as OpenTracing baggage in the customer service), we can
differentiate the traffic when it is requested from a mobile phone or a desktop browser in the VirtualService:

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

But if we force the request simulating that it is from a phone, the request will be routed to recommendation v2.

To simulate a mobile phone, the -A header can be used with this value:

```
"Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5"
```

Executing the request to the customer, we see that it is routed to recommendation v2:

```
curl -A "Mozilla/5.0 (iPhone; U; CPU iPhone OS 4(KHTML, like Gecko) Version/5.0.2 Mobile/8J2 Safari/6533.18.5" $customer_route
customer v1 => preference => recommendation v2 from '2-mzkn6': 44
```

As we can see in Kiali (if we execute more curls with the Mobile headers), the requests hit recommendation v2:

[![](/images/istio6.png "Istio Canary Mobile")]({{site.url}}/images/istio6.png)

And that's all for this blog post!

Check out the part seven of this blog series in [Traffic Mirroring in Service Mesh](https://rcarrata.com/istio/traffic-mirroring-in-service-mesh/)

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy Meshing!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
---
layout: post
title: Traffic Mirroring in Openshift Service Mesh
date: 2020-06-05
type: post
published: true
status: publish
categories:
- Istio
tags: []
author: rcarrata
comments: true
---

How to do testing cases with live production in a safe, secure and predicted mode? How can achieve
this within our Service Mesh? And what's the traffic mirroring and how can help this use case?

Let's Mesh in!!

This is the seventh blog post of the Service Mesh in Openshift series. Check the earlier posts in:

* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [II - Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)
* [III - Including microservices in Service Mesh](https://rcarrata.com/istio/adding-microservices-within-mesh/)
* [IV - Ingress Routing & Traffic Management in Service Mesh](https://rcarrata.com/istio/ingress-routing-service-mesh/)
* [V - Blue Green Deployments in Service Mesh](https://rcarrata.com/istio/blue-green-in-service-mesh/)
* [VI - Canary Deployments in Service Mesh](https://rcarrata.com/istio/traffic-mirroring-in-service-mesh/)

## Overview

Trying to enumerate all the possible combinations of test cases for testing services in non-production/test environments can be hard and dangerous.
In some cases, you’ll find that all of the effort that goes into cataloging these use cases doesn’t match up to real production use cases.
In an ideal scenario, we could use live production use cases and traffic to help illuminate all of the feature areas of the service under test that we might miss in more contrived testing environments.

In this blog post we will analyse the Traffic Mirroring Deployments and how could be implemented in our Service Mesh.

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## Prerequisites

* Openshift 4.x cluster (tested in a 4.3+ cluster)
* Openshift Service Mesh Operators installed (v1.1.1 in these blog posts)
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)
* Microservices added within Service Mesh (follow the third blog post)
* Including microservices in Service Mesh (follow the fourth blog post)

Export this environment variables to identify your cluster and namespace:

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $OCP_SUBDOMAIN
apps.ocp4.rglab.com
export NAMESPACE="istio-tutorial"
```

## 1. Traffic Mirroring

Traffic mirroring, also called shadowing, is a powerful concept that allows feature teams to bring changes to production with as little risk as possible. Mirroring sends a copy of live traffic to a mirrored service. The mirrored traffic happens out of band of the critical request path for the primary service.

For that, we may have the approach mirroring traffic in the VirtualService from customer v1 to customer v2.

```
http:
- route:
  - destination:
      host: customer
      subset: version-v1
  mirror:
    host: customer
    subset: version-v2
```

Render and apply the traffic mirror virtualservice

```
$ cat istio-files/customer-mirror-traffic.yml | envsubst | oc apply -f -
Warning: oc apply should be used on resource created by either oc create --save-config or oc apply
virtualservice.networking.istio.io/customer configured
destinationrule.networking.istio.io/customer unchanged
```

Make a request to the customer route:

```
$ echo $customer_route
customer-istio-tutorial-istio-system.apps.ocp4.rglab.com

$ curl $customer_route
customer v1 => preference => recommendation v1 from 'recommendation-2-qzptw': 608
```

as we expected the route will be routed from the ingressgateway to the customer v1, but something interesting is happening also behind the carpet.

Check the istio-proxy logs to see what happened with the mirrored traffic:

```
$ oc logs -f customer-v2-2-d8z45 -c istio-proxy
...
[2020-06-05T10:59:15.857Z] "GET / HTTP/1.1" 200 - "-" "-" 0 69 46 43 "192.168.7.77,10.254.3.1,10.254.3.16" "curl/7.69.1" "aa394a0c-464f-9cf0-a987-592fa0d2b0a0" "customer-istio-tutorial-istio-system.apps.ocp4.rglab.com-shadow" "127.0.0.1:8080" inbound|8080|http|customer.istio-tutorial.svc.cluster.local - 10.254.3.24:8080 10.254.3.16:0 outbound_.8080_.version-v2_.customer.istio-tutorial.svc.cluster.local default
```

noticed that the request is also reaching the customerv2 but the response is not sent to the client (curl in this case). This is because have the flag of "-shadow" in the host requested.

So in conclusion, this route rule sends 100% of the traffic to v1. The last stanza specifies that you want to mirror
to the customer:v2 service. When traffic gets mirrored, the requests are sent to the mirrored service
with their Host/Authority headers appended with -shadow. For example, cluster-1 becomes
cluster-1-shadow.

Also, it is important to note that these requests are mirrored as “fire and forget”, which means that the responses are discarded.

In kiali this mirror traffic is showed because, the customer v1 shows that is requested from the
client, but the customerv2 is also sending traffic to preference service, because is producing this traffic mirror

[![](/images/istio7.png "Istio Traffic Mirroring")]({{site.url}}/images/istio7.png)

### Changing the mirror requests

Furthermore, you can use the mirror_percent field to mirror a fraction of the traffic, instead of mirroring all
requests. If this field is absent, for compatibility with older versions, all traffic will be mirrored.

```
http:
- mirror:
    host: customer
    subset: version-v2
  mirror_percent: 50
  route:
  - destination:
      host: customer
      subset: version-v1
    weight: 100
```

Apply the modified mirror_percent with 50% of the mirror_request:

```
$ cat istio-files/customer-mirror-traffic-adv.yml | envsubst | oc apply -f -
virtualservice.networking.istio.io/customer configured
destinationrule.networking.istio.io/customer unchanged
```

And that's it for this blog post!

Happy Meshing!!

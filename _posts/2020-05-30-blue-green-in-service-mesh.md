---
layout: single
title: Blue Green deployments in OpenShift Service Mesh
date: 2020-05-30
type: post
published: true
status: publish
categories:
- Istio

tags: []
author: rcarrata
comments: true
---

Whats a Blue Green deployment and how to configure it our microservices within the Service Mesh? How
change the balancing of our requests in a easy and quick way?

Let's Mesh in!!

This is the fifth blog post of the Service Mesh in OpenShift series. Check the earlier posts in:
* [I - Service Mesh Installation](https://rcarrata.com/istio/service-mesh-installation/)
* [II - Microservices deployment in Service Mesh](https://rcarrata.com/istio/microservices-deployment-in-service-mesh/)
* [III - Including microservices in Service Mesh](https://rcarrata.com/istio/adding-microservices-within-mesh/)
* [IV - Ingress Routing & Traffic Management in Service Mesh](https://rcarrata.com/istio/ingress-routing-service-mesh/)

## Overview

In this blog post, we will deep dive in the traffic management, specifically into the Blue Green
deployments and how to implemented into our Mesh

NOTE: this blog post is supported by the [istio-files repository](https://github.com/rcarrata/istio-files) located in my personal Github

## 0. Prerequisites

* OpenShift 4.x cluster (tested in a 4.3+ cluster)
* OpenShift Service Mesh Operators installed (v1.1.1 in these blog posts)
* Service Mesh Control Plane deployed
* Four microservices deployed (follow the second blog post)
* Microservices added within Service Mesh (follow the third blog post)
* Including microservices in Service Mesh (follow the fourth blog post)

Export this environment variables to identify your cluster and namespace:

```
$ export APP_SUBDOMAIN=$(oc get route -n istio-system | grep -i kiali | awk '{ print $2 }' | cut -f 2- -d '.')
$ echo $APP_SUBDOMAIN
apps.ocp4.rglab.com
export OCP_NS="istio-tutorial"
```

## 1 Blue/Green Deployment

A Blue/Green deployment will allow you to define two (or more) versions of the same application to
receive traffic with zero downtime. This approach, for instance, will let you release a new version
and gradually increment the amount of traffic this version receives.

## 1.1 Deploy recommendation v2

To deploy a new version of the same application, in this case recommendation service, we are going to use exactly the same command oc new-app but we must take into account the following:

* The label app must be the same between different versions of the same app. Remember that we modified the k8s service to use a wider selector based on this label.

* The label version will identify the subset of the new pods created for this DeploymentConfig.

* Both app and version labels are used by Istio and Kiali internally

Deploy recommendation v2:

```
oc new-app -l app=recommendation,version=v2 --name=recommendation-v2 --docker-image=quay.io/rcarrata/recommendation:vertx -e JAVA_OPTIONS='-Xms512m -Xmx512m -Djava.net.preferIPv4Stack=true' -e VERSION=v2 -n $OCP_NS
```

```
oc delete svc/recommendation-v2 -n $OCP_NS
service "recommendation-v2" deleted
```

```
$ oc get pods -n $OCP_NS | grep recommendation-v2
recommendation-v2-1-deploy   0/1     Completed   0          4m5s
recommendation-v2-1-n5t9f    1/1     Running     0          4m1s
```

Apply the istio sidecar injector:

```
 oc patch dc/recommendation-v2 -p '{"spec":{"
template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}' -n $OCP_NS
deploymentconfig.apps.openshift.io/recommendation-v2 patched
```

```
$ oc get pod | grep recommendation-v2
recommendation-v2-1-deploy   0/1     Completed   0          7m
recommendation-v2-2-deploy   0/1     Completed   0          89s
recommendation-v2-2-nqq9f    2/2     Running     0          85s
```

Once your pod has 2/2 containers and it is Running, execute some test either using customer or partner services.

```
$ oc get routes -n istio-system | egrep "customer|partner"
customer               customer-istio-tutorial-istio-system.apps.ocp4.rglab.com          istio-ingressgateway   http2                        None
partner                partner-istio-tutorial-istio-system.apps.ocp4.rglab.com           istio-ingressgateway   http2                        None
```

```
$ curl customer-istio-tutorial-istio-system.apps.ocp4.rglab.com
...
customer v1 => preference => recommendation v1 from 'recommendation-2-kjpsc': 528
customer v1 => preference => recommendation v2 from '2-nqq9f': 1
customer v1 => preference => recommendation v1 from 'recommendation-2-kjpsc': 529
customer v1 => preference => recommendation v2 from '2-nqq9f': 2
customer v1 => preference => recommendation v1 from 'recommendation-2-kjpsc': 530
```

If you check the traffic you can notice that its load balanced with Round Robin algorithm (50/50% to v1 and v2). This is because is not configured any VirtualService or DestinationRule yet.


## 1.2 Applying routing weights in our mesh

Apply the VirtualService with the 25% of the traffic to the recommendation v1 and the 75% to the v2:

```
$ cat istio-files/recommendation-v1_v2_mtls.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: recommendation
  labels:
    group: recommendation
spec:
  hosts:
  - recommendation
  http:
  - route:
    - destination:
        host: recommendation
        subset: v1
      weight: 25
    - destination:
        host: recommendation
        subset: v2
      weight: 75
---
```

Then you need the destination rule to define the subsets that are going to route between:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: recommendation
  labels:
    group: recommendation
spec:
  host: recommendation
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
```

Apply the recommendations:

```
oc apply -f istio-files/recommendation-v1_v2_25_75.mtls.yml
```

Let's generate some load. For doing this we can do it with curls, but with this simple program you can generate this load:

```
$ cat istio-files/test.sh
#!/bin/bash

url=$1
i=0
while :
do
  echo Request Number $i:
  ((i++))
  curl $url
  echo ""
  #sleep 1
done
```

And after that we can check in Kiali that the traffic balanced with 25/75 (approx) to the v1 and v2:

[![](/images/istio3.png "Istio Blue Green Balancing")]({{site.url}}/images/istio3.png)

So, now you can play with your weight and configure different behaviours to control the routing between your different versions of your same repository deployed.

Check out the part six of this blog series in [Canary deployments in Service Mesh](https://rcarrata.com/istio/canary-in-service-mesh/)

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy Service Meshing!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
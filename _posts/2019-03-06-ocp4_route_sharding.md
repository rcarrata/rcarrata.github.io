---
layout: single
title: Deep dive of Route Sharding in OpenShift 4
date: 2019-06-03
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Administration", "OpenShift"]
author: rcarrata
comments: true
---

This blog post aims to provide a guide to implement Route Sharding in OpenShift Container Platform 4 (deployed in AWS), creating multiple routers for particular purposes (for example in this specific case, separating the internal and public/dmz application routes).

## Overview

In OCP, each route can have any number of labels in its metadata field. A router uses selectors (also known as a selection expression) to select a subset of routes from the entire pool of routes to serve. A selection expression can also involve labels on the route’s namespace. The selected routes form a router shard.

## Prerequisites

* IPI or UPI AWS installation of OCP 4.x
* Two or more workers deployed and running within our OCP cluster
* Cluster admin role access
* AWS CLI configured with access to AWS account
* Access to AWS Console (optional)


## Default IngressController in OCP4

In OCP 3.x, router sharding was implemented by patching the routers directly with the "oc adm router" commands to create/manage the routers and their labels, and to apply the namespace and route selectors. For more information, check the [OCP3.11 Route Sharding](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#router-sharding) official documentation.

But in OCP 4.x, the rules of the game changed: the OCP Routers (and other elements including LBs, DNS entries, etc) are managed by the [OpenShift Ingress Operator](https://github.com/openshift/cluster-ingress-operator).

Ingress Operator is an OpenShift component which enables external access to cluster services by configuring Ingress Controllers, which route traffic as specified by OpenShift Route and Kubernetes Ingress resources.
Furthermore, the Ingress Operator implements the OpenShift ingresscontroller API.

In every new OCP4 cluster, the ingresscontroller "default" is deployed in the openshift-ingress-operator namespace:

```
# oc get ingresscontroller -n openshift-ingress-operator
NAME      AGE
default   50m
```

NOTE: Ingress Operator is a core feature of OpenShift and is enabled out of the box.

As we can see in the ingresscontroller "default", an ingress controller with 2 replicas is deployed. Also pay attention that the spec definition is empty (spec: {}).

```
# oc get ingresscontroller default -n openshift-ingress-operator -o yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  creationTimestamp: "2019-07-03T09:38:34Z"
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  generation: 1
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "12518"
  selfLink: /apis/operator.openshift.io/v1/namespaces/openshift-ingress-operator/ingresscontrollers/default
  uid: 53792b3c-9d76-11e9-a92d-06d31b4c3e82
spec: {}
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2019-07-03T09:39:19Z"
    status: "True"
    type: Available
  domain: apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
  endpointPublishingStrategy:
    type: LoadBalancerService
  selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default

```

In the "openshift-ingress" namespace, the two replicas of the ingress controller "default" are deployed and running on two separate worker nodes.

```
# oc get pod -n openshift-ingress
router-default-d994bf4b-jdhj8      1/1     Running   0          106m
router-default-d994bf4b-zqv2z      1/1     Running   0          106m
```

This is because of the pod antiaffinity rule, which requires that the replicas are not deployed on the same worker node:

```
# oc get pod router-default-5446cd6d49-cbjcg -n openshift-ingress -o yaml | grep -A9 affinity
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: ingresscontroller.operator.openshift.io/deployment-ingresscontroller
            operator: In
            values:
            - default
        topologyKey: kubernetes.io/hostname
```

Furthermore, the [Kubernetes ServiceTypes](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) are also managed by the OpenShift Ingress Operator. In this case, the default ingresscontroller deploys and manages a LoadBalancer Service Type (router-default svc) and a ClusterIP Service Type (router-internal-default svc).

The LoadBalancer Service Type in AWS, deploys and manages an AWS ELB (Classic LoadBalancer type) for each ingresscontroller defined:

```
# oc get svc -n openshift-ingress
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)                      AGE
router-default            LoadBalancer   172.30.154.159   a53e0048d9d7611e9a92d06d31b4c3e8-408658557.eu-central-1.elb.amazonaws.com   80:32611/TCP,443:32297/TCP   81m
router-internal-default   ClusterIP      172.30.136.191   <none>                                                                      80/TCP,443/TCP,1936/TCP      81m
```

Checking the ELB deployed on AWS by the Ingress Operator, we can see that it has a scheme of "Internet Facing".

```
 aws elb describe-load-balancers --load-balancer-name $(oc get svc -n openshift-ingress | grep router-default | awk '{ print $4 }' | cut -d"." -f1 | cut -d"-" -f1) | jq .LoadBalancerDescriptions[].Scheme
"internet-facing"
```

This means that every ELB deployed by the Ingress Operator in this way is publicly exposed to the Internet.

At this point, and until version 4.2, there is no possibility to deploy an ELB with a "Private" schema, focused on having a Load Balancer facing only "Private" subnets and reachable only within those subnets (for example, to use with the *internalapps routes).
In 4.2, [the implementation](https://github.com/openshift/api/blob/release-4.2/operator/v1/types_ingress.go#L179) will be available to expose only on the cluster's private network using LoadBalancerScope = "Internal".


## Router sharding - Adding Ingress Controller for internal traffic application routes

In several cases, customers want to isolate DMZ traffic routes of their applications from the internal traffic application routes (for example, only reachable from inside the private subnet of the cluster).

For this purpose, the :

```
# cat router-internal.yaml
apiVersion: v1
items:
- apiVersion: operator.openshift.io/v1
  kind: IngressController
  metadata:
    name: internal
    namespace: openshift-ingress-operator
  spec:
    domain: internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
    endpointPublishingStrategy:
      type: LoadBalancerService
    nodePlacement:
      nodeSelector:
        matchLabels:
          node-role.kubernetes.io/worker: ""
    routeSelector:
      matchLabels:
        type: internal
  status: {}
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

There are several key values in this ingresscontroller:

* **Domain**: AWS Route53 wildcard A record *internalapps, created and managed automatically by the operator.
* **endpointPublishingStrategy**: used to publish the ingress controller endpoints to other networks, enable load balancer integrations, etc. LoadBalancerService in our case (AWS) to deploy the ELB.
* **nodePlacement**: describes node scheduling configuration for an ingress controller. In our case, it has a matchLabel to deploy the ingress controller only on workers.
* **routeSelector**: used to filter the set of Routes serviced by the ingress controller. If unset, the default is no filtering. In our case, the label "type: internal" is selected to define these internal routes.

For more info, check the [Ingress Operator API](https://github.com/openshift/api/blob/master/operator/v1/types_ingress.go) code, which is well documented with all the possibilities (including those defined above).

Apply the ingresscontroller internal yaml:

```
# oc apply -f router-internal.yaml
```

And check that it is created in the cluster:

```
# oc get ingresscontroller -n openshift-ingress-operator
NAME       AGE
default    103m
internal   21s
```

Automatically, the ingress operator creates a LoadBalancer ServiceType (router-internal as the AWS ELB) and a ClusterIP ServiceType:

```
# oc get svc -n openshift-ingress
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)                      AGE
router-default             LoadBalancer   172.30.154.159   a53e0048d9d7611e9a92d06d31b4c3e8-408658557.eu-central-1.elb.amazonaws.com   80:32611/TCP,443:32297/TCP   104m
router-internal            LoadBalancer   172.30.58.138    ab15128d29d8411e98dd80a317a65344-719607428.eu-central-1.elb.amazonaws.com   80:32432/TCP,443:31324/TCP   91s
router-internal-default    ClusterIP      172.30.136.191   <none>                                                                      80/TCP,443/TCP,1936/TCP      104m
router-internal-internal   ClusterIP      172.30.225.82    <none>                                                                      80/TCP,443/TCP,1936/TCP      91s
```

The logs of the ingress operator reflects the management of the AWS ELB loadbalancer and the Route53 A DNS record generated:

```
# oc logs ingress-operator-6ffcbffff-5s4hr -n openshift-ingress-operator --tail=10
2019-07-03T11:21:56.127Z        INFO    operator.dns    aws/dns.go:270  skipping DNS record update      {"record": {"Zone":{"id":"Z2E0LPV0ATXKFS"},"Type":"ALIAS","Alias":{"Domain":"*.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","Target":"ab15128d29d8411e98dd80a317a65344-719607428.eu-central-1.elb.amazonaws.com"}}}
2019-07-03T11:21:56.127Z        INFO    operator.controller     controller/controller_dns.go:33 ensured DNS record for ingresscontroller       {"namespace": "openshift-ingress-operator", "name": "internal", "record": {"Zone":{"id":"Z2E0LPV0ATXKFS"},"Type":"ALIAS","Alias":{"Domain":"*.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","Target":"ab15128d29d8411e98dd80a317a65344-719607428.eu-central-1.elb.amazonaws.com"}}}
2019-07-03T11:21:56.174Z        DEBUG   operator.init.controller-runtime.controller     controller/controller.go:236    Successfully Reconciled        {"controller": "operator-controller", "request": "openshift-ingress-operator/internal"}
2019-07-03T11:21:56.174Z        INFO    operator.controller     controller/controller.go:101    reconciling     {"request": "openshift-ingress-operator/internal"}
2019-07-03T11:21:56.213Z        INFO    operator.dns    aws/dns.go:270  skipping DNS record update      {"record": {"Zone":{"tags":{"Name":"rcarrata-ipi2-aws-rf2h9-int","kubernetes.io/cluster/rcarrata-ipi2-aws-rf2h9":"owned"}},"Type":"ALIAS","Alias":{"Domain":"*.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","Target":"ab15128d29d8411e98dd80a317a65344-719607428.eu-central-1.elb.amazonaws.com"}}}
2019-07-03T11:21:56.213Z        INFO    operator.controller     controller/controller_dns.go:33 ensured DNS record for ingresscontroller       {"namespace": "openshift-ingress-operator", "name": "internal", "record": {"Zone":{"tags":{"Name":"rcarrata-ipi2-aws-rf2h9-int","kubernetes.io/cluster/rcarrata-ipi2-aws-rf2h9":"owned"}},"Type":"ALIAS","Alias":{"Domain":"*.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","Target":"ab15128d29d8411e98dd80a317a65344-719607428.eu-central-1.elb.amazonaws.com"}}}
```

Furthermore, two replicas of the new ingress controllers appear in the openshift-ingress namespace:

```
# oc get pod -n openshift-ingress
NAME                               READY   STATUS    RESTARTS   AGE
router-default-d994bf4b-jdhj8      1/1     Running   0          116m
router-default-d994bf4b-zqv2z      1/1     Running   0          116m
router-internal-7bd5b76ddd-62fvw   1/1     Running   0          13m
router-internal-7bd5b76ddd-q8tq4   1/1     Running   0          13m
```

## Testing the Route Sharding I - internal application routes

Create a new project and deploy an application for testing purposes (in this case, we use the django-psql-example):

```
# oc new-project test-sharding
# oc new-app django-psql-example
```

The new-app deployment creates two pods, a django frontend and a postgresql database, and also a Service and a Route:

```
# oc get route -n test-sharding
NAME                  HOST/PORT                                                                              PATH   SERVICES              PORT    TERMINATION   WILDCARD
django-psql-example   django-psql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com          django-psql-example   <all>                 None
```

This route is exposed by default to the "router-default", using the *apps. domain route.

Let's tweak the route and add the label that matches the routeSelector defined in our internal ingresscontroller:

```
# cat route-django-psql-example.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: django-psql-example
    template: django-psql-example
    type: internal
  name: django-psql-example
  namespace: test-sharding
spec:
  host: django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
  subdomain: ""
  to:
    kind: Service
    name: django-psql-example
    weight: 100
  wildcardPolicy: None
```

Note that in this tweaked route, a new label "type: internal" is added, and the host in the spec definition is adapted to use the *internalapps domain route.

Let's delete the old route and apply the new one:

```
# oc delete route django-psql-example -n test-sharding
# oc apply -f route-django-psql-example.yaml
```

With a describe of the route, check that the route is created correctly:

```
# oc describe route django-psql-example -n test-sharding
Name:                   django-psql-example
Namespace:              test-sharding
Created:                2 minutes ago
Labels:                 app=django-psql-example
                        template=django-psql-example
                        type=internal
Annotations:            kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"route.openshift.io/v1","kind":"Route","metadata":{"annotations":{},"labels":{"app":"django-psql-example","template":"django-psql-example","type":"internal"},"name":"django-psql-example","namespace":"test-sharding"},"spec":{"host":"django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","subdomain":"","to":{"kind":"Service","name":"django-psql-example","weight":100},"wildcardPolicy":"None"}}

Requested Host:         django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
                          exposed on router default (host apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
                          exposed on router internal (host internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
Path:                   <none>
TLS Termination:        <none>
Insecure Policy:        <none>
Endpoint Port:          <all endpoint ports>

Service:        django-psql-example
Weight:         100 (100%)
Endpoints:      10.129.2.13:8080
```

But, wait a minute! Our route is exposed on both routers! (default and internal):

```
Requested Host:         django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
                          exposed on router default (host apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
                          exposed on router internal (host internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
```

What happened?

By default, the default router has no routeSelector (remember the spec:{} from above?), and for this reason it is exposed not only to our internal router, but also to the default.

Check if the route exposed by the internal router is working:

```
# curl -I  django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.1 200 OK
Server: gunicorn/19.4.5
Date: Wed, 03 Jul 2019 12:21:38 GMT
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 18256
Set-Cookie: 76d45f9cc1a5dfd70510f1d6e9de2f11=8b4050c0c44e03d912c4d741f7e95699; path=/; HttpOnly
Cache-control: private
```

It works! A basic curl proves that the route is exposed correctly and is reachable from outside the cluster.

## Apply a routeSelector into Router Default of OCP4

To ONLY expose the routes of *.internalapps to one router (the internal router), and exclude them from the default router (that exposes the *.apps routes normally), a routeSelector must be defined in the "default" ingresscontroller in the openshift-ingress-operator namespace:

```
# oc get ingresscontroller -n openshift-ingress-operator default -o yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  creationTimestamp: "2019-07-03T09:38:34Z"
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  generation: 3
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "56492"
  selfLink: /apis/operator.openshift.io/v1/namespaces/openshift-ingress-operator/ingresscontrollers/default
  uid: 53792b3c-9d76-11e9-a92d-06d31b4c3e82
spec:
  routeSelector:
    matchLabels:
      type: public
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2019-07-03T09:39:19Z"
    status: "True"
    type: Available
  domain: apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
  endpointPublishingStrategy:
    type: LoadBalancerService
  selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
```

As we can see in the definition of the ingresscontroller, the key is:

```
spec:
  routeSelector:
    matchLabels:
      type: public
```

Now, after editing the default ingresscontroller with the proper routeSelector, our route is only exposed by the internal ingresscontroller/router:

```
# oc describe route route django-psql-example -n test-sharding
Name:                   django-psql-example
Namespace:              test-sharding
Created:                2 minutes ago
Labels:                 app=django-psql-example
                        template=django-psql-example
                        type=internal
Annotations:            kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"route.openshift.io/v1","kind":"Route","metadata":{"annotations":{},"labels":{"app":"django-psql-example","template":"django-psql-example","type":"internal"},"name":"django-psql-example","namespace":"test-sharding"},"spec":{"host":"django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com","subdomain":"","to":{"kind":"Service","name":"django-psql-example","weight":100},"wildcardPolicy":"None"}}

Requested Host:         django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
                          exposed on router internal (host internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
Path:                   <none>
TLS Termination:        <none>
Insecure Policy:        <none>
Endpoint Port:          <all endpoint ports>

Service:        django-psql-example
Weight:         100 (100%)
Endpoints:      10.129.2.13:8080
```

Obviously, the route is still working perfectly because it is exposed by our internal router:

```
Requested Host:         django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
                          exposed on router internal (host internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com) 2 minutes ago
```

```
# curl -I  django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.1 200 OK
Server: gunicorn/19.4.5
Date: Wed, 03 Jul 2019 12:35:01 GMT
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 18256
Set-Cookie: 76d45f9cc1a5dfd70510f1d6e9de2f11=8b4050c0c44e03d912c4d741f7e95699; path=/; HttpOnly
Cache-control: private
```

## Use public routes of the Default Router in OpenShift 4


So, now that the default ingresscontroller includes the routeSelector label "public", the router is serving routes including this label in the route definition.

For example, to test this out, let's deploy a sample app:

```
# oc new-app cakephp-mysql-example
--> Deploying template "openshift/cakephp-mysql-example" to project test-sharding
```

```
# oc get pod
NAME                            READY   STATUS      RESTARTS   AGE
cakephp-mysql-example-1-build   1/1     Running     0          107s
django-psql-example-1-4ftnr     1/1     Running     0          34m
django-psql-example-1-build     0/1     Completed   0          36m
django-psql-example-1-deploy    0/1     Completed   0          34m
mysql-1-deploy                  0/1     Completed   0          105s
mysql-1-j6nx4                   1/1     Running     0          93s
postgresql-1-deploy             0/1     Completed   0          36m
postgresql-1-xhhrt              1/1     Running     0          36m
```

The exposed route for our new application is:

```
# oc get route
NAME                    HOST/PORT                                                                                      PATH   SERVICES                PORT    TERMINATION   WILDCARD
cakephp-mysql-example   cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com                cakephp-mysql-example   <all>                 None
```

So, if we test the exposed route of our application, we receive a 503 error (Service Unavailable):

```
# curl -I  cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.0 503 Service Unavailable
Pragma: no-cache
Cache-Control: private, max-age=0, no-cache, no-store
Connection: close
Content-Type: text/html
```

This error occurs because the router is only serving routes defined with the routeSelector "type: public".

If we apply the labels to the route of our app:

```
# cat route-django-psql-example.yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  labels:
    app: django-psql-example
    template: django-psql-example
    type: public
  name: django-psql-example
  namespace: test-sharding
spec:
  host: django-psql-example-test-sharding.internalapps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
  subdomain: ""
  to:
    kind: Service
    name: django-psql-example
    weight: 100
  wildcardPolicy: None
```

The route for our app that is exposed in our router... works perfectly again!

```
# curl -I cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.1 200 OK
Date: Wed, 03 Jul 2019 12:59:58 GMT
Server: Apache/2.4.34 (Red Hat) OpenSSL/1.0.2k-fips
Content-Type: text/html; charset=UTF-8
Set-Cookie: 640b33b422cc3251ac141ece0847c81c=3bb02e599f47389f9820329908ca84e9; path=/; HttpOnly
Cache-control: private
```

## Namespace selector in Route Sharding

Another type of selector in the ingresscontrollers is namespaceSelectors. These selectors allow only the routes exposed in those namespaces to be served by the routers labeled accordingly.

As an example, we can add the namespaceSelector "environment: dmz" and also combine it with the routeSelector "type: public". With this combination, we can ensure that our developers only deploy routes to the default/public ingresscontroller/router under two conditions: being in the appropriate namespace and having the label for the routeSelector type public. With only the routeSelector, anyone can expose apps to our default/public router without any restriction or limit, which the namespaceSelector addresses:

```
# oc get ingresscontroller default -n openshift-ingress-operator -o yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  creationTimestamp: "2019-07-03T09:38:34Z"
  finalizers:
  - ingresscontroller.operator.openshift.io/finalizer-ingresscontroller
  generation: 5
  name: default
  namespace: openshift-ingress-operator
  resourceVersion: "153732"
  selfLink: /apis/operator.openshift.io/v1/namespaces/openshift-ingress-operator/ingresscontrollers/default
  uid: 53792b3c-9d76-11e9-a92d-06d31b4c3e82
spec:
  namespaceSelector:
    matchLabels:
      environment: dmz
  routeSelector:
    matchLabels:
      type: public
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: "2019-07-03T09:39:19Z"
    status: "True"
    type: Available
  domain: apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
  endpointPublishingStrategy:
    type: LoadBalancerService
  selector: ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default
```

As we can check, our previous route doesn't work properly (503 error received), because the label "environment: dmz" is not applied to our namespace:

```
# curl -I cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.0 503 Service Unavailable
Pragma: no-cache
Cache-Control: private, max-age=0, no-cache, no-store
Connection: close
Content-Type: text/html
```

If we apply this label:

```
# oc label ns test-sharding environment=dmz
namespace/test-sharding labeled

# oc get ns test-sharding -o yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/description: ""
    openshift.io/display-name: ""
    openshift.io/requester: system:admin
    openshift.io/sa.scc.mcs: s0:c22,c9
    openshift.io/sa.scc.supplemental-groups: 1000480000/10000
    openshift.io/sa.scc.uid-range: 1000480000/10000
  creationTimestamp: "2019-07-03T12:05:53Z"
  labels:
    environment: dmz
  name: test-sharding
  resourceVersion: "154758"
  selfLink: /api/v1/namespaces/test-sharding
  uid: e843a65c-9d8a-11e9-9f94-02de6e49b8a0
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

The route works again, within our namespace:

```
# curl -I cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.1 200 OK
Date: Wed, 03 Jul 2019 19:07:20 GMT
Server: Apache/2.4.34 (Red Hat) OpenSSL/1.0.2k-fips
Content-Type: text/html; charset=UTF-8
Set-Cookie: 640b33b422cc3251ac141ece0847c81c=3bb02e599f47389f9820329908ca84e9; path=/; HttpOnly
Cache-control: private
```

## Conclusion

In this blog post, we explored and explained Route Sharding in OpenShift 4. Route Sharding has changed slightly operationally, but maintains the basis of the implementation from OpenShift 3.

In future blog posts, we will explore and analyse the possibility of having IngressControllers using the LoadBalancer Service Type with the internal schema, to deploy AWS LoadBalancers only to the private subnets of our AWS VPC deployment.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
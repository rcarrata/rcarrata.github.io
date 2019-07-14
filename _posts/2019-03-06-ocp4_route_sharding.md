---
layout: post
title: Openshift 4 - Route Sharding
date: 2019-06-03
type: post
published: true
status: publish
categories:
- Blog
- Personal
tags: []
author: rcarrata
comments: true
---

# Route Sharding in Openshift 4 

This blog post aims to provide a guide to implement Route Sharding in Openshift Container Platform 4 (deployed in AWS), creating multiple routers for particular purposes (for example in this specific case, separating the internal and public/dmz application routes).

## Overview

In OCP, each route can have any number of labels in its metadata field. A router uses selectors (also known as a selection expression) to select a subset of routes from the entire pool of routes to serve. A selection expression can also involve labels on the route’s namespace. The selected routes form a router shard.

## Prerequisites

* IPI or UPI AWS installation of OCP 4.x
* Two or more workers deployed and running within our OCP cluster
* Cluster admin role access
* AWS CLI configured with access to AWS account 
* Access to AWS Console (optional) 


## Default IngressController in OCP4 

In the OCP 3.x version, the router sharding was implemented with patching directly the routers with the "oc adm router" commands for create/manage the routers and their labels, to apply the namespace and route selectors. For more information, check the [OCP3.11 Route Sharding](https://docs.openshift.com/container-platform/3.11/architecture/networking/routes.html#router-sharding) official documentation.

But in OCP 4.x, the rules of the game changed: the OCP Routers (and other elements including LBs, DNS entries, etc) are managed by the [Openshift Ingress Operator](https://github.com/openshift/cluster-ingress-operator). 

Ingress Operator is an OpenShift component which enables external access to cluster services by configuring Ingress Controllers, which route traffic as specified by OpenShift Route and Kubernetes Ingress resources.
Furthermore, the Ingress Operator implements the OpenShift ingresscontroller API. 

Into every new OCP4 cluster, the ingresscontroller "default" is deployed into the openshift-ingress-operator namespace:

```
# oc get ingresscontroller -n openshift-ingress-operator
NAME      AGE
default   50m
```

NOTE: Ingress Operator is a core feature of OpenShift and is enabled out of the box.

As we can see into the ingresscontroller "default", a ingress controller with replica 2 is deployed. Also pay attention that the spec definition is empty (spec: {}).

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

In the namespace of "openshift-ingress" the two replicas of the ingress controller "default" are deployed and are running in two separates worker nodes. 

```
# oc get pod -n openshift-ingress 
router-default-d994bf4b-jdhj8      1/1     Running   0          106m
router-default-d994bf4b-zqv2z      1/1     Running   0          106m
```

This is because of the pod antiaffinity rule defined, that requires that the replicas are not deployed into the same worker node:

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

Furthermore, the [Kubernetes ServiceTypes](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) are managed also by the Openshift Ingress Operator. In this case the default ingresscontroller,deployed and manages a LoadBalancer Service Type (router-default svc) and a ClusterIP Service Type (router-internal-default svc).

The LoadBalancer Service Type in AWS, deploys and manages an AWS ELB (Classic LoadBalancer type) for each ingresscontroller defined:

```
# oc get svc -n openshift-ingress
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)                      AGE
router-default            LoadBalancer   172.30.154.159   a53e0048d9d7611e9a92d06d31b4c3e8-408658557.eu-central-1.elb.amazonaws.com   80:32611/TCP,443:32297/TCP   81m
router-internal-default   ClusterIP      172.30.136.191   <none>                                                                      80/TCP,443/TCP,1936/TCP      81m
```

Checking the ELB deployed on top of AWS by the Ingress Operator, see that have an scheme "Internet Facing".

```
 aws elb describe-load-balancers --load-balancer-name $(oc get svc -n openshift-ingress | grep router-default | awk '{ print $4 }' | cut -d"." -f1 | cut -d"-" -f1) | jq .LoadBalancerDescriptions[].Scheme
"internet-facing"
```

This involves that every ELB deployed by the ingressoperator in this way, is publicly exposed to Internet.

In this moments, and until the 4.2 there is no possibility to deploy an ELB with an Schema "Private", focused in have a Load Balancer facing only "Private" subnets and reachable only within this Subnets (for example to use with the routes *internalapps). 
At 4.2 there will be available [the implementation](https://github.com/openshift/api/blob/release-4.2/operator/v1/types_ingress.go#L179) to expose only on the cluster's private network using the LoadBalancerScope = "Internal" 


## Router sharding - Adding Ingress Controller for internal traffic application routes

In several cases, customers want to isolated DMZ traffic routes of their applications, from the internal traffic application routes (for example, only reachable from inside the Private subnet of the cluster).

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

There is several key values into this ingresscontroller:

* **Domain**: AWS Route53 wildcard A record *internalapps created and manage automatically by the operator
* **endpointPublishingStrategy**: used to publish the ingress controller endpoints to other networks, enable load balancer integrations, etc. LoadBalancerService in our case (AWS) for deploy the ELB.
* **nodePlacement**: NodePlacement describes node scheduling configuration for an ingress controller. In our case 
have a matchLabel to deploy the ingress controller only into Workers.
* **routeSelector**: routeSelector is used to filter the set of Routes serviced by the ingress controller. If unset, the default is no filtering. In our case, a label "type: internal" is selected for define these internal routes.

For more info, check the [Ingress Operator API](https://github.com/openshift/api/blob/master/operator/v1/types_ingress.go) code that is well documented with the all the possibilities (including those defined above).

Apply the ingresscontroller internal yaml:

```
# oc apply -f router-internal.yaml
```

And check that is created in the cluster:

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

Furthermore, two replicas of the new brand ingress controllers appears in the openshift-ingress namespace:

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

Let's tweak the route, and add the label that matches to the routeSelector defined into our internal ingresscontroller: 

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

Check that in this tweaked route, a new label "type: internal" is added, and also the host into the spec definition is adapted to use the *internalapps domain route.

Let's delete the old route and apply the new one

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

By default, the default router have not routeSelector (remember the spec:{} from above?), and for this reason is exposed not only to our internal router, also is exposed to the default.

Check if the route expose by the internal router is working:

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

It works! A basic curl is probing us that the route is exposed correctly and is reachable from outside the cluster.

## Apply a routeSelector into Router Default of OCP4

For ONLY expose the routes of *.internalapps to one router (the internal router), and avoid to the default router (that expose the *.apps routes normally), a routeSelector must be defined into the ingresscontroller "default" in the namespace of openshift-ingress-operator:

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

As we see into the definition of the ingresscontroller, the key is:

```
spec:
  routeSelector:
    matchLabels:
      type: public
```

Now, and after the edition of the ingresscontroller default with the proper routeSelector, our route is only exposed by the ingresscontroller/router internal only:

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

Obviously, the route is still working perfectly because is exposed by our internal router:

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

## Use public routes of the Default Router in Openshift 4


So, now that the default ingresscontroller includes the routeSelector label "public", the router is serving routes including this label in the route definition.

For example, for testing out this if we deploy an app example:

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

The route exposed of the services for our new brand applications are: 

```
# oc get route
NAME                    HOST/PORT                                                                                      PATH   SERVICES                PORT    TERMINATION   WILDCARD
cakephp-mysql-example   cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com                cakephp-mysql-example   <all>                 None
```

So, if we test the route exposed of our application, we receive an 503 error (Service Unavailable):

```
# curl -I  cakephp-mysql-example-test-sharding.apps.rcarrata-ipi2-aws.c398.sandbox389.opentlc.com
HTTP/1.0 503 Service Unavailable
Pragma: no-cache
Cache-Control: private, max-age=0, no-cache, no-store
Connection: close
Content-Type: text/html
```

This error is because of the router is serving routes defined with the routeSelector "type: public".

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

Another selectors in the ingresscontrollers are the namespaceSelectors. This selectors, allow that only the routes exposed in that namespaces are served by the routers labeled with this.

As example we can add the namespaceSelector "environment: dmz", and also combine to the routeSelector: "type: public". With this combination, we can ensure that our developers only deploys routes to the default/public ingresscontroller/router with two conditions: being in the namespace appropiated and with the label for the routeSelector type public. With only routeSelector, anyone can expose apps to our default/public without any restriction or limit, that with the namespaceSelector have:

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

As we can check, our previous route doesn't work properly (503 error received), because the label into our namespace of "environment: dmz" is not applied:

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

## Links of interest


In this moments, there is no official documentation (in OCP4.1) about how implement the router sharding in OCP4

* [How to create a router shard on OpenShift 4.0](https://access.redhat.com/solutions/3880621)

Tested with OCP 4.1.3/4.1.4 in Opentlc environments.


## Conclusion 

In this blog post, we worked and explained the Route Sharding in Openshift 4. This Route Sharding changed slightly operational, but maintain the basis of the implementation into Openshift 3.

In future blog posts, we will explore and analyse the possibility to have Ingresscontrollers using LoadBalancers Service Type with the schema internal, to deploy AWSLoadBalancers only to the Private Subnets of our AWS VPC deployment.

Special thanks to Pablo Alonso for their help and collaboration writing the KCS and helping with the content of this blog.

Happy Openshifting!
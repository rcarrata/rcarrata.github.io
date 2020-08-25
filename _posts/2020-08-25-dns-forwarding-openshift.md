---
layout: post
title: DNS Forwarding in Openshift
date: 2020-08-25
type: post
published: true
status: publish
categories:
- Openshift
tags: []
author: rcarrata
comments: true
---

How can I use a custom DNS in our cluster without modify the actual cluster DNS on top of for example AWS or Azure? How can I forward custom DNS and use specific DNS in an On-Premise Datacenter?

Let's nslookup it out!

### Overview and Example Scenario

But in which scenarios we want a Custom DNS resolvable for our cluster?

An example could be to have an Openshift Cluster deployed on top of AWS and a Direct Connect (a big cable connecting AWS VPC Openshift cluster and the on-premise) to the On-Premise Datacenter, because the enterprises have a Hybrid Cloud deployments, and some resources are only available within the cluster (as for example the DNS that solves the domain of ocp4.rober.lab).

We in this specific situation the apps.ocp4.rober.lab can not be resolved by the DNS of AWS, and need to be resolved by the specific custom DNS.

NOTE: This blog post is valid from Openshift 4.3+, and its tested with a 4.4.3 version in AWS.

### DNS Operator in Openshift 4

As we described in my [DNS Deep Dive in Openshift](https://rcarrata.com/openshift/dns-deep-dive-in-openshift/) blog post, in Openshift 4, the DNS Operator deploys and manages CoreDNS to provide a name resolution service to pods, enabling DNS-based Kubernetes Service discovery in OpenShift.

Furthermore, we investigated that with the Corefile of the CoreDNS, managed with the DNS Operator is configured the Coredns forwarding plugin that define the queries that are not resolved within the cluster by the predefined resolvers.

In an overview, if the domain that is requesting our pod/resource/app is not inside of the cluster, the CoreDNS will forward to the resolvers inside of the /etc/resolv.conf

### Testing name resolutions without DNS Forwarding

Let's test our name resolution without enabling the DNS Forwarding and see what happens:


* Enter to one random pod (I selected the Openshift insights pod but any pod in the cluster will have the exact same behaviour):

```
$ POD=$(kubectl get pod -n openshift-insights | awk '{ print $1 }' | grep -v NAME)
$ kubectl exec -ti $POD -n openshift-insights sh
```

* Execute the nslookup to one server that is reachable but is not inside to our cluster:

```
sh-4.2# nslookup labs.opentlc.com
Server:         172.30.0.10
Address:        172.30.0.10#53

Non-authoritative answer:
Name:   labs02.opentlc.com
Address: 169.45.246.141
```

What is an non-authoritative answer? And why this appears to my nslookup?

An **authoritative answer** comes from a nameserver that is considered authoritative for the domainwhich it's returning a record for (one of the nameservers in the list for the domain did a lookup on), and a **non-authoritative answer** comes from anywhere else (a nameserver not in the list for the domain that we lookup on).

It's basically a distinction between a nameserver that's an official nameserver for the domain you're querying, and a nameserver that isn't. Nameservers that aren't authoritative are getting their answers second (or third or fourth...) hand - just relaying the information along from somewhere else.

So in this case, this output is caused because we are not using the nameserver that is considered authoritative.

Let's use an authoritative nameserver and test again:

```
$ nslookup labs.opentlc.com cam.opentlc.com
Server:         cam.opentlc.com
Address:        52.2.39.130#53

labs.opentlc.com        canonical name = labs02.opentlc.com.
Name:   labs02.opentlc.com
Address: 169.45.246.141
```

in NSlookup if we specify the nameserver after the dns name that we try to solve, this resolution will be executed using this nameserver instead of the local nameserver (in our case the CoreDNS).

As we can noticed the non-authoritative message dissapeared and instead we will using the nameserver specified (check the server and the address in the two first lines).

Ok, now that we have a nameserver that resolves properly the opentlc.com subdomains, let's add this to our Openshift cluster using the DNS Forwarding.

### Enabling the DNS Forwarding in Openshift 4

As we describe before, we can use the DNS forwarding to override the forwarding configuration identified in etc/resolv.conf on a per-zone basis by specifying which name server should be used for a given zone.

Let's use the yaml with the dns forwarding specification:

```
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  name: default
spec:
  servers:
  - name: opentlc-dns
    zones:
      - opentlc.com
    forwardPlugin:
      upstreams:
        - 52.2.39.130
```

This allows the Operator to create and update the ConfigMap named dns-default with additional server configuration blocks based on Server. If none of the servers has a zone that matches the query, then name resolution falls back to the name servers that are specified in /etc/resolv.conf.

In this case, every pod in our cluster when request any resolution with the specific zone defined (in our case the opentlc.com) will be forwarded to use the DNS in the upstream spec.

* Apply the configuration to the cluster:

```
$ oc apply -f dns-operator-forwarding.yml
dns.operator.openshift.io/default configured
```

* Check that the corefile in the coredns is updated with the definitions that we applied:

```
$ oc get configmap/dns-default -n openshift-dns -o yaml
apiVersion: v1
data:
  Corefile: |
    # opentlc-dns
    opentlc.com:5353 {
        forward . 52.2.39.130
    }
    .:5353 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
            policy sequential
        }
        cache 30
        reload
    }
kind: ConfigMap
metadata:
  labels:
    dns.operator.openshift.io/owning-dns: default
  name: dns-default
  namespace: openshift-dns
  ownerReferences:
  - apiVersion: operator.openshift.io/v1
    controller: true
    kind: DNS
    name: default
```

remember that the Corefile is located in each coredns pod, but is applied in configured in Openshift4 through a configmap in the openshift-dns.

### Testing Resolutions with DNS Forwarding enabled

Let's test if the DNS forwarding works:

```
$ nslookup labs.opentlc.com
Server:         cam.opentlc.com
Address:        52.2.39.130#53

labs.opentlc.com        canonical name = labs02.opentlc.com.
Name:   labs02.opentlc.com
Address: 169.45.246.141
```

and voilà, it works like a charm without specifying the nameserver that we want to use. The coredns detected that the request belongs to the zone custom, and automatically forwarded the request to the specific nameserver defined.


Let's execute another request with the nslookup in debug mode, to other webservice (mx00.opentlc.com) that belongs to the DNS zone configured (opentlc.com):

```
$ nslookup
> set debug
> mx00.opentlc.com
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        mx00.opentlc.com.tutorial.svc.cluster.local, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
        origin = ns.dns.cluster.local
        mail addr = hostmaster.cluster.local
        serial = 1585154373
        refresh = 7200
        retry = 1800
        expire = 86400
        minimum = 5
        ttl = 5
    ADDITIONAL RECORDS:
------------
** server can't find mx00.opentlc.com.tutorial.svc.cluster.local: NXDOMAIN
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        mx00.opentlc.com.svc.cluster.local, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
        origin = ns.dns.cluster.local
        mail addr = hostmaster.cluster.local
        serial = 1585154433
        refresh = 7200
        retry = 1800
        expire = 86400
        minimum = 5
        ttl = 5
    ADDITIONAL RECORDS:
------------
** server can't find mx00.opentlc.com.svc.cluster.local: NXDOMAIN
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        mx00.opentlc.com.cluster.local, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
        origin = ns.dns.cluster.local
        mail addr = hostmaster.cluster.local
        serial = 1585154396
        refresh = 7200
        retry = 1800
        expire = 86400
        minimum = 5
        ttl = 5
    ADDITIONAL RECORDS:
------------
** server can't find mx00.opentlc.com.cluster.local: NXDOMAIN
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        mx00.opentlc.com.eu-west-1.compute.internal, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
** server can't find mx00.opentlc.com.eu-west-1.compute.internal: NXDOMAIN
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        mx00.opentlc.com, type = A, class = IN
    ANSWERS:
    ->  mx00.opentlc.com
        internet address = 23.246.247.59
        ttl = 1200
    AUTHORITY RECORDS:
    ->  opentlc.com
        nameserver = ipa3.opentlc.com.
        ttl = 86400
    ->  opentlc.com
        nameserver = cam.opentlc.com.
        ttl = 86400
    ->  opentlc.com
        nameserver = ipa2.opentlc.com.
        ttl = 86400
    ->  opentlc.com
        nameserver = ipa1.opentlc.com.
        ttl = 86400
    ADDITIONAL RECORDS:
    ->  cam.opentlc.com
        internet address = 52.2.39.130
        ttl = 1200
    ->  ipa2.opentlc.com
        internet address = 23.246.247.52
        ttl = 1200
    ->  ipa1.opentlc.com
        internet address = 23.246.247.53
        ttl = 1200
    ->  ipa3.opentlc.com
        internet address = 169.45.251.57
        ttl = 1200
------------
Name:   mx00.opentlc.com
Address: 23.246.247.59
------------
    QUESTIONS:
        mx00.opentlc.com, type = AAAA, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  opentlc.com
        origin = cam.opentlc.com
        mail addr = hostmaster.opentlc.com
        serial = 1585105216
        refresh = 3600
        retry = 900
        expire = 1209600
        minimum = 3600
        ttl = 3600
    ADDITIONAL RECORDS:
------------
```
as we can check the resolution tried to use first the coredns request inside of the cluster (check my dns deep dive blog post for more details), and when coredns checked that this webserver not belong to the cluster, sends the request to the forwarding upstream server that is defined.

In the last part of the debug request, we can see that the nameserver defined (cam.opentlc.com) is answering properly, and we can see the authority records of the webserver and some specifications also.

And that's it for this blog post.

Remember that the DNS forwarding can be useful for define private DNS servers to use, or any nameserver that we want to be used for a custom zone / subdomain of our cluster.

Happy nslookuping!

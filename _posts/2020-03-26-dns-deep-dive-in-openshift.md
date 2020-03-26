---
layout: post
title: DNS Deep Dive in Openshift 4
date: 2020-03-26
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

Let's dig and nslookup it out!

### Overview and Example Scenario

But in which scenarios we want a Custom DNS resolvable for our cluster?

 and happy Openshifting!ne example could be to have an Openshift Cluster deployed on top of AWS and a Direct Connect (a big cable connecting AWS VPC Openshift cluster and the on-premise) to the On-Premise Datacenter, because the enterprises have a Hybrid Cloud deployments, and some resources are only available within the cluster (as for example the DNS that solves the domain of ocp4.rober.lab).

We in this specific situation the apps.ocp4.rober.lab can not be resolved by the DNS of AWS, and need to be resolved by the specific custom DNS.

### DNS Operator in Openshift 4

In Openshift 4, the DNS Operator deploys and manages CoreDNS to provide a name resolution service to pods, enabling DNS-based Kubernetes Service discovery in OpenShift.

The DNS Operator implements the dns API from the operator.openshift.io API group. The operator deploys CoreDNS using a DaemonSet, creates a Service for the DaemonSet, and configures the kubelet to instruct pods to use the CoreDNS Service IP for name resolution.

```
$ oc describe dns.operator/default
Name:         default
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  operator.openshift.io/v1
Kind:         DNS
Metadata:
  Creation Timestamp:  2020-03-18T22:59:28Z
  Finalizers:
    dns.operator.openshift.io/dns-controller
  Generation:        1
  Resource Version:  1814646
  Self Link:         /apis/operator.openshift.io/v1/dnses/default
  UID:               fe6f8cd3-d587-4b74-904a-83b9ab35b2b6
Spec:
Status:
  Cluster Domain:  cluster.local
  Cluster IP:      172.30.0.10
```

### Analysis of CoreDNS in Openshift 4

As we described before CoreDNS is a flexible, extensible DNS server that can serve as the Openshift/Kubernetes cluster DNS.

The DNS server supports forward lookups (A records), port lookups (SRV records), reverse IP address lookups (PTR records), and more.

This can be configured with a Corefile, which is the CoreDNS configuration file. This Corefile is updated and configured by the DNS-Operator and the pods running the Coredns are located in the Openshift-dns namespaces.

In the namespace of openshift-dns we can check the DaemonSet and the several pods running the Coredns in each node in our cluster:

* Pods of the Coredns in openshift-dns namespaces:

```
 oc get pod -n openshift-dns
NAME                READY   STATUS    RESTARTS   AGE
dns-default-9jlc7   2/2     Running   0          51d
dns-default-b8rkg   2/2     Running   2          55d
dns-default-f284v   2/2     Running   2          55d
dns-default-ffs2l   2/2     Running   0          55d
dns-default-gtlvw   2/2     Running   2          55d
dns-default-gw5wp   2/2     Running   0          55d
dns-default-n7dnk   2/2     Running   2          55d
dns-default-ndnbm   2/2     Running   2          55d
dns-default-thmlg   2/2     Running   2          55d
dns-default-v5f5q   2/2     Running   0          51d
dns-default-vrgkh   2/2     Running   0          51d
dns-default-xv7vl   2/2     Running   0          55d
```

* DaemonSet of dns-default in ns openshift-dns:

```
oc get ds -n openshift-dns
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
dns-default   12        12        12      12           12          kubernetes.io/os=linux   240d
```

* In one of this pods we can see the process of the coredns running with the config file Corefile:

```
$ oc exec -ti dns-default-69cqb -n openshift-dns bash
Defaulting container name to dns.
Use 'oc describe pod/dns-default-69cqb -n openshift-dns' to see all of the containers in this pod.

[root@dns-default-69cqb /]# ps auxww
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.2  0.0 974732 57908 ?        Ssl  Feb03 194:36 coredns -conf /etc/coredns/Corefile
```

* If we take a look to the Corefile config file we discover some interesting stuff:

```
[root@dns-default-69cqb /]# cat /etc/coredns/Corefile
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
```

1. If we check within the line that start with "kubernetes":

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
```

that enables a plugin that configures that CoreDNS will reply to DNS queries based on IP of the services and pods of Kubernetes.
In other words, this plugin implements the Kubernetes DNS-Based Service Discovery Specification.

For example the cluster.local within this line handle all queries in the custer.local zone (and also in-addr.arpa the reverse dns lookups).

So, every resource within the Openshift Cluster uses the CoreDNS for the DNS resolution, and remember that each node has a Coredns pod (controlled by a DaemonSet). We dig in a bit later.

For more information check the [CoreDNS kubernetes plugin documentation](https://coredns.io/plugins/kubernetes/).

2. On the other hand, another interesting line is

```
forward . /etc/resolv.conf {
```

this enables the coredns forwarding plugin and define that any queries that are not within the cluster domain of Kubernetes will be forwarded to predefined resolvers (/etc/resolv.conf).

So, if the domain that is requesting our pod/resource/app is not inside of the cluster, the CoreDNS will forward to the resolvers inside of the /etc/resolv.conf

* So what's in this resolv.conf in each dns-default/coreDNS pod?

In the case of AWS, it's the resolver of AWS VPC where Openshift is deployed:

```
[root@dns-default-69cqb /]# cat /etc/resolv.conf
search eu-west-1.compute.internal
nameserver 10.0.0.2
```

as we see the search parameter is defined by the aws-region-az.compute.internal, and the nameserver is the 10.0.0.2:

```
# nslookup 10.0.0.2 10.0.0.2
2.0.0.10.in-addr.arpa   name = ip-10-0-0-2.eu-west-1.compute.internal.
```

### DNS Resolutions inside of Openshif4 cluster

So in order to check the resolution we need to have the proper net tools installed, and for this purpose we will create a debug pod of rhel-tools that already had installed it:

```
$ oc debug -t deployment/customer --image registry.access.redhat.c
om/rhel7/rhel-tools
Defaulting container name to customer.
Use 'oc describe pod/customer-debug -n tutorial' to see all of the containers in this pod.

Starting pod/customer-debug ...

sh-4.2# dig -v
DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7
```

So this pod have also their own /etc/resolv.conf that points the resolvers and the search domains:

```
sh-4.2# cat /etc/resolv.conf
search tutorial.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal
nameserver 172.30.0.10
options ndots:5
```

so the domain that will be search for are sequentially from the <namespace>.svc.cluster.local in first place, and after that the svc.cluster.local and cluster.local that are solved by the CoreDNS (without forwarding).
Furthermore, we have the eu-west-1.compute.internal that is the AWS resolver described before, that is used when the domain is not located in our Openshift cluster.

On the other hand, the nameserver is the 172.30.0.10 that is... the Coredns pod Service IP!

```
sh-4.2# nslookup 172.30.0.10
10.0.30.172.in-addr.arpa        name = dns-default.openshift-dns.svc.cluster.local.
```

* This dns-default.openshift-dns.svc.cluster.local. is the service that loadbalances the several endpoints in the pod:

```
$ oc get svc -n openshift-dns
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
dns-default   ClusterIP   172.30.0.10   <none>        53/UDP,53/TCP,9153/TCP   7d1h

$ oc describe svc -n openshift-dns | grep TargetPort -A1
TargetPort:        dns/UDP
Endpoints:         10.128.0.5:5353,10.128.2.8:5353,10.128.4.6:5353 + 9 more...
```

if we select one of this endpoints is efectively one Coredns pod:

```
$ oc get pod -n openshift-dns -o wide | grep 10.128.0.5
dns-default-s9njn   2/2     Running   16         7d1h   10.128.0.5   ip-10-0-144-209.eu-west-1.compute.internal   <none>           <none>
```

#### Resolve inside the same namespace:

```
# nslookup preference
Server:         172.30.0.10
Address:        172.30.0.10#53

Name:   preference.tutorial.svc.cluster.local
Address: 172.30.179.217

$ oc get svc preference
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
preference   ClusterIP   172.30.179.217   <none>        8080/TCP   29h
```

as we can see the resolution of the preference svc, is resolved without the whole subdomain, because the uses of the search described before.

```
sh-4.2# nslookup -debug customer
Server:         172.30.0.10
Address:        172.30.0.10#53

------------
    QUESTIONS:
        customer.tutorial.svc.cluster.local, type = A, class = IN
    ANSWERS:
    ->  customer.tutorial.svc.cluster.local
        internet address = 172.30.188.88
        ttl = 5
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
Name:   customer.tutorial.svc.cluster.local
Address: 172.30.188.88
------------
    QUESTIONS:
        customer.tutorial.svc.cluster.local, type = AAAA, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
        origin = ns.dns.cluster.local
        mail addr = hostmaster.cluster.local
        serial = 1585177538
        refresh = 7200
        retry = 1800
        expire = 86400
        minimum = 5
        ttl = 5
    ADDITIONAL RECORDS:
------------
```

#### Resolve between different namespaces:

In this case we want to resolve the svc of the console of Openshift:

```
sh-4.2# nslookup console
Server:         172.30.0.10
Address:        172.30.0.10#53

** server can't find console: NXDOMAIN
```

what happended? the query is not answered because is not specified the namespace in the same query:

```
sh-4.2# nslookup -debug console | grep NXDOMAIN
** server can't find console.tutorial.svc.cluster.local: NXDOMAIN
** server can't find console.svc.cluster.local: NXDOMAIN
** server can't find console.cluster.local: NXDOMAIN
** server can't find console.eu-west-1.compute.internal: NXDOMAIN
** server can't find console: NXDOMAIN
```

To solve this, we need to ensure that the namespace is pointed into the request:

```
# nslookup console.openshift-console
Server:         172.30.0.10
Address:        172.30.0.10#53

Name:   console.openshift-console.svc.cluster.local
Address: 172.30.105.179

$ oc get svc -n openshift-console console
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
console   ClusterIP   172.30.105.179   <none>        443/TCP   7d1h
```

```
sh-4.2# nslookup -debug console.openshift-console | grep NXDOMAIN
** server can't find console.openshift-console.tutorial.svc.cluster.local: NXDOMAIN

sh-4.2# nslookup -debug console.openshift-console | grep Name -A1
Name:   console.openshift-console.svc.cluster.local
Address: 172.30.105.179
```

so in this case, the console.openshift-console is not resolve in the first place as declared in the search with tutorial.svc.cluster (remember that was: search tutorial.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal), but the second occurence resolves correctly - svc.cluster.local with the console.openshift-console.

### DNS Resolution outside of Openshift

But in the case of the resolution outside the cluster, what are the steps behind the hood?

As we can check, a curl and therefore the resolution of a domain outside of the cluster answers perfectly:

```
sh-4.2# curl https://elpais.com -I
HTTP/1.1 200 OK
```

What happened?

The CoreDNS tried to resolved in the domains that the search described, but because have not the proper answer, forwarded the query to the resolver of AWS (remember the line of forward . /etc/resolv.conf and the 10.0.0.2 nameserver of AWS?).

So the flow is:

```
sh-4.2# nslookup -debug elpais.com | grep NXDOMAIN
** server can't find elpais.com.tutorial.svc.cluster.local: NXDOMAIN
** server can't find elpais.com.svc.cluster.local: NXDOMAIN
** server can't find elpais.com.cluster.local: NXDOMAIN
** server can't find elpais.com.eu-west-1.compute.internal: NXDOMAIN
...
------------
    QUESTIONS:
        elpais.com, type = A, class = IN
    ANSWERS:
    ->  elpais.com
        internet address = 23.203.249.106
        ttl = 20
    ->  elpais.com
        internet address = 2.21.33.72
        ttl = 20
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
Non-authoritative answer:
Name:   elpais.com
Address: 23.203.249.106
Name:   elpais.com
Address: 2.21.33.72
------------
    QUESTIONS:
        elpais.com, type = AAAA, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  elpais.com
        origin = a1-173.akam.net
        mail addr = dns.prisacom.com
        serial = 2015111949
        refresh = 10800
        retry = 3600
        expire = 1209600
        minimum = 3600
        ttl = 30
    ADDITIONAL RECORDS:
------------
```

CoreDNS search -> CoreDNS Forwards towards AWS DNS 10.0.0.2 -> AWS DNS tries to solve and forwards the query until the dns.prisacom.com returns and solves the IP.

So that's it! In the following blog post, we will use the DNS Operator to DNS Forwaring to Custom DNS, to try to solve custom domains.

Stay tuned and happy Openshifting!

---

layout: single
title: DNS Deep Dive in OpenShift 4
date: 2020-03-26
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Administration", "OpenShift", "Networking", "DNS"]
author: rcarrata
comments: true
---

How the DNS of OpenShift 4 works? And the operators to manage them? And how can I check the resolution basics within and externally to my OCP4 cluster?

Let's dig and nslookup it out!

NOTE: Blog Post updated in 22Jul2022 and tested in **OpenShift 4.10**. Working for all versions of OpenShift.

### 1. DNS Operator in OpenShift 4

In OpenShift 4, the DNS Operator deploys and manages CoreDNS to provide a name resolution service to pods, enabling DNS-based Kubernetes Service discovery in OpenShift.

The DNS Operator implements the dns API from the operator.openshift.io API group. The operator deploys CoreDNS using a DaemonSet, creates a Service for the DaemonSet, and configures the kubelet to instruct pods to use the CoreDNS Service IP for name resolution.

* The DNS cluster operator describes as:

```bash
$ oc describe clusteroperator/dns
Name:         dns
...
API Version:  config.openshift.io/v1
Kind:         ClusterOperator
Spec:
Status:
  Conditions:
    Last Transition Time:  2020-03-26T11:40:45Z
    Message:               All desired DNS DaemonSets available and operand Namespace exists
    Reason:                AsExpected
    Status:                False
    Type:                  Degraded
```

* On the other hand the dns operator is configured as:

```bash
$  oc get dnses.operator.openshift.io/default -o yaml
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
...
  name: default
spec: {}
status:
  clusterDomain: cluster.local
  clusterIP: 172.30.0.10
  conditions:
```

NOTE: Check that dns operator have the default spec as {}.

As we noticed, the DNS Management is handled by the DNS Operator that implements CoreDNS and the Node Resolver.
On the other hand, the DNS Operator is handled by the ClusterVersionOperator, as we will check a bit later.

### 2. Analysis of CoreDNS in OpenShift 4

CoreDNS is a service providing name resolution of Pod and Service resources within an OpenShift/Kubernetes cluster that implements:

* A/AAAA records for Pod and Service resources
* SRV records for Service resources with named ports
* Used for service discovery

In fact, and within the OpenShift cluster :

* Every Pod is assigned a DNS name: pod-ip.namespace-name.pod.cluster.local
* Every Service is assigned a DNS name: svc-name.namespace-name.svc.cluster.local
* CoreDNS as DaemonSet

Let's see how the implementation is in our test OpenShift cluster.

In the namespace of openshift-dns we can check the DaemonSet and the several pods running the Coredns in each node in our cluster:

* Pods of the Coredns in openshift-dns namespaces:

```bash
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

```bash
oc get ds -n openshift-dns
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
dns-default   12        12        12      12           12          kubernetes.io/os=linux   240d
```

* In one of this pods we can see the process of the coredns running with the config file Corefile:

```bash
$ oc exec -ti dns-default-69cqb -n openshift-dns bash
Defaulting container name to dns.
Use 'oc describe pod/dns-default-69cqb -n openshift-dns' to see all of the containers in this pod.

[root@dns-default-69cqb /]# ps auxww
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.2  0.0 974732 57908 ?        Ssl  Feb03 194:36 coredns -conf /etc/coredns/Corefile
```

#### 2.1 Analysis of Corefile Config file

As we noticed in the previous section, CoreDNS uses Corefile which is the CoreDNS configuration file. This Corefile is managed by the DNS-Operator though a ConfigMap.

```bash
$ oc get cm/dns-default -n openshift-dns -o yaml --export
apiVersion: v1
data:
  Corefile: |
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
```

NOTE: We also take a look to the Corefile config file inside of one of the Coredns pods in the ns of
  openshift-dns to check that in fact is the same:

```bash
# oc exec dns-default-69cqb -n openshift-dns cat /etc/coredns/Corefile
```

* If we check a bit close the Corefile, within the line that start with "kubernetes" we can see:

```bash
kubernetes cluster.local in-addr.arpa ip6.arpa {
```

this enables a plugin that configures that CoreDNS will reply to DNS queries based on IP of the services and pods of Kubernetes.
In other words, this plugin implements the Kubernetes DNS-Based Service Discovery Specification.

For example the cluster.local within this line handle all queries in the custer.local zone (and also in-addr.arpa the reverse dns lookups).

So, as we described early the resources within the OpenShift Cluster uses the CoreDNS for the DNS resolution, and remember that each node has a Coredns pod (controlled by a DaemonSet). We dig in a bit later.

For more information check the [CoreDNS kubernetes plugin documentation](https://coredns.io/plugins/kubernetes/).

* On the other hand, another interesting lines are

```bash
forward . /etc/resolv.conf {
    policy sequential
}

```

this enables the coredns forwarding plugin and define that any queries that are not within the cluster domain of Kubernetes will be forwarded to predefined resolvers (/etc/resolv.conf). So, in other words, by default the forward plugin is configured to use the node’s /etc/resolv.conf to resolve all non-cluster domain names.

Furthermore **sequential** is a policy that selects resolvers based on sequential ordering within /etc/resolv.conf.

* So what's in this resolv.conf in each dns-default/coreDNS pod?

In the case of AWS, it's the resolver of AWS VPC where OpenShift is deployed:

```bash
[root@dns-default-69cqb /]# cat /etc/resolv.conf
search eu-west-1.compute.internal
nameserver 10.0.0.2
```

as we see the search parameter is defined as the aws-region-az.compute.internal, and the nameserver is the 10.0.0.2:

```bash
# nslookup 10.0.0.2 10.0.0.2
2.0.0.10.in-addr.arpa   name = ip-10-0-0-2.eu-west-1.compute.internal.
```

### 3. DNS Resolutions Basics in Openshif4 cluster

So in order to check the resolution we need to have the proper net tools installed (dig, nslookup, tcmdump, etc), and for this purpose we will create a debug pod of rhel-tools that already had installed these tools:

```bash
$ oc debug -t deployment/customer --image registry.access.redhat.c
om/rhel7/rhel-tools
Defaulting container name to customer.
Use 'oc describe pod/customer-debug -n tutorial' to see all of the containers in this pod.

Starting pod/customer-debug ...

sh-4.2# dig -v
DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7
```

#### 3.1 Name resolution in the same namespace

Let's dig in a bit in the Name resolution basics in OpenShift4. We have our namespace "tutorial" and inside we have two pods: debug (created before) and customer (example app), both located in the same namespace.

If we check the connection from debug pod to customer svc with netcat, works liked a charm:

```bash
(debug-pod)sh-4.2# nc -zv customer 8080
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connected to 172.30.188.88:8080.
Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
```

But what happened under the hood?

The procedure for resolution from debug node when queries for customer service is the following:

A. Containers (usually) are specified with the dnsPolicy as ClusterFirst:

```bash
(laptop)$ oc get pod customer-6c7bc489c4-5hg2h  -o jsonpath={.spec.dnsPolicy}
ClusterFirst
```

so this results at the containers with the /etc/resolv.conf as following:

```bash
(debug-pod)sh-4.2# cat /etc/resolv.conf
search tutorial.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal
nameserver 172.30.0.10
options ndots:5
```

in this example, the container "debug" looks-up "customer" using the /etc/resolv.conf

B. Container "debug" queries nameserver 172.30.0.10 for "customer".

C. The query gets proxied to the CoreDNS pod. This is because, the nameserver 172.30.0.10
corresponds to the Coredns pod Service IP.

```bash
(debug-pod)sh-4.2# nslookup 172.30.0.10
10.0.30.172.in-addr.arpa        name = dns-default.openshift-dns.svc.cluster.local.
```

This dns-default.openshift-dns.svc.cluster.local. is the service that loadbalances the several endpoints in the pod:

```bash
(laptop)$ oc get svc -n openshift-dns
NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
dns-default   ClusterIP   172.30.0.10   <none>        53/UDP,53/TCP,9153/TCP   7d1h

(laptop)$ oc describe svc -n openshift-dns | grep TargetPort -A1
TargetPort:        dns/UDP
Endpoints:         10.128.0.5:5353,10.128.2.8:5353,10.128.4.6:5353 + 9 more...
```

if we select one of this endpoints is efectively one Coredns pod:

```bash
(laptop)$ oc get pod -n openshift-dns -o wide | grep 10.128.0.5
dns-default-s9njn   2/2     Running   16         7d1h   10.128.0.5   ip-10-0-144-209.eu-west-1.compute.internal   <none>           <none>
```

D. CoreDNS resolves "customer" to an IP address:

```bash
(debug-pod)sh-4.2# nslookup customer
Server:         172.30.0.10
Address:        172.30.0.10#53

Name:   customer.tutorial.svc.cluster.local
Address: 172.30.188.88

$ oc get svc customer
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
customer   ClusterIP   172.30.188.88   <none>        8080/TCP   47h
```

NOTE: check that nslookup uses CoreDNS 172.30.0.10 as DNS resolver.

A deeper look about the nslookup with the debug enabled shows:

```bash
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

the resolver solved the query using the .tutorial.svc.cluster.local domain in the first place, because of the search option directive (search tutorial.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal).

E. "debug" communicates with "customer" directly and voilà.

#### 3.2 Resolve between different namespaces

In this case we want to resolve the svc of the console of OpenShift:

```bash
sh-4.2# nslookup console
Server:         172.30.0.10
Address:        172.30.0.10#53

** server can't find console: NXDOMAIN
```

what happended? the query is not answered because is not specified the namespace in the same query:

```bash
sh-4.2# nslookup -debug console | grep NXDOMAIN
** server can't find console.tutorial.svc.cluster.local: NXDOMAIN
** server can't find console.svc.cluster.local: NXDOMAIN
** server can't find console.cluster.local: NXDOMAIN
** server can't find console.eu-west-1.compute.internal: NXDOMAIN
** server can't find console: NXDOMAIN
```

To solve this, we need to ensure that the namespace is pointed into the request:

```bash
# nslookup console.openshift-console
Server:         172.30.0.10
Address:        172.30.0.10#53

Name:   console.openshift-console.svc.cluster.local
Address: 172.30.105.179

$ oc get svc -n openshift-console console
NAME      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
console   ClusterIP   172.30.105.179   <none>        443/TCP   7d1h
```

```bash
sh-4.2# nslookup -debug console.openshift-console | grep NXDOMAIN
** server can't find console.openshift-console.tutorial.svc.cluster.local: NXDOMAIN

sh-4.2# nslookup -debug console.openshift-console | grep Name -A1
Name:   console.openshift-console.svc.cluster.local
Address: 172.30.105.179
```

so in this case, the console.openshift-console is not resolve in the first place as declared in the search with tutorial.svc.cluster (remember that was: search tutorial.svc.cluster.local svc.cluster.local cluster.local eu-west-1.compute.internal), but the second occurence resolves correctly - svc.cluster.local with the console.openshift-console.

### 4. DNS Resolution outside of OpenShift

But in the case of the resolution outside the cluster, what are the steps behind the hood?

As we can check, a curl and therefore the resolution of a domain outside of the cluster answers perfectly:

```bash
sh-4.2# curl https://elpais.com -I
HTTP/1.1 200 OK
```

What happened?

The CoreDNS tried to resolved in the domains that the search described, but because have not the proper answer, forwarded the query to the resolver of AWS (remember the line of forward . /etc/resolv.conf and the 10.0.0.2 nameserver of AWS?).

So the flow is:

```bash
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

So that's it! In the [DNS Forwarding blog post](https://rcarrata.com/openshift/dns-forwarding-openshift/), we will use the DNS Operator to configure the DNS Forwarding to a specific private DNS, to try to solve custom domains.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Stay tuned and happy OpenShifting!


<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
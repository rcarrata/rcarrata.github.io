---
layout: single
title: Egress Firewall in OpenShift with OVN Kubernetes plugin
date: 2021-11-10
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Networking", "OpenShift"]
author: rcarrata
comments: true
---

How we can limit the external hosts that some or all pods in an OpenShift project can access from within the cluster? How we can configure an Egress Firewall to only allow specific DNS names and deny the rest of the external internet communication?

Let's dig in!

## 1. Overview

From the version 4.6+ of OpenShift we can use an Egress Firewall to limit the external hosts that some or all pods can access from within the cluster.

An egress firewall supports the following scenarios:

* A pod can only connect to internal hosts and cannot initiate connections to the public internet.
* A pod can only connect to the public internet and cannot initiate connections to internal hosts that are outside the OpenShift Container Platform cluster.
* A pod cannot reach specified internal subnets or hosts outside the OpenShift Container Platform cluster.
* A pod can connect to only specific external hosts.

For example, we can allow one project access to a specified IP range but deny the same access to a different project. Or we can restrict application developers from updating from Python pip mirrors, and force updates to come only from approved sources.

The test on this PoC are executed in a 4.8.17 OpenShift environment.
 
## 2. Deploy example apps and initial tests

Let's first deploy a app of example that will be our entry point for our connectivity tests with the Egress Firewall feature.

* First, generate a new project to execute this tests:

```sh
oc new-project egress-fw-test
```

* Deploy the example app based in hello-openshift image:

```sh
kubectl run --image=quay.io/openshifttest/hello-openshift:multiarch test-egress
pod/test-egress created
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
kubectl exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=9.09 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.29 ms
```

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
kubectl exec -ti test-egress -- ping -c2 1.1.1.1
```

* Test curl to the OpenShift docs webpage:

```sh
kubectl exec -ti test-egress -- curl https://docs.openshift.com -I | head -n1
HTTP/2 200
```

Now we tested two different destination IPs and one DNS name that will be used during this blog post.

## 3. Configure the Egress Firewall to Allow only Google's DNS IP

We can configure an egress firewall policy by creating an EgressFirewall custom resource (CR) object. The egress firewall matches network traffic that meets any of the following criteria:

* An IP address range in CIDR format
* A DNS name that resolves to an IP address
* A port number
* A protocol that is one of the following protocols: TCP, UDP, and SCTP

* Allow only Google DNS in the namespace of egress-test:

```sh
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: default-google
spec:
  egress:
  - type: Allow
    to:
      cidrSelector: 8.8.8.8/32
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

as you can notice the egress.cidrSelector, allows the access only to the 8.8.8.8 IP and deny the rest of IPs (0.0.0.0/0).

* Apply this Egress Firewall rule in the namespace egress-fw-test

```sh
kubectl apply -n egress-fw-test -f egress-fw/ovn/allow-google.yaml
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
kubectl exec -ti test-egress -- ping -c2 8.8.8.8

PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=8.97 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.01 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 8.010/8.489/8.969/0.479 ms
```

this works like a charm because the IP 8.8.8.8 it's allowed in our EgressFirewall object definition, and the Egress Firewall allow this communication.

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
kubectl exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1017ms

command terminated with exit code 1
```

as expected this ping to 1.1.1.1 failed because it's denied by the EgressFirewall.

* Test curl to the OpenShift docs webpage:

```sh
kubectl exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Failed to connect to docs.openshift.com port 443 after 1504 ms: Operation timed out
command terminated with exit code 28
```

## 4. Configure the Egress Firewall to one Allow IP CIDR and one DNS Name and deny the rest

Now we will add also a DNS Name (docs.openshift.com) into the set of rules that will allow defined in the Egress Firewall.

* Add a new rule in the egress spec with the dnsName of docs.openshift.com:

```sh
cat egress-fw/ovn/allow-dns-and-ip.yaml
kind: EgressFirewall
apiVersion: k8s.ovn.org/v1
metadata:
  name: default
spec:
  egress:
  - type: Allow
    to:
      dnsName: docs.openshift.com
  - type: Allow
    to:
      cidrSelector: 8.8.8.8/16
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

This example allows Pods in the egress-fw-test namespace to connect to the host(s) that docs.openshift.com translates to and to any external host within the range 8.8.0.0 to 8.8.255.255.

* Apply the Egress Firewall in the namespace:

```sh
kubectl apply --namespace egress-fw-test -f egress-fw/ovn/allow-dns-and-ip.yaml
```

* Check that the Egress Firewall object is generated properly:

```sh
kubectl get egressfirewalls.k8s.ovn.org default -o yaml
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
...
  name: default
  namespace: egress-fw-test
  resourceVersion: "1045242"
  uid: 25b98c67-fb36-47de-8447-8500eaee2c7e
spec:
  egress:
  - to:
      dnsName: docs.openshift.com
    type: Allow
  - to:
      cidrSelector: 8.8.8.8/32
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
status:
  status: EgressFirewall Rules applied
```

* Test ICMP / Ping to Google's DNS IP (8.8.8.8):

```sh
kubectl exec -ti test-egress -- ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=52 time=8.98 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=52 time=8.50 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 8.501/8.742/8.983/0.241 ms
```

Again as the 8.8.8.8 IP is allowed this works like a charm.

* Test ICMP / Ping to Cloudfare DNS IPs:

```sh
kubectl exec -ti test-egress -- ping -c2 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1010ms

command terminated with exit code 1
```

As in the previous example, the rest of the IPs not allowed specifically will be denied by the Deny rule at the botton applying at all hosts.

* Test curl to the OpenShift docs webpage:

```sh
kubectl exec -ti test-egress -- curl https://docs.openshift.com -I -m2
curl: (28) Resolving timed out after 2000 milliseconds
command terminated with exit code 28
```

What happened? The curl failed! But we allowed the dnsName in the EgressFirewall, why we can't reach the docs.openshift.com webpage from our pod?

Let's check the resolv.conf inside of our pods:

```sh
kubectl exec -ti test-egress -- cat /etc/resolv.conf
search egress-fw-test.svc.cluster.local svc.cluster.local cluster.local ocp.rober.lab
nameserver 172.30.0.10
options ndots:5
```

Seems ok, isn't? Why then the curl is not working properly?

Well, we in fact allowed the dnsName, but who actually resolves the dns resolution is the Openshift DNS based in the CoreDNS.

When a DNS request is performed inside of Kubernetes/OpenShift, CoreDNS handles this request and tries to resolved in the domains that the search describes, but because have not the proper answer, CoreDNS running within the Openshift-DNS pods, forward the query to the external DNS configured during the installation.  

Check the [Deep Dive in DNS in OpenShift for more information](https://rcarrata.com/openshift/dns-deep-dive-in-openshift/).

In our case the rules are allowing only the dnsName, but denying the rest of the IPs, including... the Openshift-DNS / CoreDNS ones!

Let's allow the IPs from the Openshift-DNS, but first we need to check which are these IPs.

```sh
kubectl get pod -n openshift-dns -o wide | grep dns
dns-default-6wx2g     2/2     Running   2          2d1h   10.128.0.4       ocp-8vr6j-master-2         <none>           <none>
dns-default-8hm8x     2/2     Running   2          2d1h   10.130.0.3       ocp-8vr6j-master-1         <none>           <none>
dns-default-bgxqh     2/2     Running   4          2d1h   10.128.2.4       ocp-8vr6j-worker-0-82t6f   <none>           <none>
dns-default-ft6w2     2/2     Running   2          2d1h   10.131.0.3       ocp-8vr6j-worker-0-kvxr9   <none>           <none>
dns-default-nfsm6     2/2     Running   2          2d1h   10.129.2.7       ocp-8vr6j-worker-0-sl79n   <none>           <none>
dns-default-nnlsf     2/2     Running   2          2d1h   10.129.0.3       ocp-8vr6j-master-0         <none>           <none>
```

So we need to add a specific range from the 10.128.0.0 to the 10.130.0.0, but we will add a range a bit larger only for PoC purposes. Let's add a rule to allow the 10.0.0.0/16 cidr:

```sh
  egress:
  - to:
      dnsName: docs.openshift.com
    type: Allow
  - to:
      cidrSelector: 10.0.0.0/16
    type: Allow
  - to:
      cidrSelector: 0.0.0.0/0
    type: Deny
```

Apply the new modified EgressFirewall object with the allow rule described before:

```sh
kubectl apply -f egress-fw/ovn/allow-dns-and-ip-good.yaml
```

And try again the same curl to the docs.openshift.com:

```sh
kubectl exec -ti test-egress -- curl https://docs.openshift.com -I -m2 | head -n1
HTTP/2 200
```

It works!

Using the DNS feature assumes that the nodes and masters are located in a similar location as the DNS entries that are added to the ovn database are generated by the master.

NOTE: use Caution when using DNS names in deny rules. The DNS interceptor will never work flawlessly and could allow access to a denied host if the DNS resolution on the node is different then in the master.

And that's all for the Egress Firewall with OVN Kubernetes plugin in OpenShift.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Stay tuned for the next blog posts and happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
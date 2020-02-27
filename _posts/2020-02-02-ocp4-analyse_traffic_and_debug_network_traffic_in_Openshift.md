---
layout: post
title: Analyse and debug network traffic in Openshift
date: 2020-02-02
type: post
published: true
status: publish
categories:
- Openshift
tags: []
author: rcarrata
comments: true
---

## Analyse and debug your network traffic of your app deployed in Openshift or Kubernetes

Sometimes is hard to analyse what is happening as networking level into your pods deployed in
Openshift or Kubernetes.

How you can debug and/or analyse your network traffic to your application to solve
issues quicker and more effectively? How you can use the well known Wireshark tool as always?

We will be using tcpdump to capture a so-called, PCAP (packet capture) file that will contain the
pod’s network traffic. This PCAP file can then be loaded in a tool like Wireshark to analyze the
traffic and, in this case, the RESTful communication of a service running in a pod.

A sidecar container is a container that is running in the same pod as the actual service/application
and is able to provide additional functionality to the service/application.

### Deploying the sidecar

* Create a new project for testing purposes:

```
$ oc new-project test-delete-rcarrata
```

* Deploy an example application for testing it:

```
$ oc new-app django-psql-example
$ oc get pod
NAME                           READY   STATUS              RESTARTS   AGE
django-psql-example-1-build    0/1     Completed           0          3m4s
django-psql-example-1-deploy   0/1     Completed           0          74s
django-psql-example-1-j4w28    1/1     Running             0          65s
django-psql-example-2-deploy   0/1     ContainerCreating   0          4s
postgresql-1-2q9h7             1/1     Running             0          2m49s
postgresql-1-deploy            0/1     Completed           0          2m57s
```

* Fetch the deploymentconfig of the django-psql-example:
```
$ oc get dc django-psql-example -o yaml > django-psql-tcpdump.yaml
```

* Into the deploymentconfig add the container that you want to do tcpdump:

```
- name: tcpdump
  image: corfr/tcpdump
  command:
    - /bin/sleep
    - infinity
```

* In the case for the django app, the sidecar will be into the container spec:

```
spec:
  containers:
  - name: tcpdump
    image: corfr/tcpdump
    command:
      - /bin/sleep
      - infinity
  - env:
    - name: DATABASE_SERVICE_NAME
      value: postgresql
```

This will spin up an additional container with a sidecar, that you can execute tcpdump to capture and for further analysis the several
packets that are receiving / sending the django container (remember that tcpdump and django containers are in the same pod).

* Apply the sidecar deploymentconfig django psql:

```
$ oc apply -f django-psql-example.yaml
deploymentconfig.apps.openshift.io/django-psql-example configured

$ oc get pod -w
NAME                           READY   STATUS              RESTARTS   AGE
django-psql-example-1-build    0/1     Completed           0          3m24s
django-psql-example-1-deploy   0/1     Completed           0          94s
django-psql-example-2-deploy   1/1     Running             0          24s
django-psql-example-2-gfws6    0/2     ContainerCreating   0          8s
postgresql-1-2q9h7             1/1     Running             0          3m9s
postgresql-1-deploy            0/1     Completed           0          3m17s
django-psql-example-2-gfws6   0/2   ContainerCreating   0     8s
django-psql-example-2-gfws6   1/2   Running   0     10s
django-psql-example-2-gfws6   2/2   Running   0     14s
django-psql-example-2-deploy   0/1   Completed   0     30s
django-psql-example-2-deploy   0/1   Completed   0     30s
```

### Capturing and analyzing traffic

With the sidecar deployed and running, we can now start capturing data

* Log in into the tcpdump container:

```
~ $ oc rsh -c tcpdump django-psql-example-2-gfws6
~ $ tcpdump -s 0 -n -w /tmp/example.pcap
tcpdump: eth0: You don't have permission to capture on that device
(socket: Operation not permitted)
```

What happened? Due to the SCCs, the tcpdump is not capable to capture the packets into the eth0
because the container have not the proper scc permissions.

* To avoid that you need to add a specific cluster-admin permissions to the default Service Account of the namespace
with the anyuid scc:

```
oc adm policy add-scc-to-user anyuid -z default -n `oc project -q` --as=system:admin
```

IMPORTANT: This is could cause a security issue, because any pod can run as root so be careful and only implement
this in the testing namespaces, or in the namespaces that are controlled by the cluster-admin and
being noticed that security capabilities disabled.

* Rollout the deploymentconfig for deploy with the proper scc:

```
oc rollout latest dc django-psql
```

* Inside of the container of tcpdump of the pod that we deployed before (django-psql-example
  sidecard) execute the tcpdump:

```
$ tcpdump -s 0 -n -w /tmp/example.pcap

```

* Generate requests for this applications that will be captured by the tcpdump sidecar:

```
$ curl django-psql-example-test-delete-rcarrata.apps.ocp4.rcarrata.com -I
HTTP/1.1 200 OK
Server: gunicorn/19.4.5
Date: Thu, 27 Feb 2020 19:43:32 GMT
Content-Type: text/html; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 18255
Set-Cookie: 320587f6606431b421a7ed809db87323=ec4dec0bb6e99d5a3aaed6dd165eaa51; path=/; HttpOnly
Cache-control: private
```

* Control+C the tcpdump command to exit and see how many packets are captured:

```
$ tcpdump -s 0 -n -w /tmp/example.pcap

  tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
  ^C574 packets captured
  574 packets received by filter
  0 packets dropped by kernel
```

* Copy the example.pcap to your localhost:

```
$ oc cp -c tcpdump django-psql-example-2-gfws6 :/tmp/example.pcap example.pcap
```

* Examine the pcap with wireshark... and voilà! You can analyse your your network traffic!

```
$ wireshark example.pcap
```

This is very useful for debugging and for see connectivity and app issues within external systems,
or within interaction with other pods.


---
layout: post
title: Load Balancing and DNS with Openshift IPI in VMware & On-Premise platforms
date: 2020-01-02
type: post
published: true
status: publish
categories:
- Openshift
tags: []
author: rcarrata
comments: true
---

How is managed the LoadBalancers and the DNS in Openshift IPI in VMWARE and the rest of On-Premises platforms such as Baremetal, Openstack, or RHV? What are the new elements introduced and how are integrating within the cluster?

Let's dig in!

## Overview

IPI for on-premises platforms (Installer Provisioned Infrastructure) is scoped to automate a number of networking infrastructure requirements that are handled on other platforms by cloud infrastructure services.

From Openshift version 4.5 a new IPI installation is introduced for vSphere, automating the installation in these platforms, becoming easier and faster in comparation of the traditional UPI mode available from version 4.2.

But with the introduction of the of the IPI mode, a new way of doing DNS and load balancing for the api, api-int, DNS and ingress endpoints (apps) is used.

This method is based in the kubernetes native infrastructure concepts, and its used as well in Openstack IPI (4.2), RHV IPI (4.4) and BM IPI (4.4).

The main goal of this blog post, is to show the differences between IPI and UPI installations in VMWARE and also between cloud and on-premises platforms (RHV IPI, OSP IPI, BM IPI and VMWARE IPI).

## 1. Scenario

This blog post is developed and tested in a VMWARE vCenter v6.7 and with Openshift 4.5.

As said before, this is not an [installation procedure of IPI VMWARE](https://docs.openshift.com/container-platform/4.5/installing/installing_vsphere/installing-vsphere-installer-provisioned-network-customizations.html) , its the analysis of the Load Balancer and DNS resources in IPI On-Premise environments, to show the differences between Cloud environments (IPI and UPI) and the IPI On-premise environments (VMWARE, RHV, Openstack and BareMetal).

The scenario is installed by the IPI mode in the vCenter v6.7 and have 6 nodes (3 masters and 3 workers) within the Openshift 4.5 cluster.

## 2. Openshift IPI Load Balancer and DNS

As we described before a new way of doing DNS and load balancing for Openshift Cluster is introduced in IPI On-premises mode.

To start our analysis we will divide the different components of LB and DNS in IPI in three main sections:

* Control Plane Access Load Balancer
* Ingress Load Balancer
* Internal DNS


## 3. Control Plane Access Load-Balancer

The Control Plane Access refers to the access to the Kubernetes/Openshift API Server (port 6443) from clients both external and external to the cluster, that should be load balanced across control plane machines.

Furthermore, also is needed to access to the Ignition config files served by the Machine Config Server in the port 22623, from clients WITHIN the cluster (not from clients outside the OCP4 cluster) that should be also load-balanced accross control plane machines (masters).

In both cases, the installation of the IPI process (and also in Baremetal and other UPIs) expects
these ports to be reachable on the bootstrap VM first and then later on the newly-deployed control
plane machines.

On AWS/Azure/GCP an external load-balancer is required to be configured in advance in order to
provide this access.

### 3.1 API VIP (Virtual IP)

Here is where a Virtual IP (virtual IP) becomes very handy. In the VMware platform (and also in
every IPI on-premise, including RHV, and Baremetal) a VIP is used to provide failover of the API
server accross the control plane machines (including the own bootstrap VM).

This API VIP is defined by the user in the install-config.yaml in the installation process, and the installation process configures a keepalive to manage this VIP.

In our lab we selected the 10.0.0.200 as the VIP address for the API.

```
[Insert Image]
```

These keepalive instances are run as [static pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/) and are resources for deploy them into the cluster are redenred by the Machine Config Operator.

The static pods are managed directly by the kubelet daemon on a specific node, without the API server observing them.
Also Static Pods are always bound to one Kubelet on a specific node. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.

In our case in each node of the IPI installation, the static pods are running with [Filesystem-hosted static pod manifests](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/#configuration-files), using the folder of /etc/kubernetes/manifests and /etc/kubernetes/static-pod-resources as the resources for kubelet:

```
[root@vmware-nwjr2-master-0 /]# ls -lrht /etc/kubernetes/ | egrep 'manifests|static'
drwxr-xr-x.  2 root root  244 Nov 18 10:00 manifests
drwxr-xr-x. 29 root root 4.0K Dec 25 00:18 static-pod-resources
```
and in the manifests there is the yaml used for create and manage the static pods for this specific node:

```
[root@vmware-nwjr2-master-0 /]# ls -lrht /etc/kubernetes/manifests/
total 64K
-rw-r--r--. 1 root root  21K Oct 23 17:09 etcd-pod.yaml
-rw-r--r--. 1 root root 5.8K Oct 23 17:16 kube-controller-manager-pod.yaml
-rw-r--r--. 1 root root 3.5K Oct 23 17:16 kube-scheduler-pod.yaml
-rw-r--r--. 1 root root  697 Nov 18 10:00 recycler-pod.yaml
-rw-r--r--. 1 root root 2.9K Nov 18 10:00 coredns.yaml
-rw-r--r--. 1 root root 4.0K Nov 18 10:00 haproxy.yaml
-rw-r--r--. 1 root root 3.2K Nov 18 10:00 keepalived.yaml
-rw-r--r--. 1 root root 2.6K Nov 18 10:00 mdns-publisher.yaml
-rw-r--r--. 1 root root 5.4K Dec 25 00:15 kube-apiserver-pod.yaml
```

and specifically we are interest in the ones that helps us to manage the LoadBalancers and DNS:

```
[root@vmware-nwjr2-master-0 /]# ll /etc/kubernetes/manifests/ | egrep -i 'dns|haproxy|keepalivd'
-rw-r--r--. 1 root root  2927 Nov 18 10:00 coredns.yaml
-rw-r--r--. 1 root root  3997 Nov 18 10:00 haproxy.yaml
-rw-r--r--. 1 root root  3184 Nov 18 10:00 keepalived.yaml
-rw-r--r--. 1 root root  2592 Nov 18 10:00 mdns-publisher.yaml
```

### 4. Keepalive in Openshift IPI

As we described earlier, Keepalive is used to ensure that the API (and also de Ingress .apps) Virtual IPs (VIP) are always available.

Keepalive utilices a protocol called [Virtual Router Redundancy Protocol](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) or VRRP that determine the node health and elect an IP owner, and have several requirements:

* Only one host own the IP at any time
* All nodes have equal priority
* Failover can take several seconds

Node health is checked every one second. There is separated checks for each service, one for the API and another separated for the Ingress services.

##### Deep Dive in Keepalived in OCP IPI

As we discussed before, Keepalived runs as static pods inside of our Openshift cluster, but also we can check the pods running with the OCP API in the namespace of "openshift-vsphere-infra":

```
[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra | grep keepalive
keepalived-vmware-nwjr2-master-0           2/2     Running   0          70d
keepalived-vmware-nwjr2-master-1           2/2     Running   0          70d
keepalived-vmware-nwjr2-master-2           2/2     Running   0          70d
keepalived-vmware-nwjr2-worker-d47rb       2/2     Running   0          70d
keepalived-vmware-nwjr2-worker-f5zbm       2/2     Running   0          70d
keepalived-vmware-nwjr2-worker-v5vvn       2/2     Running   0          70d
```

As we can see the keepalived pods are running in every node of our standard cluster (3 masters and 3 workers):

```
[root@ocp-bastion ~]# oc get nodes
NAME                        STATUS     ROLES    AGE   VERSION
vmware-nwjr2-master-0       Ready      master   70d   v1.18.3+47c0e71
vmware-nwjr2-master-1       Ready      master   70d   v1.18.3+47c0e71
vmware-nwjr2-master-2       Ready      master   70d   v1.18.3+47c0e71
vmware-nwjr2-worker-d47rb   Ready      worker   70d   v1.18.3+47c0e71
vmware-nwjr2-worker-f5zbm   Ready      worker   70d   v1.18.3+47c0e71
vmware-nwjr2-worker-v5vvn   Ready      worker   70d   v1.18.3+47c0e71
```

This are running not as DaemonSet as we can expected, are running as static pods as we discussed before, managed by kubelet:

```
[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra keepalived-vmware-nwjr2-master-0  -o yaml  | grep manager
 manager: kubelet
```

These keepalive pods have their configuration based in the config files defined in the static-pods-resources in the node, as a hostPath:

```
[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra keepalived-vmware-nwjr2-master-0  -o json | jq .spec.volumes[0]
{
  "hostPath": {
    "path": "/etc/kubernetes/static-pod-resources/keepalived",
    "type": ""
  },
  "name": "resource-dir"
}
```

And manages as we can check the two VIPs defined and needed for the IPI installation of Openshift:

```
[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra keepalived-vmware-nwjr2-master-0  -o json | jq .spec.containers[1].command
[
  "dynkeepalived",
  "/etc/kubernetes/kubeconfig",
  "/config/keepalived.conf.tmpl",
  "/etc/keepalived/keepalived.conf",
  "--api-vip",
  "10.0.0.200",
  "--ingress-vip",
  "10.0.0.201"
]
```

the API VIP (10.0.0.200) and the Ingress VIP (10.0.0.201) are managed by this keepalive pod (the same example will be happening in the other nodes).

Furthermore all the static pods of the keepalived are running with privileged mode and within hostNetwork:

```
[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra keepalived-vmware-nwjr2-master-0  -o json | jq .spec.hostNetwork
 true

[root@ocp-bastion ~]# oc get pod -n openshift-vsphere-infra keepalived-vmware-nwjr2-master-0  -o json | jq .spec.containers[0].securityContext
{
  "privileged": true
}
```

If we check the static-pods-resources inside the node, and check the conf we can observe some interesting things:

```
[root@ocp-bastion ~]# oc debug node/vmware-nwjr2-master-0
Creating debug namespace/openshift-debug-node-bmxpf ...
sh-4.2# chroot /host bash

[root@vmware-nwjr2-master-0 /]# ls -lhrt /etc/kubernetes/manifests/keepalived.yaml
-rw-r--r--. 1 root root 3.2K Nov 18 10:00 /etc/kubernetes/manifests/keepalived.yaml
```

the manifest of the keepalive is stored in /etc/kubernetes/manifests along with the rest of the static pods running by the kubelet.

On the other hand, the config file for keepalive is stored in /etc/kubernetes/static-pod-resources and used in the command of the container pod spec:

```
[root@vmware-nwjr2-master-0 /]# ls -lrht /etc/kubernetes/static-pod-resources/keepalived/keepalived.conf.tmpl
-rw-r--r--. 1 root root 1.3K Nov 18 10:00 /etc/kubernetes/static-pod-resources/keepalived/keepalived.conf.tmpl
```

So, the keepalived.conf.tmpl is used as a templated and is rendered with the specifications (ip addresses, services to monitor, etc) and stored once is rendered in the /etc/keepalive/keepalive.conf

```
[root@vmware-nwjr2-master-0 /]# ls -lrht /etc/keepalived/keepalived.conf
-rw-r--r--. 1 root root 1.1K Dec 17 13:40 /etc/keepalived/keepalived.conf
```

TIP: Check out the container keepalive command that uses the tmpl as a template and renders this latest keepalive.conf.

Finally if we check the rendered config file, we can see the 2 VRRPs instances that keepalived uses to monitor the VIPs of the API and the Ingress with their virtual ip address and the interfaces:

```
[root@vmware-nwjr2-master-0 /]# cat /etc/keepalived/keepalived.conf
vrrp_script chk_ocp {
    script "/usr/bin/curl -o /dev/null -kLfs https://localhost:6443/readyz && /usr/bin/curl -o /dev/null -kLfs http://localhost:50936/readyz"
    interval 1
    weight 50
}

# TODO: Improve this check. The port is assumed to be alive.
# Need to assess what is the ramification if the port is not there.
vrrp_script chk_ingress {
    script "/usr/bin/curl -o /dev/null -kLfs http://localhost:1936/healthz"
    interval 1
    weight 50
}

vrrp_instance vmware_API {
    state BACKUP
    interface ens192
    virtual_router_id 109
    priority 40
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass vmware_api_vip
    }
    virtual_ipaddress {
        10.0.0.200/24
    }
    track_script {
        chk_ocp
    }
}

vrrp_instance vmware_INGRESS {
    state BACKUP
    interface ens192
    virtual_router_id 124
    priority 40
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass vmware_ingress_vip
    }
    virtual_ipaddress {
        10.0.0.201/24
    }
    track_script {
        chk_ingress
    }
}
```

And with that configuration Keepalived ensures that the VIPs managed are always monitored and responding with nodes responding to the different services.

We head the end of this first blog post about the LB and DNS in Openshift IPI for On-Premises platforms, focusing in VMWARE case.

In the next blog post, we will analyse how is the Load Balancing handled and analyse in deep the configurations specifics.

Happy Openshifting and Happy New Years folks!

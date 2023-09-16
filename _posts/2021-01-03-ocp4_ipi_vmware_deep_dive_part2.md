---
layout: single
title: Load Balancing and DNS with OpenShift IPI in VMware & On-Premise (Part II)
date: 2021-01-03
type: post
published: true
status: publish
categories:
- OpenShift
- Networking
- Administration
- Kubernetes
- Virtualization
tags: []
author: rcarrata
comments: true
---

This is the second blog post of Load Balancing and DNS with OpenShift IPI series .

Check the earlier posts:
* [Part I - Load Balancing and DNS with OpenShift IPI](https://rcarrata.com/openshift/ocp4_ipi_vmware_deep_dive/)

### 5. Load Balancer in OpenShift IPI

As we checked in the previous section, the Keepalived helps to manage the VIPs for the API OpenShift services used in our OpenShift cluster.

But the thing is... who is in charge to Load Balancing between the different masters of our cluster?

The well-know **Haproxy**.

#### 5.1 Deep Dive Haproxy for API Load Balancer

Haproxy is also used as a Load Balancer for our Routers in OCP, since OpenShift 3 and is well-known by their performance and flexibility acting as a Load Balancer and Ingress for our applications within the cluster.

And in IPI mode, a **separated instance of Haproxy** its also used for provide Load Balancing for our API and Ingress services.

Let's check the Haproxy used in IPI to see what's happening behind the hood.

The Haproxy are running only in the masters also as an static pods:

```bash
[root@ocp-bastion ~]# kubectl get pod -n openshift-vsphere-infra | grep haproxy
haproxy-vmware-nwjr2-master-0              2/2     Running   0          71d
haproxy-vmware-nwjr2-master-1              2/2     Running   0          71d
haproxy-vmware-nwjr2-master-2              2/2     Running   0          71d
```

As we can check into the haproxy pods the config files are mounted as a hostPath, with the static-pod-resources and the /etc/haproxy folders:

```bash
[root@ocp-bastion ~]# kubectl get pod -n openshift-vsphere-infra haproxy-vmware-nwjr2-master-0 -o json | jq .spec.volumes[0]
{
  "hostPath": {
    "path": "/etc/kubernetes/static-pod-resources/haproxy",
    "type": ""
  },
  "name": "resource-dir"
}

[root@ocp-bastion ~]# kubectl get pod -n openshift-vsphere-infra haproxy-vmware-nwjr2-master-0 -o json | jq .spec.volumes[3]
{
  "hostPath": {
    "path": "/etc/haproxy",
    "type": ""
  },
  "name": "conf-dir"
}
```

Also an interesting part of the pod definition yaml of the Haproxy pod is the command executed in the second container (the first is an haproxy config reloader):

```bash
[root@ocp-bastion ~]# kubectl get pod -n openshift-vsphere-infra haproxy-vmware-nwjr2-master-0 -o json | jq .spec.containers[1].command
[
  "monitor",
  "/var/lib/kubelet/kubeconfig",
  "/config/haproxy.cfg.tmpl",
  "/etc/haproxy/haproxy.cfg",
  "--api-vip",
  "10.0.0.200"
]
```

as we can check the haproxy is using the config file in /etc/haproxy/haproxy.cfg rendered from the haproxy.cfg.tmpl located in the /etc/kubernetes/static-pod-resources

```bash
[root@vmware-nwjr2-master-0 /]# ls -lrht /etc/kubernetes/static-pod-resources/haproxy/
total 4.0K
-rw-r--r--. 1 root root 918 Nov 18 10:00 haproxy.cfg.tmpl
```

If we analyse the lines more significant about the config file of haproxy we discover some interesting and useful things:

```bash
[root@vmware-nwjr2-master-0 /]# grep -A2 frontend /etc/haproxy/haproxy.cfg
frontend  main
  bind :::9445 v4v6
  default_backend masters
```

the frontend its exposed in the 9445 port for both ipv4 and ipv6 and have as default backends the masters (we will see that the routers of OpenShift are balanced not using this haproxy instance, this haproxy is ONLY for the API).

Also we can analyze the backend masters in the same config file:

```bash
[root@vmware-nwjr2-master-0 /]# grep -wi -A10 "backend masters" /etc/haproxy/haproxy.cfg
backend masters
   option  httpchk GET /readyz HTTP/1.0
   option  log-health-checks
   balance roundrobin
   server vmware-nwjr2-master-0 10.0.0.34:6443 weight 1 verify none check check-ssl inter 3s fall 2 rise 3
   server vmware-nwjr2-master-2 10.0.0.90:6443 weight 1 verify none check check-ssl inter 3s fall 2 rise 3
   server vmware-nwjr2-master-1 10.0.0.99:6443 weight 1 verify none check check-ssl inter 3s fall 2 rise 3
```

in this backend we can figure out some things:

* the backends are the 3 masters of the OpenShift cluster, specifically the port 6443 that correspond to the OpenShift/Kubernetes API server.
* The API is balanced using the roundrobin protocol
* The weight defined in the backends are the same, so each master is equally load balanced (no priority neither weight is used).

#### 5.2 Haproxy, KeepAlived and VRRP

As we discussed before the Keepalived uses VRRP as the protocol for determine the node health and elect the IP owner.

But how the Keepalived uses the VRRP for monitor VIPs and also how is linked with the Haproxy?

First we need to check again the Keepalived keepalived.tmpl used as a template (we focused only in the API checks and vrrp in this config file, in the next blog post we will analyze the Ingress checks and vrrps):

```bash
[core@vmware-nwjr2-master-0 ~]$ cat /etc/kubernetes/static-pod-resources/keepalived/keepalived.conf.tmpl
vrrp_script chk_ocp {
    script "/usr/bin/curl -o /dev/null -kLfs https://localhost:6443/readyz && /usr/bin/curl -o /dev/null -kLfs http://localhost:50936/readyz"
    interval 1
    weight 50
}

vrrp_instance {{ .Cluster.Name }}_API {
    state BACKUP
    interface {{ .VRRPInterface }}
    virtual_router_id {{ .Cluster.APIVirtualRouterID }}
    priority 40
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass {{ .Cluster.Name }}_api_vip
    }
    virtual_ipaddress {
        {{ .Cluster.APIVIP }}/{{ .Cluster.VIPNetmask }}
    }
    track_script {
        chk_ocp
    }
}
```

In the Keepalived config file we see that the script have one track_script with the name "chk_ocp" that checks two main urls:

* https://localhost:6443/readyz
* http://localhost:50936/readyz

The first part of the check is clear, is probing the readiness of the ocp/k8s api exposed in the port 6443.

```bash
[root@vmware-nwjr2-master-0 ~]# curl -kLfs https://localhost:6443/readyz
ok
```

The second part is the interesting one. Let's see.

First, let's discover what's listening in that port:

```bash
[root@vmware-nwjr2-master-0 ~]# ss -laputn | grep 50936 | grep LISTEN
tcp   LISTEN    0      128                       *:50936                      *:*            users:(("haproxy",pid=5594,fd=10)
```

Seems to be the own Haproxy! Let's check the haproxy.cfg with the grep of the specific port:

```bash
[root@vmware-nwjr2-master-0 ~]# cat /etc/haproxy/haproxy.cfg | grep -B1 -A3 50936
listen health_check_http_url
  bind :::50936 v4v6
  mode http
  monitor-uri /readyz
  option dontlognull
```

The haproxy is listening (bind :::50936) to the port 50936 and have the line of monitor-uri /readyz configured.

So, in conclusion the vrrp script checks the http://localhost:50936/readyz that is the haproxy health_check for determine if the haproxy is running in this node in a **healthy state**.

```bash
[root@vmware-nwjr2-master-0 ~]# curl -kLfs http://localhost:50936/readyz
<html><body><h1>200 OK</h1>
Service ready.
</body></html>
```

If one of the two checks is not successful more than 3 times x the advent_time (configured 1 second ), track_script of the vrrp fails and the keepalived considers that the service of this node failed and switchs the virtual_address to the other master.

```bash
[root@vmware-nwjr2-master-0 ~]# cat /etc/keepalived/keepalived.conf | egrep -A16 "vrrp_instance vmware_API" | grep -A2 virtual_ipaddress
    virtual_ipaddress {
        10.0.0.200/24
    }
```

Finally, if we pay attention in the keepalived config file  are that all nodes within the API vrrp instance instance have the same priority, and weight based in the same template (located in the static-pod-resources) and for this reason the load balancing of the VIP are treated equally in terms of priority.

```bash
[root@ocp-bastion manu]# for i in $(kubectl get nodes -o wide | grep master | awk '{ print $6 }');do ssh -i id_rsa core@$i "hostname && cat /etc/keepalived/keepalived.conf | egrep 'priority|weight'"; done
vmware-nwjr2-master-0
    weight 50
    weight 50
    priority 40
    priority 40
vmware-nwjr2-master-1
    weight 50
    weight 50
    priority 40
    priority 40
vmware-nwjr2-master-2
    weight 50
    weight 50
    priority 40
    priority 40
```

For more information about the VRRP and Keepalived, check [this useful article](https://www.redhat.com/sysadmin/advanced-keepalived) about Advanced Keepalived.

#### 5.3 API VIP status and VRRP

Let's see where the VIP is located in our case with a little for loop checking the ip addresses in each master node:

```bash
[root@ocp-bastion]# for i in $(kubectl get nodes -o wide | grep master | awk '{ print $6 }');do ssh -i id_rsa core@$i "hostname && ip -br a | grep ens"; done
vmware-nwjr2-master-0
ens192           UP             10.0.0.34/24 fe80::4e59:e2d1:e8f6:f78c/64
vmware-nwjr2-master-1
ens192           UP             10.0.0.99/24 fe80::f65c:b6fa:d244:60fe/64
vmware-nwjr2-master-2
ens192           UP             10.0.0.90/24 10.0.0.200/24 fe80::d39c:82a6:1f15:b8c8/64
```

Check the ens192 of the master2, have an ip address more than the others, right?

If we check specifically this master2, we can deep dive a bit in the interface ens192:

```bash
[root@vmware-nwjr2-master-2 /]# ip ad | grep -A3 200
    inet 10.0.0.200/24 scope global secondary ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::d39c:82a6:1f15:b8c8/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

As we can see in the master-2 of our cluster beside of the primary IP from the node 10.0.0.x/24, is assigned a secondary IP that corresponds with our VIP assigned (10.0.0.200/24).

##### Haproxy & Keepalived request flow

So, now that we have all the pieces involved explained, let's check again the flow of a request to the Haproxy showed in the last blog:

[![](/images/vmware_ipi_3.png "Keepalive Diagrams")]({{site.url}}/images/vmware_ipi_3.png)

* The VRRP protocol is used by Keepalived to determine the node health and elect an IP owner
* The node health are checked every second for each service (separated checks, one for API and another for INGRESS)
* The ARP is used to associate the VIP with the owner node's interface (masters)
* Active Node uses RARP (Reverse ARP) to claim traffic
* Active node passes traffic to endpoint

### 6. API Load Balancer

Finally we will analyze a diagram of how the API Load Balancer works in OpenShift IPI with all the elements discussed in our blog posts:

[![](/images/vmware_ipi_4.png "Keepalive Diagrams")]({{site.url}}/images/vmware_ipi_4.png)

1. The client creates a new request to api.ocp4.example.com
2. The pod of Haproxy on the actively host the API IP address (as determined by keepalived) load balances across control plane nodes (masters) using round robin protocol.
3. The Connection if forwarded to the chosen control plane node
4. The control plane node responds directly to the client, with [Direct Return](https://www.haproxy.com/blog/layer-4-load-balancing-direct-server-return-mode/)

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

And that's it for this blog post! 

Check out the [second part of this blog post series](https://rcarrata.com/openshift/ocp4_ipi_vmware_deep_dive_part2/), where we will analyze how is the load balancing for the Ingress (.apps) for OpenShift IPI.

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
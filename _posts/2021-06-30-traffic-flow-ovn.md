---
layout: post
title: Monitoring and analysis of Network Flow Traffic in OpenShift (Part I)
date: 2021-06-30
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How can we monitor the traffic in and out of our OpenShift cluster in an easy way? How can we do an analysis about the network traffic flows of our workloads in our OpenShift SDN? Let's start to dig in! 

## Overview

Until the 4.8 release, the network traffic analysis / monitoring within the OpenShift SDN was hard to perform. If you wanted to know the network flows of your traffic of your workloads in the SDN you needed to analyse node to node the Openflow (protocol designed to allow network controllers to determine the path
) and the Flow tables from the OVS pods, using utilities like ovs-ofctl to extract the flow entries in the flow table.

For example to check the flow entries with ovs-ofctl:

```
# ovs-ofctl dump-flows -O OpenFlow13 br0 |head -2 | tail -1
 cookie=0x0, duration=135692.072s, table=0, n_packets=0, n_bytes=0, priority=250,ip,in_port=2,nw_dst=224.0.0.0/4 actions=drop
```

Furthermore you needed to check the vSwitches of OVS (br0) to check the ports, vxlans (even we only used vxlan0 in OpenShift), and the tun0 among others:

```
# ovs-ofctl show -O OpenFlow13 br0 
OFPT_FEATURES_REPLY (OF1.3) (xid=0x2): dpid:0000e6172964fa4b
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS GROUP_STATS QUEUE_STATS
OFPST_PORT_DESC reply (OF1.3) (xid=0x3):
 1(vxlan0): addr:5a:3c:e3:28:0f:06
 [...]
```

Putting all of this together in order to analyse and monitor our traffic was not easy, neither straightforward. We need to [know which table](https://github.com/openshift/origin/blob/release-3.11/pkg/network/node/ovscontroller.go) is used for in each case and try to put together all pieces for debug and analyse our traffic.

## Tracking Network Flows 

One very cool feature in my opinion in next OpenShift 4.8 is the ability to track and monitor OVS flows for networking analytics using **OVN Kubernetes CNI plugin**.

As the [Tracking network flows documentation](https://docs.openshift.com/container-platform/4.10/networking/ovn_kubernetes_network_provider/tracking-network-flows.html) of OpenShift (still in preview) refers you can collect information about pod network flows from your cluster to assist with tasks like:

* Monitor ingress and egress traffic on the pod network.

* Troubleshoot performance issues.

* Gather data for capacity planning and security audits.

When you enable the collection of the network flows, only the metadata about the traffic is collected. And this data is collected using different possible network record formats:

* **NetFlow**: feature that was introduced on Cisco routers that provides the ability to collect IP network traffic as it enters or exits an interface. 
* **sFlow**: or sampled flow is an industry standard for packet export at Layer 2 of the OSI model. It provides a means for exporting truncated packets, together with interface counters for the purpose of network monitoring.
* **IPFIX**: is an IETF protocol that defines how IP flow information is to be formatted and transferred from an exporter to a collector.

When you configure the Cluster Network Operator (CNO) with one or more collector IP addresses and port numbers, the Operator configures Open vSwitch (OVS) on each node to send the network flows records to each collector.

## Implementing Tracking Network Flow in OpenShift 4.8+

### PreRequirements 

For our tests we have our cluster of 4.8 (fc7), with OVN Kubernetes as CNI plugin:

```
$ kubectl get clusterversion
NAME      VERSION      AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.8.0-fc.7   True        False         28h     Cluster version is 4.8.0-fc.7
```

```
$ kubectl get network cluster -o jsonpath='{.status.networkType}'
OVNKubernetes
```

### OSS Options for Network Flow Collector

As the documentation says, we need to apply the manifest for the CNO to configure the OVS to send the network flow records to one Network Flow collector.  

But what we can choose as options to collect our Network Flows?

After some research we found 3 options available:

* [GoFlow](https://github.com/cloudflare/goflow)
* [vFlow](https://github.com/VerizonDigital/vflow)
* [ElastiFlow](https://github.com/robcowart/elastiflow) 

GoFlow and vFlow are both written in Go which I like and both appear to be highly scalable collectors. What they don’t have are built-in front ends so will require a bit more work to extract interesting data.

### Using GoFlow to collect our Network Flow data 

Let's start with GoFlow and try collecting some Network Flows from our cluster. As we said before this application is a NetFlow/IPFIX/sFlow collector in Go.
It gathers network information (IP, interfaces, routers) from different flow protocols

We will use a Centos8 Stream as our Network Flow Collector node running our GoFlow. 

* Download the latest goflow rpm and installed in the centos8 vm:

```
# wget https://github.com/cloudflare/goflow/releases/download/v3.4.2/goflow-3.4.2-1.x86_64.rpm
```

```
# yum install goflow-3.4.2-1.x86_64.rpm -y
```

* Start the GoFlow in NetworkFlow and SFlow modes enabled without sending to Kafka the events:

```
# goflow -kafka=false -sflow=true -nf=true
INFO[0000] Starting GoFlow
INFO[0000] Listening on UDP :2056                        Type=NetFlowLegacy
INFO[0000] Listening on UDP :6343                        Type=sFlow
INFO[0000] Listening on UDP :2055                        Type=NetFlow
```

* Grab the ip where GoFlow is running:

```
# ip ad show eth0 | grep -i "inet" | head -n 1 | awk '{print $2}'
192.168.50.219/24
```

### Configure OpenShift to Tracking Network Flows

Create a patch file that specifies the network flows collector type and the IP address and port information of the collectors:

```
cat networkflow.yaml
spec:
  exportNetworkFlows:
    netFlow:
      collectors:
        - 192.168.50.219:2056
```

as you noticed, we're using the netFlow format and using also the port of NetFlowLegacy 2056 as shown in the documentation.

* Patch the CNO with the network flows collectors configuration defined before:

```
$ kubectl patch network.operator cluster --type merge -p "$(cat networkflow.yaml)"
```

* View the Operator configuration to confirm that the exportNetworkFlows field is configured

```
$ kubectl get network.operator cluster -o jsonpath="{.spec.exportNetworkFlows}"
{"netFlow":{"collectors":["192.168.50.219:2056"]}}
```

* View the network flows configuration in OVS from each node:

```
for pod in $(oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-node -o jsonpath='{range@.items[*]}{.metadata.name}{"\n"}{end}'); do echo ; echo $pod ; oc -n openshift-ovn-kubernetes exec -c ovnkube-node $pod  -- bash -c 'for type in ipfix sflow netflow ; do ovs-vsctl find $type ; done'; done

ovnkube-node-6m56c
_uuid               : bd93e98f-8ff2-488e-a0c2-b073cffb1b74
active_timeout      : 60
add_id_to_interface : false
engine_id           : []
engine_type         : []
external_ids        : {}
targets             : ["192.168.50.219:2056"]

ovnkube-node-9dfhz
_uuid               : 1ed3ed4f-c921-46fd-af7a-11a75c9b2cbe
active_timeout      : 60
add_id_to_interface : false
engine_id           : []
engine_type         : []
external_ids        : {}
targets             : ["192.168.50.219:2056"]
```
### Checking the GoFlow Collector network flow records

Almost immedialely we observe in our goflow logs running in Centos8 tons of Network Flows running: 

```
# goflow -kafka=false -sflow=true -nf=true
INFO[0000] Starting GoFlow
INFO[0000] Listening on UDP :2056                        Type=NetFlowLegacy
INFO[0000] Listening on UDP :6343                        Type=sFlow
INFO[0000] Listening on UDP :2055                        Type=NetFlow

Type:NETFLOW_V5 TimeReceived:1624985192 SequenceNum:4298 SamplingRate:0 SamplerAddress:192.168.50.11 TimeFlowStart:1624985192 TimeFlowEnd:1624985192 Bytes:140 Packets:2 SrcAddr:10.130.0.2 DstAddr:10.129.0.26 Etype:2048 Proto:6 SrcPort:57064 DstPort:8443 InIf:5 OutIf:65533 SrcMac:00:00:00:00:00:00 DstMac:00:00:00:00:00:00 SrcVlan:0 DstVlan:0 VlanId:0 IngressVrfID:0 EgressVrfID:0 IPTos:0 ForwardingStatus:0 IPTTL:0 TCPFlags:22 IcmpType:0 IcmpCode:0 IPv6FlowLabel:0 FragmentId:0 FragmentOffset:0 BiFlowDirection:0 SrcAS:0 DstAS:0 NextHop:0.0.0.0 NextHopAS:0 SrcNet:0 DstNet:0 HasEncap:false SrcAddrEncap:<nil> DstAddrEncap:<nil> ProtoEncap:0 EtypeEncap:0 IPTosEncap:0 IPTTLEncap:0 IPv6FlowLabelEncap:0 FragmentIdEncap:0 FragmentOffsetEncap:0 HasMPLS:false MPLSCount:0 MPLS1TTL:0 MPLS1Label:0 MPLS2TTL:0, MPLS2Label: 0, MPLS3TTL:0 MPLS3Label:0 MPLSLastTTL:0 MPLSLastLabel:0 HasPPP:false PPPAddressControl:0

Type:NETFLOW_V5 TimeReceived:1624985192 SequenceNum:4298 SamplingRate:0 SamplerAddress:192.168.50.11 TimeFlowStart:1624985192 TimeFlowEnd:1624985192 Bytes:327 Packets:4 SrcAddr:10.128.0.8 DstAddr:10.129.0.2 Etype:2048 Proto:6 SrcPort:8443 DstPort:36190 InIf:4 OutIf:65533 SrcMac:00:00:00:00:00:00 DstMac:00:00:00:00:00:00 SrcVlan:0 DstVlan:0 VlanId:0 IngressVrfID:0 EgressVrfID:0 IPTos:0 ForwardingStatus:0 IPTTL:0 TCPFlags:25 IcmpType:0 IcmpCode:0 IPv6FlowLabel:0 FragmentId:0 FragmentOffset:0 BiFlowDirection:0 SrcAS:0 DstAS:0 NextHop:0.0.0.0 NextHopAS:0 SrcNet:0 DstNet:0 HasEncap:false SrcAddrEncap:<nil> DstAddrEncap:<nil> ProtoEncap:0 EtypeEncap:0 IPTosEncap:0 IPTTLEncap:0 IPv6FlowLabelEncap:0 FragmentIdEncap:0 FragmentOffsetEncap:0 HasMPLS:false MPLSCount:0 MPLS1TTL:0 MPLS1Label:0 MPLS2TTL:0, MPLS2Label: 0, MPLS3TTL:0 MPLS3Label:0 MPLSLastTTL:0 MPLSLastLabel:0 HasPPP:false PPPAddressControl:0
```

So cool! Isn't it? 


### sFlow configuration

Let's use the sFlow records now and see the outputs!


* Change the Network Flow collector from Netflow to sFlow in the CNO configuration:

```
kubectl get network.operator cluster -o jsonpath="{.spec.exportNetworkFlows}"
{"sFlow":{"collectors":["192.168.50.219:6343"]}}
```

In this case we will use the sFlow format for record our network flows.

* Enable the GoFlow only enabling the sFlow format:

```
[root@centos8-3 ~]# goflow -kafka=false -sflow=true -nf=false
INFO[0000] Starting GoFlow
INFO[0000] Listening on UDP :2056                        Type=NetFlowLegacy
INFO[0000] Listening on UDP :6343                        Type=sFlow

Type:NETFLOW_V5 TimeReceived:1624986809 SequenceNum:17277 SamplingRate:0 SamplerAddress:192.168.50.13 TimeFlowStart:1624986808 TimeFlowEnd:1624986808 Bytes:446 Packets:5 SrcAddr:10.128.2.2 DstAddr:10.128.2.3 Etype:2048 Proto:6 SrcPort:49986 DstPort:8181 InIf:4 OutIf:65533 SrcMac:00:00:00:00:00:00 DstMac:00:00:00:00:00:00 SrcVlan:0 DstVlan:0 VlanId:0 IngressVrfID:0 EgressVrfID:0 IPTos:0 ForwardingStatus:0 IPTTL:0 TCPFlags:27 IcmpType:0 IcmpCode:0 IPv6FlowLabel:0 FragmentId:0 FragmentOffset:0 BiFlowDirection:0 SrcAS:0 DstAS:0 NextHop:0.0.0.0 NextHopAS:0 SrcNet:0 DstNet:0 HasEncap:false SrcAddrEncap:<nil> DstAddrEncap:<nil> ProtoEncap:0 EtypeEncap:0 IPTosEncap:0 IPTTLEncap:0 IPv6FlowLabelEncap:0 FragmentIdEncap:0 FragmentOffsetEncap:0 HasMPLS:false MPLSCount:0 MPLS1TTL:0 MPLS1Label:0 MPLS2TTL:0, MPLS2Label: 0, MPLS3TTL:0 MPLS3Label:0 MPLSLastTTL:0 MPLSLastLabel:0 HasPPP:false PPPAddressControl:0

Type:NETFLOW_V5 TimeReceived:1624986809 SequenceNum:17277 SamplingRate:0 SamplerAddress:192.168.50.13 TimeFlowStart:1624986807 TimeFlowEnd:1624986807 Bytes:1648 Packets:6 SrcAddr:10.129.2.9 DstAddr:10.128.2.7 Etype:2048 Proto:6 SrcPort:60638 DstPort:3000 InIf:8 OutIf:65533 SrcMac:00:00:00:00:00:00 DstMac:00:00:00:00:00:00 SrcVlan:0 DstVlan:0 VlanId:0 IngressVrfID:0 EgressVrfID:0 IPTos:0 ForwardingStatus:0 IPTTL:0 TCPFlags:24 IcmpType:0 IcmpCode:0 IPv6FlowLabel:0 FragmentId:0 FragmentOffset:0 BiFlowDirection:0 SrcAS:0 DstAS:0 NextHop:0.0.0.0 NextHopAS:0 SrcNet:0 DstNet:0 HasEncap:false SrcAddrEncap:<nil> DstAddrEncap:<nil> ProtoEncap:0 EtypeEncap:0 IPTosEncap:0 IPTTLEncap:0 IPv6FlowLabelEncap:0 FragmentIdEncap:0 FragmentOffsetEncap:0 HasMPLS:false MPLSCount:0 MPLS1TTL:0 MPLS1Label:0 MPLS2TTL:0, MPLS2Label: 0, MPLS3TTL:0 MPLS3Label:0 MPLSLastTTL:0 MPLSLastLabel:0 HasPPP:false PPPAddressControl:0
```

Amazing! it's gold information showing things like Source Address, destination address, the protocols, the macaddress, and tons of additional information!

Let's take a deeper look on that useful info.

### Looking into the extracted network flows

If we select the last trace obtained (totally random):

```
Type:NETFLOW_V5 TimeReceived:1624986809 SequenceNum:17277 SamplingRate:0 SamplerAddress:192.168.50.13 TimeFlowStart:1624986807 TimeFlowEnd:1624986807 Bytes:1648 Packets:6 SrcAddr:10.129.2.9 DstAddr:10.128.2.7 Etype:2048 Proto:6 SrcPort:60638 DstPort:3000 InIf:8 OutIf:65533 SrcMac:00:00:00:00:00:00 DstMac:00:00:00:00:00:00 SrcVlan:0 DstVlan:0 VlanId:0 IngressVrfID:0 EgressVrfID:0 IPTos:0 ForwardingStatus:0 IPTTL:0 TCPFlags:24 IcmpType:0 IcmpCode:0 IPv6FlowLabel:0 FragmentId:0 FragmentOffset:0 BiFlowDirection:0 SrcAS:0 DstAS:0 NextHop:0.0.0.0 NextHopAS:0 SrcNet:0 DstNet:0 HasEncap:false SrcAddrEncap:<nil> DstAddrEncap:<nil> ProtoEncap:0 EtypeEncap:0 IPTosEncap:0 IPTTLEncap:0 IPv6FlowLabelEncap:0 FragmentIdEncap:0 FragmentOffsetEncap:0 HasMPLS:false MPLSCount:0 MPLS1TTL:0 MPLS1Label:0 MPLS2TTL:0, MPLS2Label: 0, MPLS3TTL:0 MPLS3Label:0 MPLSLastTTL:0 MPLSLastLabel:0 HasPPP:false PPPAddressControl:0
```

we can have a lot of information that we can use for investigate and analyze the traffic.

* SamplerAddress:192.168.50.13
* SrcAddr:10.129.2.9 
* DstAddr:10.128.2.7
* SrcPort:60638 
* DstPort:3000

So in this case we are observing on the host 192.168.50.13 of our cluster one workload communicating with another from the 10.129.2.9 to the 10.128.2.7.

The IPs are corresponding of the cluster Network subnet of our cluster, inside of the SDN and manage for our Cluster Network Operator:

```
kubectl get networks cluster -o jsonpath={'.status.clusterNetwork}'
[{"cidr":"10.128.0.0/14","hostPrefix":23}]
```

The Node is the compute-0 where both workloads are running:

```
oc get node compute-0 -o wide
NAME        STATUS   ROLES    AGE   VERSION                INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                                                       KERNEL-VERSION          CONTAINER-RUNTIME
compute-0   Ready    worker   69d   v1.21.0-rc.0+4b2b6ff   192.168.50.13   <none>        Red Hat Enterprise Linux CoreOS 48.84.202105281935-0 (Ootpa)   4.18.0-305.el8.x86_64   cri-o://1.21.0-100.rhaos4.8.git3dfc2a1.el8
```

The source address is the pod in the OpenShift pipelines namespace, the tekton pipelines controller

```
$ kubectl get pod -A -o wide | grep 10.128.2.9
openshift-pipelines                                tekton-pipelines-controller-6f6fc5fc95-kwpks                      1/1     Running     0          29m     10.128.2.9      compute-0   <none>           <none>
```

And the destination corresponds to the grafana pod in the OpenShift monitoring namespace:

```
$ kubectl get pod -A -o wide | grep 10.128.2.7
openshift-monitoring                               grafana-74655dbf66-f6hjg                                          2/2     Running     0          29m     10.128.2.7      compute-0   <none>           <none>
```

Finally, [Ricardo Carrillo have a fork of the GoFlow repo](https://github.com/rcarrillocruz/goflow/commit/18e0fae87ac4bc78fc8c9c37af64b0ce330c4748), that implements an Enricher interface including to StateSFlow / SFlow. Big thanks Ricardo for your work and for sharing this! Teamwork++

And with that, we completed our first network flow collection of our OpenShift cluster! 
In the next blog post we will use other tools to collect and analyse our Network Flows in a nicer and graphical way!  

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Stay tuned and happy netflowing!

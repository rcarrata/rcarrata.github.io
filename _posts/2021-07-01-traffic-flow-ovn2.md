---
layout: single
title: Monitoring and analysis of Network Flow Traffic in OpenShift (Part II)
date: 2021-07-01
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Networking", "OpenShift", "Observability", "DevSecOps"]
author: rcarrata
comments: true
---

How can you monitor and analyse the network flow records of OpenShift in a graphical way and execute search queries for specific values? And how can you have dashboards to expose this information for your SREs or team members?

This is the second blog post about monitoring and analysis of Network Flow Traffic in OpenShift and it's based on the [Monitoring Network Flow Traffic in OpenShift](https://rcarrata.com/openshift/traffic-flow-ovn/) blog post. So if you didn't check it, go ahead and take a look! :)

## Overview

Now that we can collect the network flows of our network traffic in the OpenShift clusters using the OVN Kubernetes CNI plugin, our job is done, right? But if you have checked the last blog post, the amount of network flows collected is massive, and without proper categorization it is hard to search for anything valuable without using greps on each flow and diving deep into every record collected.

But is this the only way to do it? 

How about collecting the network flows and aggregating them in a central point to visualize later and create dashboards to expose and consume the information in a nicer way? What if we use a stack that is well known in OpenShift like the Elastic Stack?

* Elasticsearch - We will use ES as our network flow store, that will be where the flow records in sFlow format will be stored.  
* Kibana - this will be our UI component that we can use to view the flow records, graphs, dashboards, etc.
* Logstash - Logstash dynamically ingests, transforms, and ships your data regardless of format or complexity. We will use a specific implementation of Logstash called [ElastiFlow](https://github.com/robcowart/elastiflow)

You can use the ECK Operator, as a [very nice OpenShift article](https://www.openshift.com/blog/run-elastic-cloud-on-kubernetes-on-red-hat-openshift) describes.

In this specific case, for quick tweaks that we needed to address, I prepared a [GitHub repository](https://github.com/rcarrata/ocp4-netflow) with all the pieces needed for this blog post.

## Installing the ELK and the ElastiFlow in OpenShift 4

* Clone the repository:

```
git clone https://github.com/rcarrata/ocp4-netflow.git
```

* Deploy ELK with the kustomization using oc command:

```
oc apply -k elastiflow/overlay
```

* Assign privileged scc to the SA of elastiflow

```
oc adm policy add-scc-to-user privileged -z default -n elastiflow
```

This is needed because ES and ElastiFlow need some capabilities that are restricted by default in OpenShift.

NOTE: this is NOT recommended for a productive environment, nor other critical environments. It's just a PoC, so please DON'T do it in prod/pre environments. Use the proper SAs with the proper rbac :)

* Check the resources created in the namespace

```
$ oc get pod -n efk
NAME                          READY   STATUS    RESTARTS   AGE
elasticsearch-0               1/1     Running   0          1h
elastiflow-75cfd94848-btqjl   1/1     Running   0          1h
kibana-658885cf88-wp54g       1/1     Running   0          1h
```

We are using the following versions in this PoC:

* ES - 7.0.1
* ElastiFlow - 3.5.0
* Kibana - 7.0.1

* Check the Route and the Services:

```
oc get svc,route -n efk
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/elasticsearch   ClusterIP      172.30.167.126   <none>        9200/TCP         25h
service/kibana          ClusterIP      172.30.152.42    <none>        5601/TCP         25h
service/logstash        LoadBalancer   172.30.11.107    <pending>     6343:30391/UDP   25h

NAME                                     HOST/PORT                                  PATH   SERVICES   PORT   TERMINATION     WILDCARD
route.route.openshift.io/kibana-secure   kibana-elastiflow.apps.xxx.xxx.com          kibana     5601   edge/Redirect   None
```

NOTE: Red Hat does NOT offer support of ANY kind for ElastiFlow or any associations, so please be aware of that.

## Configure the ElastiFlow dashboards in Kibana

Access the route of the Kibana dashboard in the ns elastiflow:

Once the page loads, go to Management -> Index Patterns -> Create Index Pattern. Just pop a star/wildcard in the box and hit next step until done like so:

[![](/images/flow0_99.png "Flow 0")]({{site.url}}/images/flow0_99.png)

Now we need to import some dashboards. One of the things I really like about this project is that it has a nice collection ready for import.

We will use one of the dashboard templates created originally in the [elastiflow repository](https://github.com/robcowart/elastiflow/tree/master/kibana). We will use a specific version matching our ElastiFlow version, located in [our example repository](https://github.com/rcarrata/ocp4-netflow/blob/main/dashboards/elastiflow.kibana.7.0.x.json).

Go to Management -> Saved Objects -> Import saved objects and upload the "elastiflow.kibana.7.0.x.json" file.

[![](/images/flow0_98.png "Flow 1")]({{site.url}}/images/flow0_98.png)

After waiting a bit, you will receive a lot of very nice dashboards available to check:

[![](/images/flow0_97.png "Flow 2")]({{site.url}}/images/flow0_97.png)

## Check the Network Flow traffic in Kibana dashboards

Now that we have the whole ELK and ElastiFlow set up properly and the dashboards properly configured, let's dig into the dashboards and the information they expose!

If you go to Discover you will see a lot of sFlow flow records of our network traffic collected in ES:

[![](/images/flow0_1.png "Flow 3")]({{site.url}}/images/flow0_1.png)

A lot of useful info, isn't it? 

If you go to the Dashboards you will see the dashboards available:

[![](/images/flow0.png "Flow 4")]({{site.url}}/images/flow0.png)

This represents all the servers and clients that originated inside our OpenShift cluster within the SDN managed by the OVN Kubernetes CNI plugin and CNO.

And it can be filtered in a nice way, using Client - Servers filtering by the IPs or even some services like Etcd-Client.

## Filtering and obtaining information about our network traffic

For example, if you want to know the Network Flows with the destination of our Kubernetes API service, you can filter with the Server tab:

[![](/images/flow2.png "Flow 5")]({{site.url}}/images/flow2.png)

Another example is to use the Traffic Details tab inside this dashboard to select one of the workloads you are interested in.

For example, we want to know what's the traffic originated from the Grafana instance:

```
$ oc get pod -A -o wide | grep grafana
openshift-monitoring                               grafana-74655dbf66-f6hjg                                          2/2     Running     0          43h     10.128.2.7      compute-0   <none>           <none>
```

In the Traffic Details tab, filter by the source address and select the ClusterIP address of our Grafana instance (10.128.2.7):

[![](/images/flow7.png "Flow 6")]({{site.url}}/images/flow7.png)

As you can see we have 3 clients, one of them is the 10.129.2.9:

```
oc get pod -A -o wide | grep 10.129.2.9
openshift-monitoring                               prometheus-k8s-0                                                  7/7     Running     1          43h     10.129.2.9      compute-2   <none>           <none>
```

Our traffic originated from Grafana going to the Prometheus instance in order to access the stored metrics.

## Analysing Flow Records of the etcd clients

One nice thing about this dashboard is that it allows us to filter also by categories, for example by service etcd-client.

With that we can extract very useful information such as who originated the request, the amount of traffic generated, which type of traffic, etc:

[![](/images/flow4.png "Flow 7")]({{site.url}}/images/flow4.png)

If you need more information about one of the flow records, you can go to the Flow Records tab to check the sFlow records collected from our cluster:

[![](/images/flow5.png "Flow 7")]({{site.url}}/images/flow5.png)

And that's all about how to monitor and analyse network traffic from OpenShift using Elastic Stack.

Thanks for reading and hope that you enjoyed the blog post as much as I did writing it.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy NetFlowing! 
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
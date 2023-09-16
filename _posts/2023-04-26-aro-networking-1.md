---
layout: single
title: Ingress and Egress Networking Deep Dive in Public ARO Clusters
date: 2023-04-26
type: post
published: false
status: publish
categories:
- ARO
- Kubernetes
- Cloud
- OpenShift
- Azure
tags: []
author: rcarrata
comments: true
---

## Overview


### 1.1 Azure Infrastructure Components

Resources which have to be deployed by the Customer like Firewall, UDR, ...

[![](/images/aro-networking1.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking1.png)

* Azure Portal Components View

[![](/images/aro-networking2.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking2.png)

### 2.1 ARO Infrastructure Components

Resources which are deployed with the Service like LBs, VMs, etc

[![](/images/aro-networking3.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking3.png)

#### 2.1.1 ARO Specific Resource Group (aka ARO Infra RG)

[![](/images/aro-networking4.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking4.png)

#### 2.1.2 ARO Public IPs

[![](/images/aro-networking5.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking5.png)

#### 2.1.3 ARO Load Balancer

[![](/images/aro-networking6.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking6.png)

[![](/images/aro-networking7.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking7.png)

#### 2.1.4 VMs (Master + Workers)

[![](/images/aro-networking8.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking8.png)

#### 2.1.5 Network Security Groups

[![](/images/aro-networking13.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking13.png)

## 3.3 ARO Service Public API Portal Overview

### 3.3.1 Public IPs for API (<cluster_name-hash-pip-v4)

#### Azure Public LB - FrontEnd IP

[![](/images/aro-networking9.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking9.png)

#### Azure Public LB - Azure Load Balancing Rules

[![](/images/aro-networking10.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking10.png)

#### Azure Public LB - Public IP for API attached

[![](/images/aro-networking11.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking11.png)

#### Azure Public LB - Backend Pool

[![](/images/aro-networking12.png "ARO Networking Diagram")]({{site.url}}/images/aro-networking12.png)

Apps User
Initiates connections to the default Ingress endpoint (*.apps.xxx.eastus.aroapp.io/).

Common Endpoints
ARO Console
OAuth 
Monitoring (Prometheus) 
User Workloads (myapp.apps.xxx.eastus.aroapp.io) 
Image Registry (if it’s exposed externally)
etc

Typical personas:
Developer
End-User
Endpoint monitoring
2.3 Cluster Workload
Initiates outbound connections to the Internet / Azure APIs

ARO Operators with external/Internet requests
ARO OAuth + AAD idp
Group Sync Operator
Image Registry
User App Workloads (connecting to ExpressRoute / Internet)
etc  


*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*
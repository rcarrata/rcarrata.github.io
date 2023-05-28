---
layout: post
title: Secure your Private OpenShift clusters with Azure Firewall and Hub-Spoke architectures
date: 2022-04-22
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How can we generate Private clusters OpenShift in Azure, secured by Azure Firewall? How can we have a Fine-Grained egress access from our Applications within our OpenShift cluster, controlling and managing all the inbound and outbound traffic? 

This is the second blog post of the series of OpenShift deployments in Azure. Check the first part in [Deep dive in Private OpenShift 4 clusters deployments in Azure](https://rcarrata.com/openshift/ocp4-azure-ipi-private/).

Let's dig in!

## 1. Overview

As we checked in the previous blog post, we have several options to deploy our Private OpenShift clusters, depending of their Outbound / User-Defined Routing modes that we want to use.

We analyzed a couple of this User Defined Routing (UDR), including Azure Load Balancer (default), and also the use of the Proxy Configuration for the OpenShift cluster installations.

Now we're analyzing two of the most complex but flexible and interesting options too:

* Private cluster with Azure Firewall

* Private cluster with network address translation (NAT Gateway)

Both are super useful and try to solve the same issue: manage and control the egress routing traffic from the OCP cluster to Internet, but the point of view of the implementation are quite different.

In this blog post we will deep dive in the Azure Firewall mode, showing some of the features and security capabilities, that can help us to secure our workloads and OpenShift private clusters. 

## 2. Egress / Outbound Mode with Azure Firewall

Let's start digging a bit deeper!

As we commented, we can use an Azure Firewall to provide outbound Internet routing for the VNet for the OCP cluster installation and usage.

As we mentioned, this solution to securing outbound addresses lies in use of a firewall device that can control outbound traffic based on domain names. Azure Firewall, for example, can restrict outbound HTTP and HTTPS traffic based on the FQDN of the destination. We can also configure your preferred firewall and security rules to allow these required ports and addresses, but in this blog post we will stick with the Azure Firewall managed.

In the following diagram we can view a possible architecture of an Private OpenShift cluster with Azure Firewall for the Outbound / Egress Traffic:

[![](/images/azurefw0.png "azurefw0.png")]({{site.url}}/images/azurefw0.png)

Let's dig a bit deeper on this architecture, because have a lot of details that it's can be worth it to be explained more carefully.

We can divide this architecture in several parts:

* Azure Hub and Spoke Architecture
* Azure Firewall
* User Define Routing and Azure Firewall
* OpenShift Private Cluster outbound routing

[![](/images/azurefw1.png "azurefw1.png")]({{site.url}}/images/azurefw1.png)

Let's start to dig a bit deeper in all of this Azure resources needed for our architecture.

### 2.1 Azure Hub & Spoke Architecture

The [Hub-Spoke topology in Azure](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke?tabs=cli) is a reference architecture provided by Microsoft:

[![](/images/azurefw0_1.png "azurefw0_1.png")]({{site.url}}/images/azurefw0_1.png)

NOTE: this diagram is extracted from the MSFT official documentation where this architecture is detailed.

* In this architecture the **hub virtual network** acts as a central point of connectivity to many spoke virtual networks. The hub can also be used as the connectivity point to your on-premises networks.

* The **Spoke virtual networks** are used to isolate workloads in their own virtual networks, managed separately from other spokes. Each workload might include multiple tiers, with multiple subnets connected through Azure load balancers.

* The **Virtual network peering** is also present allowing that two virtual networks can be connected using a peering connection. Peering connections are non-transitive, low latency connections between virtual networks. Once peered, the virtual networks exchange traffic by using the Azure backbone without the need for a router.

[![](/images/azurefw0_3.png "azurefw0_3.png")]({{site.url}}/images/azurefw0_3.png)

The Hub-Spoke topology in Azure is a recommended best practice architecture that have several benefits such as cost savings, overcoming limits and the most important workload isolation.

In our case, we've used the Hub-Spoke architecture because we can control all the outbound traffic (and also inbound), and filter in fine grain which URLs, IPs are white or blacklisted or even filtered by certain origins from our OpenShift cluster.

That adds much more flexibility to the Private OpenShift cluster deployment, adding more security and control to the ingress/egress traffic that comes from/to our OCP cluster.

### 2.2 Azure Firewall and OCP

[Azure Firewall](https://docs.microsoft.com/en-us/azure/firewall/overview) is a cloud-native and intelligent network firewall security service that provides the best of breed threat protection for your cloud workloads running in Azure.

It's a fully stateful, firewall as a service with built-in high availability and unrestricted cloud scalability. It provides both east-west and north-south traffic inspection.

[![](/images/azurefw0_2.png "azurefw0_2.png")]({{site.url}}/images/azurefw0_2.png)

We will benefit from this Azure Firewall, that will act as a WAF, filtering and controlling all the traffic that comes from our cluster (or goes into).

Another feature that we can benefit is the URL filtering, blacklisting certain URLs/IP ranges, and also is capable even of act as IDPS and/or adding TLS inspection. Cool isn't?

The Azure Firewall lives in their own Subnet, defined as Hub in the Hub-Spoke architecture that is detailed in the step before.

[![](/images/azurefw0_4.png "azurefw0_4.png")]({{site.url}}/images/azurefw0_4.png)

This allows that ALL the traffic to/from our cluster (or clusters) will go through our VNET - Azure Firewall because of the Route Tables that will be explained below.

And on the other hand, imagine that instead of 1 cluster of OpenShift, you will deploy N OpenShift clusters that are private. With this architecture, you will have the possibility to deploy them as a Spokes in their own VNET / Subnets, having a proper architecture and control of your re^sources in Azure.

### 2.3 User Define Routing and Azure Firewall

Imagine that we want to deploy our OpenShift cluster, but how the worker and master nodes and the pods within the OpenShift cluster can communicate with Internet? They need to go though the Azure Firewall that controls all the outbound traffic that will be egressing the OpenShift cluster (in one of the Spokes) towards the Hub VPC network where the Azure Firewall is deployed.

That's called User Define Routing or UDR and it's defined when we install the OpenShift cluster as we described in the first blog post [Deep dive in Private OpenShift 4 clusters deployments in Azure](https://rcarrata.com/openshift/ocp4-azure-ipi-private/).

In Azure the route table already has a default 0.0.0.0/0 to Internet, but without a Public IP to SNAT just adding this route will not provide you egress.

We need to remember that when using an outbound type of UDR, a load balancer public IP address for inbound requests is not created unless a service of type loadbalancer is configured. A public IP address for outbound requests is never created by OpenShift installer if an outbound type of UDR is set, ensuring that no public endpoint it's generated in the VPC of our OpenShift cluster.

In this case, the Outbound type of UDR **requires that there is a Azure route for 0.0.0.0/0 and next hop destination of NVA (Network Virtual Appliance) in the route table**.

For this reason we need to create a [couple of Route Table entries](https://github.com/rcarrata/ocp4-azure-ipi/blob/main/roles/ocp4-cloud-ipi/tasks/azure-infra-firewall.yml#L100) that will have **as next hop type the NVA (Network Virtual Appliance)** that in our case will be the virtual appliance aka Azure Firewall, that will have also defined the Azure route table and the next hop ip address of the Hub VNet of the Azure Firewall, forcing all the traffic that exits from the Spoke to go through this route, and therefore through the Azure Firewall to Internet:

[![](/images/azurefw1_1.png "azurefw1_1.png")]({{site.url}}/images/azurefw1_1.png)

We will need to create to different Azure route table entries for the subnets, one for the control-plane / masters and another for the computes / workers, making an association for the route tables to the subnets added, telling these subnets that will need to use this route table.

In a nutshell the following diagrams depicts the Azure Route Table that enables the User Defined Routing (UDR) that replaces the Azure Default Route that is present by default in each VPC:

[![](/images/azurefw3.png "azurefw3.png")]({{site.url}}/images/azurefw3.png)

As we can see we have the DG-Route that uses as Address Prefix the 0.0.0.0/0 (all traffic that egress the OCP Cluster), and as next hop the ip address (10.1.1.4) of the Azure FW Subnet (remember that it was 10.1.1.0/24 and it's within this range):

[![](/images/azurefw5.png "azurefw5.png")]({{site.url}}/images/azurefw5.png)

And finally we have a Public IP for the Azure Firewall, where we can expose our specific applications, and could be our ingress if we want to expose only AppA and not expose or our OCP console or our other apps within our cluster. Total flexibility and security right?

[![](/images/azurefw7.png "azurefw7.png")]({{site.url}}/images/azurefw7.png)

If you want to expand the information about the Azure networking part, check the [UDR and the Azure Networking guide](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#user-defined) and also the [Route traffic through NVAs by using custom settings](https://docs.microsoft.com/en-gb/azure/virtual-wan/scenario-route-through-nvas-custom) in the Azure site.

### 2.4 Setup and configure Azure Firewall security

Now that we have explained the networking specifics about the UDR and the networking egress, let's assume that our traffic it's within the Azure Firewall Subnet.

What's are the specifics of the Azure Firewall?

We used in this architecture an [Azure Firewall Standard](https://docs.microsoft.com/en-us/azure/firewall/features), because offers for this example the capabilities that are enough for this PoC, but if you need more features such as TLS inspection, IDPS, or Url filtering among others, you can select the [Azure Firewall Premium](https://docs.microsoft.com/en-us/azure/firewall/premium-features).

For deploying the [Azure Firewall in our architecture](https://github.com/rcarrata/ocp4-azure-ipi/blob/main/roles/ocp4-cloud-ipi/tasks/azure-infra-firewall.yml#L66) we need to associate the Public IP, and also select the Firewall subnet where will be deployed, but because we're cool kids we're using Ansible to automate the whole process :D.

[![](/images/azurefw8.png "azurefw8.png")]({{site.url}}/images/azurefw8.png)

Now that we've deployed our cool Azure Firewall, let's secure our scenario a bit more: let's filter the URLs that our cluster can access using the Azure Firewall Rules.

We can configure NAT rules, network rules, and applications rules on Azure Firewall using either classic rules or Firewall Policy. Azure Firewall denies all traffic by default, until rules are manually configured to allow traffic.

Let's configure our Application Rules within the Azure Firewall Policy to define what URLs and websites are allowed (the rest will be denied, securing our network, Zero Trust Network principle :D):

[![](/images/azurefw2.png "azurefw2.png")]({{site.url}}/images/azurefw2.png)

Some of these are needed for the installation of the OpenShift cluster like quay.io / openshift.com, and also the *azure.com or*microsoftonline.com for performing the API call towards the Azure API (for example by the Machine Config Operator to generate more machines though MachineSets):

[![](/images/azurefw9.png "azurefw9.png")]({{site.url}}/images/azurefw9.png)

But also we allowed others useful sites like the dockerhub, or the Google Services that could be useful for our pods within the Cluster to be reachable.

But all other traffic is denied, because our pod don't need to go to our favorite sports website or ... a hacker website, or to grab some dangerous rootkit or malware, right?

### 2.5 Automating deployment of the OpenShift cluster using Azure Firewall as Outbound traffic

Now that we defined and explained all the resources that we will have in the architecture, we need to deploy our architecture and then our OpenShift cluster within.

First we need to deploy the VPC and Subnets, then the Firewall and the Route Tables, and all that we explained before.

Then we need to deploy our bastion, and finally deploy our cluster of OpenShift using all the Azure resources that we needed as prerequisites.

We need to deploy all of this things, manually? No, let's use Ansible for that!

1. Clone the repository where the ocp4 azure ipi demo is stored:

```sh
git clone https://github.com/rcarrata/ocp4-azure-ipi.git
cd ocp4-azure-ipi
```

2. Install Azure CLI:

```sh
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/azure-cli.repo

sudo dnf install azure-cli

az login
```

3. Fill and create the Azure Creds for the Service Principal inherit by the Openshift Installer:

```sh
cat ~/.azure/osServicePrincipal.json
{"subscriptionId":"xxxx","clientId":"xxxx","clientSecret":"xxxx","tenantId":"xxxx"}
```

4. Install Ansible and Azure Cli dependencies:

```sh
pip install virtualenv
virtualenv ansible-210
source ansible-210/bin/activate
pip install ansible==2.10
pip install selinux
ansible-galaxy collection install azure.azcollection
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements-azure.txt
```

5. Add the vault directory and the vault yaml and generate a Vault-File with the credentials of Azure and OCP4 PullSecret:

```sh
ansible-vault edit vault/azure.yml
```

6. Fill inside the vault.yml with:

```sh
azure_subscription_id: SECRET
azure_client_id: SECRET
azure_secret: SECRET
azure_tenant: SECRET
ocp4_pull_secret: '<<< pull_secret_azure >>>'
```

7. For obtain the pull_secret go to [OCP4 Install](https://cloud.redhat.com/openshift/install)

8. Generate the .vault-password-file and put the password:

```
touch .vault-password-file
echo "yourpasswordfancy" >> .vault-password-file
```

9. Install Openshift with Azure Firewall as the Egress Outbound:

```sh
ansible-playbook install-private.yml -e "egress=firewall" --vault-password-file .vault-file-password
```

## 2.6 Checking the Egress from the OpenShift cluster through Azure Firewall

Let's check the egress traffic from the OpenShift cluster through the Azure Firewall, and we will see if the Azure Firewall is securing and filtering the outbound traffic from our pods within our OCP cluster.

* From the Bastion host, export the kubeconfig of the brand new cluster:

```md
[root@bastion ~] ssh -D 9000 <<public_bastion_ip>> -l az-admin -i vault/ssh-key

[root@bastion ~] export KUBECONFIG=ocp4/auth/kubeconfig
```

* We will select one of the pods that is deployed in the cluster (the console pod, but could be another totally random):

```md
[root@bastion ~] ./ocp4/kubectl get pod -n openshift-console -l app=console --no-headers | awk '{ print $1 }' | head -n1
```

* Let's connect to the pod of the console to perform the connectivity checks:

```md
[root@bastion ~] CONSOLE_POD=$(./ocp4/kubectl get pod -n openshift-console -l app=console --no-headers | awk '{ print $1 }' | head -n1)

[root@bastion ~] ./ocp4/kubectl -n openshift-console exec $CONSOLE_POD -ti -- bash
bash-4.4$ 

* Check the connectivity from the pod to the Google website that is allowed in the Azure Firewall:  

```md
bash-4.4$ curl www.google.com -I
HTTP/1.1 200 OK
```

as we can see the connectivity it's ok.

* Check the connectivity from the pod to the Red Hat website, also allowed in the Azure Firewall:

```md
bash-4.4$ curl www.redhat.com -Iv

* Rebuilt URL to: www.redhat.com/
* Trying 23.73.225.160...
* TCP_NODELAY set
* Connected to www.redhat.com (23.73.225.160) port 80 (#0)

> HEAD / HTTP/1.1
> Host: www.redhat.com
> User-Agent: curl/7.61.1
> Accept: */*
```

* On the other hand, also check the OpenShift docs from the pod:

```md
bash-4.4$ curl https://docs.openshift.com -I
HTTP/1.1 200 Connection established
HTTP/2 200
```

* Then we check one website that is not explicitly allowed in the Azure Firewall (wikipedia website):

```md
bash-4.4$ curl www.wikipedia.com
Action: Deny. Reason: No rule matched. Proceeding with default action.

bash-4.4$ curl www.wikipedia.com -I
HTTP/1.1 470 status code 470
```

as we can see it's denied, as the reason it's the rule not matched by the Azure Firewall.

* Finally we can check another website that is not allowed in the Firewall rules:

```md
bash-4.4$ curl www.bbc.co.uk
Action: Deny. Reason: No rule matched. Proceeding with default action.

bash-4.4$ curl www.bbc.co.uk -I
HTTP/1.1 470 status code 470

bash-4.4$ curl https://www.bbc.co.uk
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to www.bbc.co.uk:443
```

## 3. Connecting to our private cluster from the Internet

Now that we have deploy the cluster, how we can access to the console that it's not exposed neither in the Azure Firewall?

As we discussed in the previous blog post, there are several options, like having a VPN Gateway, an Express Route or even deploying a VM and use Azure Bastion to connecting from there to the Bastion.

But there is a very simple, cost-effective way to do that, use the Bastion for enable a [Proxy Socks5](https://en.wikipedia.org/wiki/SOCKS) ssh connection.

A SOCKS proxy is an SSH encrypted tunnel in which configured applications forward their traffic down, and then, on the server-end, the proxy forwards the traffic to the general Internet.

Unlike a VPN, a SOCKS proxy has to be configured on an app-by-app basis on the client machine, but we can set up apps without any specialty client software as long as the app is capable of using a SOCKS proxy. On the server-side, all you need to configure is SSH.

Let's do try it!

* Connect to the bastion using the private key generated

```md
ssh -i vault/ssh-key  -D 9000 az-admin@xx.xx.xx.xx
Activate the web console with: systemctl enable --now cockpit.socket
[az-admin@bastion ~]$ sudo -i
```

* And then connect using the Proxy Socks with the DNS option enabled. An example with Firefox is:

[![](/images/azureipi14.png "azureipi14.png")]({{site.url}}/images/azureipi14.png)

as you can check the SOCKSv5 needs to be enabled, we used here the port 9000.

And then the most important check is the "Proxy DNS when using SOCKSv5" option. This option will redirect and resolve all the requests from our browser through the Proxy socks SSH tunnel, reaching the Azure DNS private records, and then resolving our OpenShift Console.

* Now if we use our regular ocp console url in our browser, we will see the OpenShift console without need of setting an expensive VPN or ExpressRoute.

[![](/images/azureipi15.png "azureipi15.png")]({{site.url}}/images/azureipi15.png)

But remember that it's only for testing purpose, once the ssh connection is closed, or if the VM suffers any issue, this method is also unavailable.

So with that we finished the second part of this blog around "Secure your Private OpenShift clusters with Azure Firewall and Hub-Spoke architectures".

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

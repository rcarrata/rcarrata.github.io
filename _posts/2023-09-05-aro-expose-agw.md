---
layout: single
title: Exposing apps using Application Gateway LB in Private ARO clusters
date: 2023-09-05
type: post
published: true
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

What is the role of Application Gateway in enabling the secure exposure of customer applications within Private Azure Red Hat OpenShift (ARO) clusters?
How does Application Gateway integrate with Private ARO clusters and align with the connectivity strategy of Open Hybrid Cloud?
What are the benefits of using Application Gateway for load balancing customer applications within ARO clusters, especially during high demand scenarios?

## Overview

In the dynamic world of hybrid multi-cloud environments, organizations are constantly seeking robust solutions to expose their customer applications securely. In this blog post, we will focus on the critical topic of exposing customer applications within Private Azure Red Hat OpenShift (ARO) clusters and shed light on the pivotal role played by Application Gateway in enabling this process.

Application Gateway, a versatile component of the Azure ecosystem, serves as a powerful tool for secure application exposure. It seamlessly integrates with Private ARO clusters, aligning with the overarching connectivity strategy of Open Hybrid Cloud.

By leveraging Application Gateway, organizations can unlock a lot of benefits when it comes to exposing customer applications with Private ARO Cluster. Firstly, it acts as a highly efficient load balancer, intelligently distributing incoming traffic across multiple instances of applications running within the ARO cluster. This ensures optimal performance and availability, even during high demand scenarios.

Moreover, Application Gateway provides robust security features, safeguarding customer applications from external threats. It offers comprehensive SSL/TLS termination, enabling end-to-end encryption for enhanced data protection. Additionally, its Web Application Firewall (WAF) functionality protects against common web vulnerabilities, providing an additional layer of defense.

The objective of this blog post is to demo exposing some Customer Applications deployed in a Private ARO cluster, that requires to expose only the Application itself, not the ARO API Kubernetes Ingress or any other *.apps routes.

Also the certificates needs to be taken in consideration, due to we will NOT use a Custom Domain for our ARO Cluster. We will be using a Let's Encrypt certificate with the APP FQDN, and we will put the certificate in the AppGW and in the OpenShift App Route in the ARO cluster.  

[![](/images/appgw0.png "AppGW")]({{site.url}}/images/appgw0.png)

## Azure Application Gateway

[Azure Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/overview) is a load balancer designed for managing web traffic directed towards your web applications. Unlike traditional load balancers that operate at the transport layer (OSI layer 4 - TCP and UDP) and direct traffic solely based on source IP address and port to a destination IP address and port, Application Gateway goes a step further.

Application Gateway has the capability to make routing decisions based on additional attributes of an HTTP request, such as the URI path or host headers.

## Prerequisites 

* [ARO Private Cluster](https://learn.microsoft.com/en-us/azure/openshift/howto-create-private-cluster-4x) (use 10.0.10.0/16 as VNet CIDR)
* [Jumphost VM with Public IP](https://learn.microsoft.com/en-us/azure/openshift/howto-restrict-egress#create-a-jump-host-vm)

## Setting Environment Variables

* Set some specific environment variables for the ARO environment:

```md
export NAMESPACE=aro-app-agw
export AZR_CLUSTER=aro-$USER
export AZR_RESOURCE_LOCATION=eastus
export AZR_RESOURCE_GROUP=aro-$USER-rg
export AppGW_CIDR="10.0.10.0/23"
export AppGW_SUBNET="Ingress-subnet"
export ARO_VNET_NAME="aro-$USER-vnet"
export APP_NAME="aro-hello-openshift"
export DNS_ZONE_NAME="test.openshiftdemo.dev"
export APPGW_DOMAIN="$APP_NAME.$DNS_ZONE_NAME"
export AppGW_PIP="AppGW-pip"
export AZR_DNS_RESOURCE_GROUP="mobb-dns"
export EMAIL=username.taken@gmail.com
```

NOTE: Customize these variables for your own deployment!

## AppGW Networking and Private DNS Zones

* Create Subnet for AppGW:

```md
az network vnet subnet create \
  --resource-group $AZR_RESOURCE_GROUP \
  --vnet-name $ARO_VNET_NAME \
  --name $AppGW_SUBNET \
  --address-prefixes $AppGW_CIDR \
  --service-endpoints Microsoft.ContainerRegistry
```

NOTE: due to the Subnets from AppGW and ARO share the same VNet is not needed to add a peering. If you want to split between two different VNets instead of Subnets, please be remember adding the VNet peering between them.

* Create a static public IP address for the Application Gateway LB:

```md
az network public-ip create \
  --resource-group $AZR_RESOURCE_GROUP \
  --name $AppGW_PIP \
  --allocation-method Static \
  --sku Standard
```

* Create a Private DNS Zone with the same public DNS_Zone_Name as we will be using in the blog post:

```md
az network private-dns zone create \
    --resource-group $AZR_RESOURCE_GROUP \
    --name $DNS_ZONE_NAME
```

This is needed because the AppGW requires to reach the internal ARO LB when the Backend Rule is applied in the AppGW for our Custom Domain Application. 

* Retrieve the ARO Ingress Internal IP Load Balancer:

```md
INGRESS_IP="$(az aro show -n $AZR_CLUSTER -g $AZR_RESOURCE_GROUP --query 'ingressProfiles[0].ip' -o tsv)"
echo $INGRESS_IP
```

* Add a record for our FQDN to a private DNS zone pointing to the ARO Ingress Internal IP LB:

```
az network private-dns record-set a add-record \
  --resource-group $AZR_RESOURCE_GROUP \
  --zone-name $DNS_ZONE_NAME \
  --record-set-name "$APP_NAME" \
  --ipv4-address $INGRESS_IP
```

NOTE: We are using the same $APP_NAME (in our case aro-hello-openshift) with the same private DNS zone (test.openshiftdemo.dev), pointing to the Azure Internal LB that will load balance to the Workers where the ARO OpenShift Routers are (OpenShift Ingress Controllers that manages the Haproxies OpenShift Routers).

* Link Private DNS Zone to ARO Virtual Network:

```md
az network private-dns link vnet create \
    --resource-group $AZR_RESOURCE_GROUP \
    --zone-name $DNS_ZONE_NAME \
    --name private-dnszone-link-$ARO_VNET_NAME \
    --virtual-network $ARO_VNET_NAME \
    --registration-enabled false
```

## Application GW and WAF policy

Now that we have our Networking and the DNS resolution deployed and configured, let's create the Application Gateway LB and the WAF Policies.

### Create the Application Gateway Load Balancer and WAF policies

* Create a Web Application Firewall (WAF) policy for the Application Gateway:

```md
az network application-gateway waf-policy create \
  --resource-group $AZR_RESOURCE_GROUP \
  --name AppGW-WAF-Policy-$USER
```

* Creates an Application Gateway with the specified configurations, including the WAF policy, public IP, and subnet:

```md
az network application-gateway create \
  --name "AppGW-aro-$USER" \
  --location $AZR_RESOURCE_LOCATION \
  --resource-group $AZR_RESOURCE_GROUP \
  --capacity 1 \
  --priority 1 \
  --sku WAF_v2 \
  --http-settings-cookie-based-affinity Disabled \
  --public-ip-address $AppGW_PIP \
  --vnet-name $ARO_VNET_NAME \
  --subnet $AppGW_SUBNET \
  --waf-policy AppGW-WAF-Policy-$USER
```

* The AppGW needs to be deployed and assigned to the proper resource group with the Public IP attached:

[![](/images/appgw1.png "AppGW")]({{site.url}}/images/appgw1.png)

NOTE: The WAF policy needs to be enabled, because by default it's in Disabled mode. 

[![](/images/appgw2.png "AppGW")]({{site.url}}/images/appgw2.png)

### AppGW Load Balancer Application Certificates

The AppGW and our apps deployed will need to have the proper certificates attached, due to the AppGW will check also against the backend, if the certificated presented is valid and have the proper FQDN.

* Generate SSL certificates using Let's Encrypt's Certbot tool:

```
export SCRATCH_DIR=/tmp/scratch
mkdir -p $SCRATCH_DIR

certbot certonly --manual \
  --preferred-challenges=dns \
  --email $EMAIL \
  --server https://acme-v02.api.letsencrypt.org/directory \
  --agree-tos \
  --manual-public-ip-logging-ok \
  -d "$APPGW_DOMAIN" \
  --config-dir "$SCRATCH_DIR/config" \
  --work-dir "$SCRATCH_DIR/work" \
  --logs-dir "$SCRATCH_DIR/logs"
```

NOTE: don't close or interrupt this process, we will finish after the dns challenge in Azure.

* Open a second terminal and paste the DNS_Challenge (and remember to export again the variables from the beggining):

```
export DNS_CHALLENGE="xxxx"
```

* Adds a TXT record to the Azure DNS zone for the ACME challenge:

```
az network dns record-set txt add-record \
  --resource-group $AZR_DNS_RESOURCE_GROUP \
  --zone-name $DNS_ZONE_NAME \
  --record-set-name "_acme-challenge.$APP_NAME" \
  --value "$DNS_CHALLENGE"
```

* Wait up to 5mins (maybe more) until the TXT record propagates and check the DNS resolution from the ACME challenge within Azure DNS. Check that the dig output matches with the DNS Challenge shown before by the certbot command:

```
dig @8.8.8.8 +short TXT _acme-challenge.$APPGW_DOMAIN
```

@8.8.8.8 (not use the local dns cached)

* Return to the previous terminal and finish the generation of the ACME certificate for our ARO example App

* Certificate Bundle: Concatenate the generated SSL certificates into a bundle file and export it as a PKCS12 file.

```
export PFX_PASS="mypa55w0rd"

cat $SCRATCH_DIR/config/live/$APPGW_DOMAIN/fullchain.pem $SCRATCH_DIR/config/live/$APPGW_DOMAIN/privkey.pem > $SCRATCH_DIR/config/live/$APPGW_DOMAIN/gw-bundle.pem

openssl pkcs12 -export -out $SCRATCH_DIR/config/live/$APPGW_DOMAIN/gw-bundle.pfx -in $SCRATCH_DIR/config/live/$APPGW_DOMAIN/gw-bundle.pem
```

* Delete the TXT record created for the ACME challenge:

```
az network dns record-set txt delete \
  --resource-group $AZR_DNS_RESOURCE_GROUP \
  --zone-name $DNS_ZONE_NAME \
  --name "_acme-challenge.$APP_NAME"
```

### Updating the DNS Records for AppGW and the exposed app

We need to update the DNS records for AppGW, using the Public IP that was generated in the step before.

* Retrieve the public IP address of the Application Gateway:

```
AGW_PIP=$(az network public-ip show -g $AZR_RESOURCE_GROUP --name $AppGW_PIP --query ipAddress -o tsv)
```

* Update the DNS record with the Application Gateway's public IP address:

```
az network dns record-set a add-record \
--resource-group $AZR_DNS_RESOURCE_GROUP \
--zone-name $DNS_ZONE_NAME \
--record-set-name "$(echo $APPGW_DOMAIN | sed 's/\..*//')"  \
--ipv4-address $AGW_PIP
```

* Verify the DNS resolution for the updated domain: 

```
dig @8.8.8.8 +short $APPGW_DOMAIN
```

* Creates an HTTPS listener for the Application Gateway:

```
az network application-gateway ssl-cert create \
  --resource-group $AZR_RESOURCE_GROUP \
  --gateway-name "AppGW-aro-$USER" \
  --name gw-bundle \
  --cert-file $SCRATCH_DIR/config/live/$APPGW_DOMAIN/gw-bundle.pfx \
  --cert-password $PFX_PASS
```

## AppGW Listeners and Backends

* In the Listeners section, create a new HTTPS listener using the Azure portal:

```
Listener name: aro-route-https-listener
Frontend IP: Public
Port: 443
Protocol: HTTPS
Http Settings - choose to Upload a Certificate (upload the PFX file from earlier)
Cert Name: gw-bundle
PFX certificate file: gw-bundle.pfx
Host Type: single 
Host name: $APPGW_DOMAIN (aro-hello-openshift.test.openshiftdemo.dev)
```

Note: You can also create multiple listeners - one per site and re-use the certificate and select basic site. Also we are using the Azure Portal because the CLI doesn't support MultiHostnames.

[![](/images/appgw3.png "AppGW")]({{site.url}}/images/appgw3.png)

* Create a new backend pool (cli):

```
az network application-gateway address-pool create \
  --gateway-name "AppGW-aro-$USER" \
  --resource-group $AZR_RESOURCE_GROUP \
  --name aro-routes \
  --servers aro-hello-openshift.test.openshiftdemo.dev
```

[![](/images/appgw4.png "AppGW")]({{site.url}}/images/appgw4.png)


* Create a new backend HTTP setting using the Azure Portal:

```
In the HTTP settings section, create a new HTTP setting:
HTTP settings name: aro-route-https-settings
Backend protocol: HTTPS
Backend port: 443
Use well known CA certificat: Yes (if you used one; otherwise upload your CA cer file)
Override with new host name: Yes
Choose: Override with specific domain name
Host name: $APPGW_DOMAIN
```

NOTE: We are using the Azure Portal because the CLI doesn't support MultiHostnames. 

[![](/images/appgw5.png "AppGW")]({{site.url}}/images/appgw5.png)

* Define a rule for each website/api (cli):

```
az network application-gateway rule create \
  --gateway-name "AppGW-aro-$USER" \
  --resource-group $AZR_RESOURCE_GROUP \
  --name aro-app-https-rules \
  --http-listener aro-route-https-listener \
  --address-pool aro-routes \
  --http-settings aro-route-https-settings \
  --priority 2
```
[![](/images/appgw6.png "AppGW")]({{site.url}}/images/appgw6.png)

### Exposing HTTPD App

Now it's time to publish our Hello OpenShift app deployed in the Private Cluster, exposed using the AppGW.

* Open a sshuttle connection to the ARO Private Cluster:

```md
JUMP_IP=$(az vm list-ip-addresses -g $AZR_RESOURCE_GROUP -n aro-$USER-jumphost -o tsv \
--query '[].virtualMachine.network.publicIpAddresses[0].ipAddress')
echo $JUMP_IP

sshuttle --dns -NHr "aro@${JUMP_IP}"  10.0.0.0/8 --daemon
```

* Login to ARO Private Cluster (using sshuttle):

```
ARO_URL=$(az aro show -g $AZR_RESOURCE_GROUP -n $AZR_CLUSTER --query apiserverProfile.url -o tsv)
ARO_PASS=$(az aro list-credentials --name $AZR_CLUSTER --resource-group $AZR_RESOURCE_GROUP -o tsv --query kubeadminPassword)
oc login --username kubeadmin --password $ARO_PASS --server=$ARO_URL
ARO_DOMAIN=$(oc get dns cluster -o jsonpath='{.spec.baseDomain}')
```

* Create new project for testing app:


oc new-project aro-appgw
```

* Deploy a httpd server K8s Deployment and expose using a K8s Service:

```md
oc create deployment hello-openshift --image=quay.io/openshifttest/hello-openshift:1.2.0 --port 8080
oc expose deployment hello-openshift
```

* Add the Edge Route with the hostname as the "aro-httpd.$DOMAIN"

```md
oc create route edge --service=hello-openshift --hostname=$APPGW_DOMAIN \
    --key $SCRATCH_DIR/config/live/$APPGW_DOMAIN/privkey.pem \
    --cert $SCRATCH_DIR/config/live/$APPGW_DOMAIN/fullchain.pem
```

## Testing that the App works 

Now that we've deployed the App, let's test if works.

* Grab the App Route:

```md
APP=$(oc get route hello-openshift -o jsonpath='{.spec.host}')
```

* Execute a couple of requests, and check the response code:

```md
curl https://$APP
Hello OpenShift!

curl https://$APP -I
HTTP/1.1 200 OK
x-request-port: 8080
```

* Check the App exposed using the Custom Domain, and published in the AppGW Listener:

[![](/images/appgw7.png "AppGW")]({{site.url}}/images/appgw7.png)

And with that ends the third blog post around exposing Applications using App Gateway Load Balancers in Private ARO clusters.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="You like this blog? It helped? Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
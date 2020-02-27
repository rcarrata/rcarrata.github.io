---
layout: post
title: UPI Installation of Openshift 4
date: 2019-08-22
type: post
published: true
status: publish
categories:
- Openshift
tags: []
author: rcarrata
comments: true
---


# OCP4 UPI Install Cloudformation

## Download openshift-install binary
```
openshift_version=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/ | grep -E "openshift-install-linux-.*.tar.gz" | sed -r 's/.*href="([^"]+).*/\1/g')
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_version 
or curl  https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_version --output $openshift_version
sudo tar -xvzf $openshift_version -C /usr/local/bin/
```

## Download oc binary
```
openshift_cli=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/ | grep -E "openshift-client-linux-.*.tar.gz" | sed -r 's/.*href="([^"]+).*/\1/g')
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_cli
or 
curl  https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/$openshift_cli --output $openshift_cli
sudo tar -xvzf $openshift_cli -C /usr/local/bin/
```


## 1.Creating the installation files for AWS

```
[root@clientvm 0 ~]# mkdir upi_ocp4_aws
[root@clientvm 0 ~]# cd upi_ocp4_aws/
```

* Generate the install-config.yaml file:

```
[root@clientvm 0 ~/upi_ocp4_aws]# openshift-install create install-config --dir=.
? SSH Public Key /root/.ssh/cluster-aa37-key.pub
? Platform aws
? Region eu-west-1
? Base Domain aa37.sandbox675.opentlc.com
? Cluster Name rcarrata-upi-aws
? Pull Secret [? for help] *********************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************************
```

* Change the worker number to 0, because the workers will be deployed with Cloudformation templates
(and not with the MachineSets as the IPI installation does):

```
sed -i '3,/replicas: / s/replicas: .*/replicas: 0/' install-config.yaml
```

* The install-config.yml will have these properties:

```
[root@clientvm 0 ~/upi_ocp4_aws]# cat install-config.yaml
apiVersion: v1
baseDomain: aa37.sandbox675.opentlc.com
compute:
- hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: rcarrata-upi-aws
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-west-1
pullSecret: <<Pull-Secret>>
sshKey: |
  ssh-rsa <<RSA>>
```

* Backup the install-config.yaml for future purposes:

```
[root@clientvm 130 ~/upi_ocp4_aws]# cp -pr install-config.yaml install-config.yaml.bkp
```

* Generate the Kubernetes manifests for the cluster:

```
[root@clientvm 0 ~/upi_ocp4_aws]# openshift-install create manifests --dir=.
WARNING There are no compute nodes specified. The cluster will not fully initialize without compute
nodes.
INFO Consuming "Install Config" from target directory
```

* Remove the files that define the control plane machines and remove the Kubernetes manifest files
that define the worker machines:

```
[root@clientvm 0 ~/upi_ocp4_aws]# rm -f openshift/99_openshift-cluster-api_master-machines-*.yaml
[root@clientvm 0 ~/upi_ocp4_aws]# rm -f openshift/99_openshift-cluster-api_worker-machineset-*
[root@clientvm 0 ~/upi_ocp4_aws]# ll openshift/
total 32
-rw-r--r--. 1 root root  293 Jun 18 09:05 99_binding-discovery.yaml
-rw-r--r--. 1 root root  219 Jun 18 09:05 99_cloud-creds-secret.yaml
-rw-r--r--. 1 root root  181 Jun 18 09:05 99_kubeadmin-password-secret.yaml
-rw-r--r--. 1 root root 2729 Jun 18 09:05 99_openshift-cluster-api_master-user-data-secret.yaml
-rw-r--r--. 1 root root 2729 Jun 18 09:05 99_openshift-cluster-api_worker-user-data-secret.yaml
-rw-r--r--. 1 root root  879 Jun 18 09:05 99_openshift-machineconfig_master.yaml
-rw-r--r--. 1 root root  879 Jun 18 09:05 99_openshift-machineconfig_worker.yaml
-rw-r--r--. 1 root root  222 Jun 18 09:05 99_role-cloud-creds-secret-reader.yaml
```

* Obtain the Ignition config files:

```
[root@clientvm 0 ~/upi_ocp4_aws]# openshift-install create ignition-configs --dir=.
INFO Consuming "Master Machines" from target directory
INFO Consuming "Worker Machines" from target directory
INFO Consuming "Openshift Manifests" from target directory
INFO Consuming "Common Manifests" from target directory
```

## 2.Upload the ignition files to the s3 bucket

* Create the s3 bucket for the ignition files:

```
[root@clientvm 255 ~/upi_ocp4_aws]# aws s3 mb s3://rcarrata-upi-aws
make_bucket: rcarrata-upi-aws

[root@clientvm 0 ~/upi_ocp4_aws]# aws s3 ls
2019-06-11 13:00:20 cf-templates-9y3oiuo0ffcr-eu-west-1
2019-06-12 09:49:32 image-registry-eu-west-1-94afc372799f407d92d0a5fd9e3006a4-5f7b
2019-06-10 13:57:24 image-registry-eu-west-1-a8335277611649cf8527cf3402ca1b6e-ab3f
2019-06-18 09:48:38 rcarrata-upi-aws
2019-06-10 13:05:42 terraform-20190610130539799800000001
```

* Upload the aws s3 bootstrap.ign to the s3 bucket:

```
[root@clientvm 0 ~/upi_ocp4_aws]# aws s3 cp bootstrap.ign s3://rcarrata-upi-aws
upload: ./bootstrap.ign to s3://rcarrata-upi-aws/bootstrap.ign

[root@clientvm 0 ~/upi_ocp4_aws]#
[root@clientvm 0 ~/upi_ocp4_aws]# aws s3 ls s3://rcarrata-upi-aws
2019-06-18 09:50:41     279214 bootstrap.ign
```

## 3.Generating Cloudformation Templates

```for i in $(ls *.json.orig); do cp -p $i $(echo $i | sed -e "s/\.json\.orig/\.json/g"); done```

## 4. Creating the VPC

As this client is not necessary to create the VPC and their resources, because the client need to
install OCP4 into their own AWS network infrastructure already created.

## 5. Creating Networking and Load Balancing Components in AWS

### 5.1 Inputs for CF Networking and Load Balancing

* ClusterName

```ClusterName="rcarrata-upi-aws" ; sed -i -e "s/clustername/$ClusterName/g" *.json```

* InfrastructureName

```
[root@clientvm 0 ~/upi_ocp4_aws]# infrastructurename=$(jq -r .infraID ~/upi_ocp4_aws/metadata.json)
[root@clientvm 0 ~/upi_ocp4_aws]# echo $infrastructurename
rcarrata-upi-aws-pgmmr
```

```sed -i -e "s/infrastructurename/$infrastructurename/g" *.json```

* HostedZoneName

```hostedzonename="aa37.sandbox675.opentlc.com"```

NOTE: Remember to do not include the absolute domain name "with the dot(.) at the end", use the
relative domain name (without the dot(.) at the end"

```sed -i -e "s/hostedzonename/$hostedzonename/g" *.json```

* HostedZoneId

```
[root@clientvm 0 ~]# hostedzoneid=$(aws route53 list-hosted-zones | jq -r --arg hostedzonename "$hostedzonename." '.HostedZones[] | select(.Config.PrivateZone==false and .Name==$hostedzonename) | .Id' | cut -d"/" -f3)
```

```sed -i -e "s/hostedzoneid/$hostedzoneid/g" *.json```

* VpcId

```
[root@clientvm 0 ~]# vpcid=$(aws ec2 describe-vpcs | jq -r '.Vpcs[] | select(.Tags[].Value=="vpc-ocp")? | .VpcId')

[root@clientvm 0 ~]# echo $vpcid
vpc-068d6d598e99bb487
```

```sed -i -e "s/vpcid/$vpcid/g" *.json```

* PrivateSubnets

* NOTE: The filter for identify the Private and the Public subnets in this case is
  Tags.Key=="kubernetes.io/role/internal-elb because the Subnets are deployed with this tags

```
[root@clientvm 130 ~/06-cloudformation]# privatesubnets=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="private")  | .SubnetId' | paste -s -d",")

[root@clientvm 0 ~/06-cloudformation]# echo $privateSubnets
subnet-064445b01e62bb354,subnet-0ba6da724f2e3b5ee,subnet-016378620a80fd8e5
```

```sed -i -e "s/privatesubnets/$privatesubnets/g" *.json```

* PublicSubnets

```
[root@clientvm 0 ~]# publicsubnets=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="public")  | .SubnetId' | paste -s -d",")

[root@clientvm 0 ~]# echo $publicsubnets
subnet-0152e8b64674198e5,subnet-057a7683cd0c03610,subnet-0ad38bf28936e9ff1
```

```sed -i -e "s/publicsubnets/$publicsubnets/g" *.json```

## 5.1 Creating Networking and Load Balancing Components in AWS

```
[root@clientvm 0 ~/06-cloudformation]#  aws cloudformation create-stack --stack-name clusterinfra --template-body file://02_cluster_infra.yaml --parameters file://02_cluster_infra.json --capabilities CAPABILITY_NAMED_IAM

{
    "StackId": "arn:aws:cloudformation:eu-west-1:823863422774:stack/clusterinfra/23284640-91de-11e9-b5c3-0a2f634edb2c"
}
```

```
[root@clientvm 127 ~/06-cloudformation]# aws cloudformation wait stack-create-complete --stack-name clusterinfra
```

```
[root@clientvm 255 ~]# aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[]'
```

## 6. Creating security group and roles in AWS

### 6.1 Input Parameters Json Cloudformation Template

* PrivateSubnets (already filled)

* VpcCidr

```
vpccidr=$( aws ec2 describe-vpcs | jq -r --arg vpcid "$vpcid" '.Vpcs[] | select(.VpcId==$vpcid) | .CidrBlock' | sed 's/^\([0-9]*\.[0-9]*.[0-9]*\.[0-9]*\)/\1\\/')
```
```sed -i -e "s/vpccidr/$vpccidr/g" *.json```

**NOTE**: To adapt

* PrivateSubnets (already filled)

* VpcId (already filled)

### 6.1 Executing Cloudformation Template for Networking and Load Balancing

```
aws cloudformation create-stack --stack-name clustersecurity --template-body file://03_cluster_security.yaml --parameters file://03_cluster_security.json --capabilities CAPABILITY_NAMED_IAM

{
    "StackId":
    "arn:aws:cloudformation:eu-west-1:823863422774:stack/clustersecurity/81a14a00-9279-11e9-aaa9-06366d428b7a"
}
```

```
[root@clientvm 0 ~/06-cloudformation]# aws cloudformation wait stack-create-complete --stack-name clustersecurity
```

```
[root@clientvm 130 ~/06-cloudformation]# aws cloudformation describe-stacks --stack-name clustersecurity | jq -r '.Stacks[].Outputs[] | .OutputKey +": "+ .OutputValue'

MasterSecurityGroupId: sg-054ce5b3ab76ec33e
MasterInstanceProfile: clustersecurity-MasterInstanceProfile-1RPVN7PDJKI5O
WorkerSecurityGroupId: sg-0caf26028b4c08eaf
WorkerInstanceProfile: clustersecurity-WorkerInstanceProfile-9O9642CXG3SV
```

## 7. Creating the bootstrap node in AWS

### 7.1 Input Parameters for Bootstrap Node

* RhcosAmi:  [RHCOSAMI](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-aws-user-infra-rhcos-ami_installing-aws-user-infra)

```rhcosami="ami-02fdd627029c0055b"```

```sed -i -e "s@rhcosami@$rhcosami@g" *.json```

* AllowedBootstrapSshCidr: by default to "0.0.0.0/0"

* PublicSubnets (already filled)

* MasterSecurityGroupID:

```
[root@clientvm 0 ~/06-cloudformation]# mastersecuritygroupid=$(aws cloudformation describe-stacks --stack-name clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterSecurityGroupId") | .OutputValue')

```

```sed -i -e "s/mastersecuritygroupid/$mastersecuritygroupid/g" *.json```

* PublicSubnets (already filled)

* VpcId (already filled)

* BootstrapIgnitionLocation

```
[root@clientvm 0 ~/06-cloudformation]# aws s3 ls s3://rcarrata-upi-aws/bootstrap.ign
2019-06-18 09:50:41     279214 bootstrap.ign
```

```bootstrapignitionlocation="s3://rcarrata-upi-aws/bootstrap.ign"```

```sed -i -e "s@bootstrapignitionlocation@$bootstrapignitionlocation@g" *.json```

* AutoRegisterELB: Whether or not to register a network load balancer (NLB). By default yes

* RegisterNlbIpTargetsLambdaArn:

```
[root@clientvm 0 ~/06-cloudformation]#  registernlbiptargetslambdaarn=$(aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="RegisterNlbIpTargetsLambda") | .OutputValue')

[root@clientvm 0 ~/06-cloudformation]# echo $RegisterNlbIpTargetsLambdaArn
arn:aws:lambda:eu-west-1:823863422774:function:clusterinfra-RegisterNlbIpTargets-1AIHYDPFJAM2Q
```

```sed -i -e "s@registernlbiptargetslambdaarn@$registernlbiptargetslambdaarn@g" *.json```

* ExternalApiTargetGroupArn

```
[root@clientvm 0 ~]# externalapitargetgrouparn=$(aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[]       | select(.OutputKey=="ExternalApiTargetGroupArn") | .OutputValue')

[root@clientvm 0 ~]# echo $externalapitargetgrouparn
arn:aws:elasticloadbalancing:eu-west-1:823863422774:targetgroup/clust-Exter-U9N1JEU32WTC/a1dd12b0d61ba868
```

```sed -i -e "s@externalapitargetgrouparn@$externalapitargetgrouparn@g" *.json```

* InternalApiTargetGroupArn

```
[root@clientvm 130 ~/06-cloudformation]# internalapitargetgrouparn=$( aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InternalApiTargetGroupArn") | .OutputValue')
```

```sed -i -e "s@internalapitargetgrouparn@$internalapitargetgrouparn@g" *.json```

* InternalServiceTargetGroupArn

```
[root@clientvm 0 ~/06-cloudformation]# internalservicetargetgrouparn=$(aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="InternalServiceTargetGroupArn") | .OutputValue')
```


```sed -i -e "s@internalservicetargetgrouparn@$internalservicetargetgrouparn@g" *.json```

```
[root@clientvm 0 ~/06-cloudformation]# publicsubnet=$( aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="public")?  | .SubnetId' |
head -1 )
```


```sed -i -e "s@publicsubnet@$publicsubnet@g" *.json```

### 7.2 Executing Bootstrap Cloudformation Template

```
[root@clientvm 0 ~/06-cloudformation]# aws cloudformation create-stack --stack-name clusterbootstrap --template-body file://04_cluster_bootstrap.yaml --parameters file://04_cluster_bootstrap.json --capabilities CAPABILITY_NAMED_IAM

{
    "StackId":
    "arn:aws:cloudformation:eu-west-1:823863422774:stack/clusterbootstrap/6ba53df0-9293-11e9-965a-0afa58e29f3e"
}
```

```
[root@clientvm 130 ~/06-cloudformation]# aws cloudformation wait stack-create-complete --stack-name
clusterbootstrap
```

## 8. Creating the control plane machines in AWS

### 8.1 Input Variables for Cloudformation Template

* RhcosAmi: (already filled)

* PrivateHostedZoneId

```
[root@clientvm 0 ~/06-cloudformation]#  privatehostedzoneid=$( aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[]       | select(.OutputKey=="PrivateHostedZoneId") | .OutputValue')

```

```sed -i -e "s@privatezoneid@$privatehostedzoneid@g" *.json```

* PrivateHostedZoneName

```
[root@clientvm 0 ~/06-cloudformation]# InfrastructureShortName=$(jq -r .clusterName ~/upi_ocp4_aws/metadata.json | cut -d"-" -f1-3)

[root@clientvm 0 ~/06-cloudformation]# privatehostedzonename=$(aws route53 list-hosted-zones | jq -r '.HostedZones[] | select(.Config.PrivateZone==true) | .Name' | grep $InfrastructureShortName)
```

```sed -i -e "s@privatezonename@$privatehostedzonename@g" *.json```

* Master0Subnet

```
[root@clientvm 0 ~]# master0subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="private") | .SubnetId' | paste -s -d"," - | cut -d"," -f1)


[root@clientvm 0 ~]# echo $master0subnet
subnet-064445b01e62bb354
```

```sed -i -e "s@master0subnet@$master0subnet@g" *.json```


* Master1Subnet

```
[root@clientvm 0 ~]# master1subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="private") | .SubnetId' | paste -s -d"," - | cut -d"," -f2)

[root@clientvm 0 ~]# echo $master1subnet
subnet-0ba6da724f2e3b5ee
```

```sed -i -e "s@master1subnet@$master1subnet@g" *.json```

* Master2Subnet

```
master2subnet=$(aws ec2 describe-subnets --filter="Name=vpc-id,Values=$vpcid" | jq -r '.Subnets[] | select(.Tags[].Value=="private") | .SubnetId' | paste -s -d"," - | cut -d"," -f3)

[root@clientvm 0 ~]# echo $master2subnet
subnet-016378620a80fd8e5
```

```sed -i -e "s@master2subnet@$master2subnet@g" *.json```

* MasterSecurityGroupId

```
[root@clientvm 0 ~]# mastersecuritygroupid=$(aws cloudformation describe-stacks --stack-name clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterSecurityGroupId") | .OutputValue')

[root@clientvm 0 ~]# echo $mastersecuritygroupid
sg-054ce5b3ab76ec33e
```

```sed -i -e "s@mastersecuritygroupid@$mastersecuritygroupid@g" *.json```

* IgnitionLocation:

```
[root@clientvm 0 ~/upi_ocp4_aws]# apiserverdnsname=$(aws cloudformation describe-stacks --stack-name clusterinfra | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="ApiServerDnsName") | .OutputValue')

[root@clientvm 0 ~/upi_ocp4_aws]# echo $apiserverdnsname
api-int.rcarrata-upi-aws.aa37.sandbox675.opentlc.com

[root@clientvm 130 ~/upi_ocp4_aws]# masterignitionlocation="https://$apiserverdnsname:22623/config/master"
```

```sed -i -e "s@masterignitionlocation@$masterignitionlocation@g" *.json```

* CertificateAuthorities

```
[root@clientvm 130 ~/06-cloudformation]#  mastercert=$(cat ../upi_ocp4_aws/master.ign | jq -r .ignition.security.tls.certificateAuthorities[].source)
```

```sed -i -e "s@mastercert@$mastercert@g" *.json```

* MasterInstanceProfile

```
[root@clientvm 0 ~/06-cloudformation]# masterinstanceprofile=$(aws cloudformation describe-stacks --stack-name clustersecurity | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="MasterInstanceProfile") | .OutputValue')

[root@clientvm 0 ~/06-cloudformation]# echo $masterinstanceprofile
clustersecurity-MasterInstanceProfile-1RPVN7PDJKI5O
```

```sed -i -e "s@masterinstanceprofile@$masterinstanceprofile@g" *.json```

* MasterInstanceType: [MasterInstancesTypes](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html#installation-creating-aws-control-plane_installing-aws-user-infra)

```masterinstancetype="m4.xlarge"```

```sed -i -e "s@masterinstancetype@$masterinstancetype@g" *.json```

## 8.1 Executing the control plane machines in AWS

```
# aws cloudformation create-stack --stack-name clustermaster --template-body file://05_cluster_master_nodes.yaml --parameters file://05_cluster_master_nodes.json --capabilities CAPABILITY_NAMED_IAM
{
    "StackId": "arn:aws:cloudformation:eu-west-1:823863422774:stack/clustermaster/79636a60-8e7f-11e9-b025-0673c4cbbe82"
}

# aws cloudformation wait stack-create-complete --stack-name clustermaster
```

## 9. Creating the worker nodes in AWS

* WorkerInstanceProfile

```
workerinstanceprofile=$( aws cloudformation describe-stacks --stack-name clustersecurity     | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="WorkerInstanceProfile") | .OutputValue')

[root@clientvm 0 ~/06-cloudformation]# echo $workerinstanceprofile
clustersecurity-WorkerInstanceProfile-1RPVN7PDJKI5O
```

```sed -i -e "s/workerinstanceprofile/$workerinstanceprofile/g" *.json```

* WorkerSecurityGroupId

```
workersecuritygroupid=$( aws cloudformation describe-stacks --stack-name clustersecurity  | jq -r '.Stacks[].Outputs[] | select(.OutputKey=="WorkerSecurityGroupId") | .OutputValue')
```

```sed -i -e "s/workersecuritygroupid/$workersecuritygroupid/g" *.json```

* WorkerInstanceType

```workerinstancetype="m4.large"```

```sed -i -e "s/workerinstancetype/$workerinstancetype/g" *.json```
```
workerignitionlocation="https://$apiserverdnsname:22623/config/worker"
```

```sed -i -e "s@workerignitionlocation@$workerignitionlocation@g" *.json```

* CertificateAuthorities

```
[root@clientvm 130 ~/06-cloudformation]#  workercert=$(cat ../upi_ocp4_aws/worker.ign | jq -r .ignition.security.tls.certificateAuthorities[].source)
```

```sed -i -e "s@workercert@$workercert@g" *.json```

```
# aws cloudformation create-stack --stack-name clusterworker --template-body file://06_cluster_worker_node.yaml --parameters file://06_cluster_worker_node.json --capabilities CAPABILITY_NAMED_IAM
{
    "StackId": "arn:aws:cloudformation:eu-west-1:823863422774:stack/clustermaster/79636a60-8e7f-11e9-b025-0673c4cbbe82"
}

# aws cloudformation wait stack-create-complete --stack-name clusterworker
```
```
openshift-install wait-for bootstrap-complete --dir=.
```

**IMPORTANT STEP:**

You must watch if the csrs are aprroved, if not you must approve them with the next command:
```
export KUBECONFIG="kubeconfig"
csr=$(oc get csr | grep "Pending")
if $csr oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
```
Delete the bootstrap resources.
```
 aws cloudformation delete-stack --stack-name clusterbootstrap
```
And finally, wait for the installation complete.
```
openshift-install wait-for install-complete --dir=.
```

---
layout: post
title: Securing CICD pipelines with Stackrox / RHACS and Sigstore 
date: 2022-07-12
type: post
published: false
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How can we ensure that our supply chain is secured, and all the container images deployed in our Kubernetes clusters are signed and secured, and no one can deploy malicious container images? How can we Sign and Verify container images within the CICD Pipelines, adding Security to our DevOps pipelines?

This is the third blog post of the series of Secure CICD pipelines using Sigstore and Open Source security tools. 

Check the earlier posts in:

* [I - Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno](https://rcarrata.com/kubernetes/sign-images-1/)
* [II - Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno - (Part II)](https://rcarrata.com/kubernetes/sign-images-2/)

Let's dig in!

## 1. Overview

In the previous blog posts we've used Kyverno and Sigstore to secure the CICD pipelines within Tekton. 
Furthermore we worked with GitHub registry to store the signatures of the images that we built, and to store the images that our Tekton Pipelines produced.

But these are not the only options that we have for secure our CICD pipelines.

Let's discover how other Open Source tools like Stackrox / RHACS and Quay can help, alongside with Sigstore to sign, verify and enforce our CICD / DevOps pipelines.

As in the previous blog posts we will start from a typical CICD pipeline where we build - bake - deploy our container images in our Kubernetes / OpenShift clusters.

And then we will use several components that will help us to secure this pipeline:

* [Cosign](https://github.com/sigstore/cosign): Open Source tool within Sigstore project that provides image-signing and verification, so we can sign our container images, to ensure that our container image stored in our registry is signed properly and also the signature is stored within the Registry and always available to be verified.

* [Quay](https://github.com/quay/quay): Open Source container registry that stores, builds, and deploys container images. It analyzes your images for security vulnerabilities, identifying potential issues that can help you mitigate security risks.
NOTE: we will use the SaaS option, Quay.io in this blog post but it's totally possible to reproduce this demo in the non SaaS Quay.

* [Stackrox](https://github.com/stackrox/stackrox): StackRox Kubernetes Security Platform performs a risk analysis of the container environment, delivers visibility and runtime alerts, and provides recommendations to proactively improve security by hardening the environment.
Red Hat offers their supported product based in Stackrox, [Red Hat Advanced Security for Kubernetes](https://www.redhat.com/en/technologies/cloud-computing/openshift/advanced-cluster-security-kubernetes).

On the other hand, ArgoCD / OpenShift GitOps and Tekton Pipelines / OpenShift Pipelines will be also used in this demo as we did in the previous examples.

Finally all the files and code used in the demo is available in the [K8S sign-acs repository](https://github.com/rcarrata/ocp4-network-security/tree/sign-acs/sign-images).

NOTE: All the demo will be deployed in an OpenShift 4 cluster, but also vanilla K8S can be used, using OLM and modifying the namespaces used.

## Installing ArgoCD and Tekton

Because we will install all the pieces using ArgoCD and GitOps, we need first to bootstrap ArgoCD cluster. We will use OpenShift GitOps based in ArgoCD in this case.

* Clone the repository and checkout into the proper branch:

```bash
git clone https://github.com/rcarrata/ocp4-network-security.git
git checkout sign-acs && cd sign-images
```

* Install OpenShift GitOps / Pipelines (ArgoCD and Tekton):

```bash
until kubectl apply -k bootstrap/; do sleep 2; done
```

* After couple of minutes check the OpenShift GitOps / ArgoCD and Pipelines / Tekton components are up && running:

```
ARGOCD_ROUTE=$(kubectl get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}')

curl -ks -o /dev/null -w "%{http_code}" https://$ARGOCD_ROUTE
```

## Quay.io Repository Setup

We need a repository for store our container images generated in our pipeline, for these reason we will generate an empty public quay.io repository and we will grant the proper RBAC and security to use them in our pipelines.

* Add a Public Quay Repository in Quay.io (we've used pipelines-vote-api repository as an example):

<img align="center" width="650" src="assets/quay1.png">
[![](/images/azureipi16.png "azureipi16.png")]({{site.url}}/images/azureipi16.png)

* In the Settings repository, add a Robot Account user and assign Write or Admin permissions to this Quay Repository:

<img align="center" width="650" src="assets/quay2.png">
[![](/images/azureipi16.png "azureipi16.png")]({{site.url}}/images/azureipi16.png)

* Grab the QUAY_TOKEN and the USERNAME that is shown within the Robot Account generated in the previous step and export into your shell:

```
export USERNAME=<Robot_Account_Username>
export QUAY_TOKEN=<Robot_Account_Token>
```

## Configure Quay credentials and RBAC in K8S/OCP Cluster

* Export the token for the GitHub Registry / quay.io:

```bash
export QUAY_TOKEN=""
export EMAIL="xxx"
export USERNAME="rcarrata+acs_integration"
export NAMESPACE="demo-sign"
```

* Create the namespace for the demo-sign:

```bash
kubectl create ns ${NAMESPACE}
```

* Generate a docker-registry secret with the credentials for GitHub Registry to push/pull the images and signatures:

```bash
kubectl create secret docker-registry regcred --docker-server=quay.io --docker-username=${USERNAME} --docker-email=${EMAIL}--docker-password=${QUAY_TOKEN} -n ${NAMESPACE}
```

* Add the imagePullSecret to the ServiceAccount “pipeline” in the namespace of the demo:

```
export SERVICE_ACCOUNT_NAME=pipeline
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
 -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
oc secrets link pipeline regcred -n demo-sign
#oc secrets link default regcred -n demo-sign
```

## Deploy Tekton Demo Tasks and Pipelines using GitOps

Let's deploy now the several demo tasks and pipelines that are needed for this demo. We can deploy it manually... or use GitOps to deploy them automatically! 

* Create an ArgoCD App for deploying all the Tekton Pipelines, Tasks and other CICD components needed for these demo:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: signed-pipelines-app
  namespace: openshift-gitops
spec:
  destination:
    namespace: demo-sign
    server: https://kubernetes.default.svc
  project: default
  source:
    path: sign-images/manifests
    repoURL: https://github.com/rcarrata/ocp4-network-security
    targetRevision: sign-acs
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

After a couple of minutes our ArgoCD app will deploy every manifest that it's needed within our Kubernetes / OpenShift cluster:

[![](/images/sign-acs5.png "sign-acs5.png")]({{site.url}}/images/sign-acs5.png)

## Install Stackrox / RHACS using GitOps

Now it's time to install Stackrox / RHACS in our cluster de Kubernetes. We will use again GitOps to deploy all the components needed.

* We will be using the [ACS GitOps repository](https://github.com/rcarrata/rhacs-gitops/tree/main/apps/acs) within our ArgoApp for deploy Stackrox:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: acs-operator
  namespace: openshift-gitops
spec:
  destination:
    namespace: openshift-gitops
    server: https://kubernetes.default.svc
  project: default
  source:
    path: apps/acs
    repoURL: https://github.com/rcarrata/acs-gitops
    targetRevision: develop
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
EOF
```

* As you can check in [some of the k8s manifests](https://github.com/rcarrata/rhacs-gitops/blob/main/apps/acs/3.rbac.yaml#L6) of this ArgoApp that we used to deploy Stackrox, we will use [ArgoCD Syncwaves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) to orchestrate the steps of the installation:

[![](/images/sign-acs6.png "sign-acs6.png")]({{site.url}}/images/sign-acs6.png)

* After the Argo App is fully synched and finished properly, check the Stackrox / ACS route:

```
ACS_ROUTE=$(k get route -n stackrox central -o jsonpath='{.spec.host}')
curl -Ik https://${ACS_ROUTE}
```

NOTE: Check that you're getting a 200.

* Furthermore you can check that your cluster is secured and managed by Stackrox / RHACS within the ${ACS_ROUTE}/main/clusters:

[![](/images/sign-acs7.png "sign-acs7.png")]({{site.url}}/images/sign-acs7.png)
TODO: Upload the image

## Generate Roxctl API Token within Stackrox

Now, that we've our Stackrox cluster up && running, securing our cluster of Kubernetes / OpenShift, let's integrate the Stackrox cluster with their CLI, roxctl.

This is needed because we will use roxctl cli within the Tekton Pipelines image-check tasks, with the Stackrox / RHACS central API. 

For this reason, we need to generate an API Token to authenticate and authorize the roxctl cli against the API of Stackrox Central cluster.

* Generate an API Token within Stackrox, go to Platform Configuration -> Integrations -> Authentication Tokens -> API Token and generate new API Token:

<img align="center" width="450" src="assets/acs1.png">

* Grab the token generated, and export into the ROX_API_TOKEN variable:

```
export ROX_API_TOKEN="xxx"
```

* [Install the roxctl cli](https://docs.openshift.com/acs/3.66/cli/getting-started-cli.html#installing-cli-on-linux_cli-getting-started) and use the roxctl check image to verify if the API Token is working properly:

```
roxctl --insecure-skip-tls-verify image check --endpoint $ACS_ROUTE:443 --image quay.io/centos7/httpd-24-centos7:centos7
```

The output of the command will show that two policies are violated, so the roxctl image check is working as expected:

```
WARN:   A total of 2 policies have been violated
ERROR:  failed policies found: 1 policies violated that are failing the check
ERROR:  Policy "Fixable Severity at least Important" - Possible remediation: "Use your package manager to update to a fixed version in future builds or speak with your security team to mitigate the vulnerabilities."
ERROR:  checking image failed after 3 retries: failed policies found: 1 policies violated that are failing the check
```

NOTE: For further information check the [ACS Integration with CI Systems](https://docs.openshift.com/acs/3.70/integration/integrate-with-ci-systems.html#cli-authentication_integrate-with-ci-systems) guide.

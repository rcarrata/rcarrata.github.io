---
layout: post
title: Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno - (Part II)
date: 2022-03-07
type: post
published: true
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: false
---

How can we Sign and Verify container images within the CICD Pipelines, adding Security to our DevOps pipelines? How can we ensure that our supply chain is secured, and all the container images deployed in our Kubernetes clusters are signed and secure, and no one can deploy malicious container images?

This is the second blog post about Signing and Verify container images to improve security in CICD pipelines, using Sigstore, Kyverno and Tekton pipelines among others.

Check the first part in [Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno - Part I](https://rcarrata.com/kubernetes/sign-images-1/).

Let's dig in!

## 1. Overview

In the previous blog post we've demonstrate how Sigstore is an awesome project focused on software signing and transparency log technologies and can help us to improve software supply chain security.

Cosign is a tool within this project that provides image-signing and verification, so we can sign our container images, to ensure that our container image stored in our registry is signed properly and also the signature is stored within the Registry and always available to be verified.

On the other hand, we also checked that Kyverno is an policy engine designed for Kubernetes, capable to manage policies that can validate, mutate and generate Kubernetes resources and also ensure OCI image supply chain security.

We will see in this blog post how we can combine Sigstore signing images and Kyverno in a CICD pipeline, to increase the security in the supply chain of a software DevOps pipeline.

In this blog post we will using Tekton Pipelines, but other CICD tools like Jenkins, GitHub Actions, GitLab CI and others can be used. On the other hand we used here Kyverno, but other policy engine like OPA could be also used to verify the images in our cluster.

NOTE: All the code / examples used in this blog post are [available in a GitHub repository](https://github.com/rcarrata/ocp4-network-security/tree/main/sign-images). Check this out!

## 2. Signing Container Images within CICD Pipelines

As we checked in the previous part of this blog post, cosign can be used for sign the images, store this signature in an OCI registry, and verify them to increase container image security supply chain.

Now we will demonstrate how container images can be signed during the build phase of a CI/CD pipeline using Cosign.

Let's present our CICD pipeline that we will used during this blog post:

[![](/images/signing0.png "signing0.png")]({{site.url}}/images/signing0.png)

as you can check there are some standard steps that we are familiar with, and follows the three steps of the most basic CICD pipeline:

```sh
CLONE -> BUILD -> DEPLOY
```

1. **Clone / Fetch Repository** - For this secure demo pipeline, we're using a [Voting app example](https://github.com/openshift/pipelines-vote-api) written in Golang. The ClusterTask of git-clone is used in this step cloning the repository in a workspace called "shared-workspace". Task available in [Tekton Hub](https://hub.tekton.dev/tekton/task/git-clone).

2. **Buildah Custom** - These Buildah Custom step is divided in 3 different substeps within the [Buildah-Custom ClusterTask](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/pipelines/clustertasks-buildah-custom.yaml)

In a nutshell these tasks are Build, Push to the registry and Collect the Image Digest to use it in further steps.

Why don't use the Buildah task out of the box? Because this specific task uses the Registry Credentials for allow Buildah to push the images to a private Registry (GitHub registry in this case, but private repository).

[![](/images/signing10.png "signing10.png")]({{site.url}}/images/signing10.png)

3. **Sign Images** - That's the interesting part that we want to deep dive a bit, because it's the part when we use Cosign to sign our image when it's built and pushed to the registry.

[![](/images/signing4.png "signing4.png")]({{site.url}}/images/signing4.png)

As we can remember from the [previous part](https://rcarrata.com/kubernetes/sign-images-1/) of this blog post series, Cosign supports container signing, verification and storage in an OCI registry, and in a nutshell is a tool that greatly simplifies how content is signed and verified by storing signatures from container images and other types in OCI registries.

Cosign store it in a Kubernetes secret using their current context. The secret contains the private and public keys, as well as the password to decrypt the private key.

If we remember, for sign a container and store the signature in the registry there is a command with cosign:

```sh
cosign sign --key cosign.key ghcr.io/rcarrata/ubi-minimal:8.5-230
```

So in this Tekton step, a specific Task inside of the [Tekton pipeline](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/manifests/sign-images-pipeline.yaml#L64), signs the images and pushes this signature to the registry itself.

This [Task and step](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/manifests/tasks-cosign.yaml#L33) signs with the private key generated by cosign, the image that is generated in the step before with Buildah, and pushes the signature to the registry:

```sh
cosign sign --key k8s://$(params.namespace)/$(params.cosignkey) $(params.image)
```

[![](/images/signing8.png "signing8.png")]({{site.url}}/images/signing8.png)

Signatures are uploaded to an OCI artifact stored with a predictable name:

```sh
cosign triangulate ghcr.io/rcarrata/ubi-minimal:8.5-230
ghcr.io/rcarrata/ubi-minimal:sha256-xxx.sig
```

Then we will use an image verification policy defined in a ClusterPolicy CR of Kyverno:

```sh
...
 rules:
    - name: check-image
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - image: "$IMAGE:*"
        key: |-
```

Policy enforcement is captured using Kubernetes events. Kyverno also reports policy violations for existing resources.

The Kyverno verifyImages rule uses Cosign to verify container image signatures, attestations and more stored in an OCI registry.

So when we try to deploy the image in our cluster of Kubernetes/OpenShift, Kyverno will ensure that the image is verified with the Public Key generated by Cosign, and will verify the signature that the cosign used to sign the image using the Private Key generated by Cosign.

4. **Apply the manifests** - Once that the container image is generated and pushed into the registry along with the signature produced using cosign, the k8s manifests needs to be applied, for example the Deployments and the Services.

5. **Update Deployment** - This final step will update the image of the Deployment with the image tag that is created in the pipeline (my_image:sha256-xxx). This will update and apply the new image and will be validated by Kyverno.

If the Kyverno verifyImages policy rule check fails if the signature is not found in the OCI registry, or if the image was not signed using the specified key, Kyverno won't allow the deployment of the image updated, and the Tekton pipeline will fail.

## 3. Deploying the Tekton Tasks and Pipeline

To run this pipeline first you need to create / apply all the Tekton Tasks and Pipelines that we will be using during this blog post.

IMPORTANT: This blog post assume that we've followed all the steps of the previous blog post including installing Tekton, generate keypairs, among other things.

We have two options, deploying with kubectl or... use GitOps and ArgoCD!

### 3.1 Deploy Tekton Tasks and Kyverno using GitOps

First of all we want to mention that deploy the demo with GitOps it's optional, you can use the kubectl to deploy the objects, but because we're cool kids we will use GitOps to save some time.

First, let's install Kyverno with GitOps using Helm chart.

* To do that, let's deploy an ArgoCD Application with the Helm chart and the targetRevision for Kyverno:

```sh
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kyverno
  namespace: openshift-gitops
spec:
  destination:
    namespace: kyverno
    server: https://kubernetes.default.svc
  project: default
  source:
    helm:
    chart: kyverno
    repoURL: https://kyverno.github.io/kyverno/
    targetRevision: v2.3.1
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

* Let's apply it in our K8S/OCP cluster:

```sh
kubectl apply -f argocd/kyverno-app.yaml
```

* After a couple of minutes, our Kyverno components are deployed in the Kubernetes clusters, and all are in sync:

[![](/images/signing2.png "signing2.png")]({{site.url}}/images/signing2.png)

NOTE: remember that you need to patch the Kyverno Deployment to add the registry credentials as we showed in the first part of this blog post:

```sh
kubectl get deploy kyverno -n kyverno -o yaml | grep containers -A5
      containers:
      - args:
        - --imagePullSecrets=regcred
        env:
        - name: INIT_CONFIG
          value: kyverno
```

Once we have Kyverno installed let's install the Tekton Tasks and Pipelines into the namespace selected.

* To deploy that we're using another ArgoCD Application:

```sh
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: signed-pipelines-demo
  namespace: openshift-gitops
spec:
  destination:
    namespace: workshop
    server: https://kubernetes.default.svc
  project: default
  source:
    path: sign-images/manifests/
    repoURL: https://github.com/rcarrata/ocp4-network-security
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  ignoreDifferences:
  - group: kyverno.io
    kind: ClusterPolicy
    jsonPointers:
    - /spec/rules
  - group: kyverno.io
    kind: Policy
    jsonPointers:
    - /spec/rules
```

as you can check we're using the [ignoreDifferences](https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/#application-level-configuration) pattern in this case, to ensure that ArgoCD ignores differences within some CRD of Kyverno such as ClusterPolicy or Policy.

For what reason? If we don't do that, we will have OutOfSync this object in our ArgoCD Application, because it's synched and managed by the Kyverno controller that changes dynamically the CR and it's content shows differences with the Git server manifest, showing this OutOfSync.

* Let's apply the Tetkon Tasks and Pipelines, and our Kyverno ClusterPolicy too:

```sh
kubectl apply -f argocd/kyverno-app.yaml
```

* After a few minutes we will have synched all the objects that are in the [GitHub repository path](https://github.com/rcarrata/ocp4-network-security/tree/main/sign-images/manifests), including the [Kyverno ClusterPolicy for Checking Images](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/manifests/kyverno-check-images-cluster-policy.yaml) that we checked in the first blog post:

[![](/images/signing5.png "signing5.png")]({{site.url}}/images/signing5.png)

* With that we will have our 2 ArgoCD Applications in sync:

[![](/images/signing1.png "signing1.png")]({{site.url}}/images/signing1.png)

### 3.2 Deploy Tekton Tasks and Kyverno with cli

Let's deploy our Tekton Tasks and other objects using the CLI. If you followed the step before, skip this step because we installed all of the needed for continue, so jump to the next step.

* Clone the repository and change the directory to access to the demo:

```sh
git clone https://github.com/rcarrata/ocp4-network-security.git
cd ocp4-network-security/sign-images
```

* Apply all the Tasks and ClusterTasks:

```sh
kubectl apply -k manifests/
```

## 4. Running Signed Pipeline

Now that we have deployed our Tekton Tasks and Tekton Pipelines, let's test them running the [Signed Tekton Pipeline](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/manifests/sign-images-pipeline.yaml).

* But before to start, we need to add the imagePullSecrets to the ServiceAccount "pipeline" and "default":

```sh
export NAMESPACE=workshop
export SERVICE_ACCOUNT_NAME=pipeline
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
  -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
kubectl patch serviceaccount default \
 -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
```

this is only necessary because we need to allow our Tekton Pipeline/Tasks and the Kubelet also to be able to access the GitHub Registry using the credentials that we defined in the first blog post.

* Let's run the pipeline using a [PipelineRun](https://tekton.dev/docs/pipelines/pipelineruns/):

```sh
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: build-and-deploy-signed
  namespace: workshop
  labels:
    tekton.dev/pipeline: build-and-deploy-signed
spec:
  params:
    - name: deployment-name
      value: pipelines-vote-api
    - name: git-url
      value: 'https://github.com/openshift/pipelines-vote-api.git'
    - name: git-revision
      value: master
    - name: IMAGE
      value: 'ghcr.io/rcarrata/pipelines-vote-api:v2'
    - name: namespace
      value: workshop
  pipelineRef:
    name: build-and-deploy-signed
  serviceAccountName: pipeline
  timeout: 1h0m0s
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: source-pvc
```

as we can check we defined in the PipelineRun several interesting things that we can check:

1. Tekton Params to be used:

```sh
- deployment-name -> name of the image and the 
- git-url         -> git url with the source code / Dockerfile / k8s manifests
- git-revision    -> git branch with the source code / Dockerfile / k8s manifests
- IMAGE           -> container image generated and pushed into the GitHub Registry
- namespace       -> where is deployed this demo 
```

2. pipelineRef: pipeline to be executed.

3. serviceAccountName: SA to be used within the pipeline steps (pipeline)

4. workspace: shared workspace across the steps. Tekton Tasks executed will be using this workspace, that will be a PersistentVolumeClaim.

* Now that we checked all the parameters, let's create the pipelineRun:

```sh
kubectl create -f run/sign-images-pipelinerun.yaml
```

* After that the Build and Deploy Signed pipeline will start with the parameters described, going though each step declared before:

[![](/images/signing9.png "signing9.png")]({{site.url}}/images/signing9.png)

* We can check as we have the container image pushed into the GitHub Registry, as well as the Signature generated by cosign in the sign

[![](/images/signing11.png "signing11.png")]({{site.url}}/images/signing11.png)

* After that we can check as our K8s Deployment is successfully generated and updated with the image generated in this step:

```sh
kubectl get deploy pipelines-vote-api -o jsonpath='{.spec.template.spec.containers[0].image}'
ghcr.io/rcarrata/pipelines-vote-api:v2@sha256:c6da814d4ed548d6bad1336521375803d0c8f441112df845b02377215c639f26
```

if we checked the tag image used, corresponds with the tag published in the GitHub Registry, matching the **c6da814d4ed548d6bad1336521375803d0c8f441112df845b02377215c639f26** sha256 hash.

* If we check the Deployment, we will see that our application is up && running within our pod:

```sh
kubectl get deploy -l app=pipelines-vote-api
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
pipelines-vote-api   1/1     1            1           1m
```

and the pods are in Running state:

```sh
kubectl get pod -l app=pipelines-vote-api
NAME                                  READY   STATUS    RESTARTS   AGE
pipelines-vote-api-68c67bd97b-wwbts   1/1     Running   0          2m
```

This application could be deployed, because passed the Kyverno verifyImages rule that we implemented and uses Cosign to verify container image signatures within the Kyverno ClusterPolicy deployed in earlier steps, and explained in the previous post:

```sh
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image
spec:
  validationFailureAction: enforce
  background: false
  webhookTimeoutSeconds: 30
  failurePolicy: Fail
  rules:
    - name: check-image
      match:
        any:
        - resources:
            kinds:
              - Pod
      verifyImages:
      - image: "ghcr.io/rcarrata/pipelines-vote-api:*"
        key: |-
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEo31F9RmEeZHCQexbv0vUyIKuFHXR
          YM3Brq4nDqKQ3/3YRsAHgXeNOWZwemOnoZCJOkjDlM4CzkcrmGUCT7dmpw==
          -----END PUBLIC KEY-----
```

as we can check the key is a public key generated by cosign and it's used by Kyverno + Cosign for verify the image signature stored in the GitHub Registry.

This policy will validate that all images that match ghcr.io/rcarrata/pipelines-vote-api:* are signed with the specified key that is stored as a secret in the cluster and contains the private key used to sign the container image.

The policy rule check fails if the signature is not found in the OCI registry, or if the image was not signed using the specified key.

## 5. Running Unsigned Pipeline

Let's try to Hack our pipeline!

For this, let's think that some rogue user / hacker gained intel about where is the source code, and have the ability to build a Pipeline to generate the image, introducing some CVEs that he can exploit to gain access to our systems.

If we have not implemented a proper Sign and Verify steps in our pipelines and within our K8s / OCP clusters for our container images, the Hacker can deploy a rogue container image with some backdoor simulating that is our legit application, with the exact same name as is deployed, introducing risks that can be fatal for our supply chain security.

How to fix this? Let's see the power of Signing and Verifying container images using Kyverno and Sigstore!

We will use a basic pipeline of CLONE -> BUILD -> PUSH -> DEPLOY but without the step of signing. Let's call this pipeline [build-and-deploy-unsigned](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/manifests/unsigned-images-pipeline.yaml).

Let's check the PipelineRun that we create to execute the Unsigned Pipeline that will try to hack our Supply Chain:

```sh
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: build-and-deploy-unsigned
  namespace: workshop
  labels:
    tekton.dev/pipeline: build-and-deploy-unsigned
spec:
  params:
    - name: deployment-name
      value: pipelines-vote-api
    - name: git-url
      value: 'https://github.com/openshift/pipelines-vote-api.git'
    - name: git-revision
      value: master
    - name: IMAGE
      value: 'ghcr.io/rcarrata/pipelines-vote-api:unsigned'
    - name: namespace
      value: workshop
  pipelineRef:
    name: build-and-deploy-unsigned
  serviceAccountName: pipeline
  timeout: 1h0m0s
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: source-pvc
```

as we can check, we will be generate and push a container image that will have the tag of unsigned, and that won't be signed by Cosign with our Keypair.

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

[![](/images/signing12.png "signing12.png")]({{site.url}}/images/signing12.png)

* The Pipeline will clone the source code, and will generate the container image without signing and will push using Buildah to the GitHub Registry:

[![](/images/signing13.png "signing13.png")]({{site.url}}/images/signing13.png)

We're hacked! Right?

Not really :) Kyverno and ClusterPolicies to the rescue!

* The pipeline will fail because, in the last step the pipeline will try to deploy the Deployment using the Rogue/Hacked/Unsigned image within the Deployment, and the Kyverno Cluster Policy will deny the request because it's the container image is **not signed with the Cosign keypair** within the cosign step using the same private key as the pipeline before:

[![](/images/signing6.png "signing6.png")]({{site.url}}/images/signing6.png)

we need to remember that the Kyverno policy rule check fails if the signature is not found in the OCI registry, or if the image was not signed using the specified key.

* As we can see the error that the pipeline outputs it's due to the Kyverno Cluster Policy with the image signature mismatch error:

[![](/images/signing7.png "signing7.png")]({{site.url}}/images/signing7.png)

So we're saved! We protected our Supply Chain increasing the security and hardening of our CICD processes!

And with that we finished our second blog post about Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno.

Happy Hacking!

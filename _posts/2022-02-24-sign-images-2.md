---
layout: post
title: Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno - (Part II)
date: 2022-02-29
type: post
published: false
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: false
---

How I can Sign and Verify container images within the CICD Pipelines (we're using Tekton but is still valid in any CICD pipeline)

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

<img align="center" width="670" src="assets/signing0.png">

as you can check there are some standard steps that we are familiar with, and follows the three steps of the most basic CICD pipeline:

```sh
CLONE -> BUILD -> DEPLOY
```

1. **Clone / Fetch Repository** - For this secure demo pipeline, we're using a [Voting app example](https://github.com/openshift/pipelines-vote-api) written in Golang. The ClusterTask of git-clone is used in this step cloning the repository in a workspace called "shared-workspace". Task available in [Tekton Hub](https://hub.tekton.dev/tekton/task/git-clone).

2. **Buildah Custom** - These Buildah Custom step is divided in 3 different substeps within the [Buildah-Custom ClusterTask](https://github.com/rcarrata/ocp4-network-security/blob/main/sign-images/pipelines/clustertasks-buildah-custom.yaml)
In a nutshell these tasks are Build, Push to the registry and Collect the Image Digest to use it in further steps.
Why don't use the Buildah task out of the box? Because this specific task uses the Registry Credentials for allow Buildah to push the images to a private Registry (GitHub registry in this case, but private repository).

3. **Sign Images** - That's the interesting part that we want to deep dive a bit, because it's the part when we use Cosign to sign our image when it's built and pushed to the registry.

## 3. Deploying the Tekton Tasks and Pipeline

To run this pipeline first you need to create / apply all the Tekton Tasks and Pipelines that we will be using during this blog post.

IMPORTANT: This blog post assume that we've followed all the steps of the previous blog post including installing Tekton, generate keypairs, among other things.

* Clone the repository and change the directory to access to the demo:

```sh
git clone https://github.com/rcarrata/ocp4-network-security.git
cd ocp4-network-security/sign-images
```

* Apply all the Tasks and ClusterTasks

```sh
kubectl apply -k manifests
```

```sh
export NAMESPACE=workshop
export SERVICE_ACCOUNT_NAME=pipeline
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
  -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
kubectl patch serviceaccount default \
 -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n $NAMESPACE
```

## 4. Running Signed Pipeline

```sh
kubectl create -f run/sign-images-pipelinerun.yaml
```

## 5. Running Unsigned Pipeline

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

* The pipeline will fail because, in the last step the pipeline will try to deploy the image, and the Kyverno Cluster Policy will deny the request because it's not signed with the cosign step using the same private key as the pipeline before:

<img align="center" width="970" src="assets/unsigned-1.png">

* As we can see the error that the pipeline outputs it's due to the Kyverno Cluster Policy with the image signature mismatch error:

<img align="center" width="570" src="assets/unsigned-2.png">

---
layout: post
title: Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno - (Part II)
date: 2022-02-27
type: post
published: true
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

## Overview

In the previous blog post we've demonstrate how Sigstore is an awesome project focused on software signing and transparency log technologies and can help us to improve software supply chain security.

Cosign is a tool within this project that provides image-signing and verification, so we can sign our container images, to ensure that our container image stored in our registry is signed properly and also the signature is stored within the Registry and always available to be verified.

## Running Unsigned Pipeline

* Run the pipeline for build the image and push to the GitHub registry, but this time without sign with cosign private key:

```bash
kubectl create -f run/unsigned-images-pipelinerun.yaml
```

* The pipeline will fail because, in the last step the pipeline will try to deploy the image, and the Kyverno Cluster Policy will deny the request because it's not signed with the cosign step using the same private key as the pipeline before:

<img align="center" width="970" src="assets/unsigned-1.png">

* As we can see the error that the pipeline outputs it's due to the Kyverno Cluster Policy with the image signature mismatch error:

<img align="center" width="570" src="assets/unsigned-2.png">

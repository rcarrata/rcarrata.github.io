---
layout: post
title: Signing and Verifying Images to Secure CICD pipelines using Sigstore and Kyverno
date: 2022-02-23
type: post
published: true
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How can we improve our supply chain security by signing container images, in an open, accessible and transparent manner? How can we store the signatures in a safe and organized way?

And how can we ensure that no one can introdce a risk to our entire software supply chain by deploying malicious images in our Kubernetes clusters?

Let's dig in!

## Overview

The answer to securing the software supply chains lies in digitally-signing the various artifacts that comprise our applications, from binaries and containers to aggregated files (like tarballs) and software-bills-of-materials (SBOM).

Digital signatures effectively "freeze" an object in time, indicating that in it's current state it is verified and it hasn’t been altered in any way.

Sigstore offers a method to better secure software supply chains in an open, transparent and accessible manner.

Think of Sigstore as a new standard for signing, verifying and protecting software and automating how you digitally sign and check components, for a safer chain of custody tracing software back to the source.

So with Sigstore, we can generate the key pairs needed to sign and verify artifacts, automating as much as possible so there’s no risk of losing or leaking them.

On the other hand, you can se benefit from the Transparent ledger technology. This means that anyone can find and verify signatures, and check whether someone’s changed the source code, the build platform or the artifact repository.

But wait a moment, we can sign the images, but how we can ensure that only signed images that are valid and verified can be deployed to our clusters? Manually?

Let's bring Kyverno to the stage!

Kyverno is a policy engine designed for Kubernetes. Using Kyverno, policies are managed as Kubernetes resources and no new language is required for writing policies.

This allows using familiar tools such as kubectl, git, and kustomize to manage policies. Kyverno policies can validate, mutate, and generate Kubernetes resources. The Kyverno CLI can be used to test policies and validate resources as part of a CI/CD pipeline.

In a nutshell, Kyverno offers an image verification that uses the Cosign component from the Sigstore project.

Let's start to have fun with Sigstore, Tekton, Kyverno and Kubernetes!

NOTE: this blog post uses Tekton but any CICD can like Jenkins, GitHub Actions, etc. can be used instead. 

NOTE: All the code / examples used in this blog post are [available in a GitHub repository](https://github.com/rcarrata/ocp4-network-security/tree/main/sign-images). Check this out!

## 1. Installing OpenShift Pipelines / Tekton in Kubernetes / OpenShift

* **Option 1** - Install OpenShift Pipelines in OpenShift using the Operator:

```sh
cat bootstrap/openshift-pipelines-operator-subscription.yaml

apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

* Apply the OpenShift Pipelines Subscription in the cluster:

```sh
kubectl apply bootstrap/openshift-pipelines-operator-subscription.yaml


```

* Check that the OpenShift Pipelines are properly installed:

```sh
kubectl get csv | grep pipelines

openshift-pipelines-operator-rh.v1.6.2   Red Hat OpenShift Pipelines   1.6.2     redhat-openshift-pipelines.v1.5.2   Succeeded
```

* **Option 2** - Install Tekton pipelines (upstream) in Kubernetes vanilla:

```sh
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

kubectl wait -n tekton-pipelines \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/part-of=tekton-pipelines,app.kubernetes.io/component=controller \
  --timeout=90s
```

* Install Tekton Dashboard to check the Tekton Pipelines:

```sh
curl -sL https://raw.githubusercontent.com/tektoncd/dashboard/main/scripts/release-installer | \
   bash -s -- install latest

kubectl wait -n tekton-pipelines \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/part-of=tekton-dashboard,app.kubernetes.io/component=dashboard \
  --timeout=90s
```

* Deploy the Ingress CR to expose the Dashboard:

```sh
kubectl apply -n tekton-pipelines -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tekton-dashboard
spec:
  rules:
  - host: tekton-dashboard.$(hostname -I | awk '{print $1}').nip.io
    http:
      paths:
      - pathType: ImplementationSpecific
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
EOF
```

## 2. Install Kyverno in OpenShift / Kubernetes

Now it's time to install Kyverno in our OpenShift / Kubernetes clusters. We will be using Helm to install Kyverno as per [documentation](https://kyverno.io/docs/installation/#install-kyverno-using-helm).

* Add the Kyverno Helm repository:

```sh
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

* Then Install Kyverno using Helm:

```sh
helm install kyverno --namespace kyverno kyverno/kyverno --create-namespace
```

* Check that Kyverno pods are Running state:

```sh
kubectl get pod -n kyverno
NAME                      READY   STATUS    RESTARTS   AGE
kyverno-55f86d8cd-69pzv   1/1     Running   0          2m3s
```

* Check the new CRDs that are available once the Kyverno components are installed:

```sh
kubectl api-resources | grep kyverno
clusterpolicies                       cpol               kyverno.io/v1                                 false        ClusterPolicy
clusterreportchangerequests           crcr               kyverno.io/v1alpha2                           false        ClusterReportChangeRequest
generaterequests                      gr                 kyverno.io/v1                                 true         GenerateRequest
policies                              pol                kyverno.io/v1                                 true         Policy
reportchangerequests                  rcr                kyverno.io/v1alpha2                           true         ReportChangeRequest
```

## 3. Set up the Registry credentials in the Kubernetes cluster

In this blog, we're using the GitHub registry or ghcr.io as our container registry for storing the images and the signatures. However, there are more options available such aS [Container Image Registries supported by Sigstore](https://github.com/sigstore/cosign#registry-support).

By default the container images stored in GitHub registries are private, so we need to add the registry credentials to our Kubernetes clusters in order to allow the Tekton pipelines, and the Kyverno to read/write (pull/push) container images from GitHub Registry.

* Export the token for the GitHub Registry / ghcr.io:

```sh
export PAT_TOKEN="xxx"
export EMAIL="xxx"
export USERNAME="xxx"
export NAMESPACE="workshop"
```

NOTE: no regular password can be used with GH Registry, a [Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) needs to be generated and used to the authentication.

* Create the namespace for the demo example:

```sh
kubectl create ns $NAMESPACE
```

* Generate a docker-registry secret with the credentials for GitHub Registry to push/pull the images and signatures:

```sh
kubectl create secret docker-registry ghcr-auth-secret --docker-server=ghcr.io --docker-username=${USERNAME} --docker-email=${EMAIL} --docker-password=${PAT_TOKEN} -n ${NAMESPACE}
```

NOTE: This secret is located in the ${NAMESPACE} because the Tekton Pipelines will be running in this specific namespace and needs to access to the Registry to pull/push.

* Add the imagePullSecret to the ServiceAccount "pipeline" in the namespace of the demo:

```sh
export SERVICE_ACCOUNT_NAME=pipeline
kubectl patch serviceaccount $SERVICE_ACCOUNT_NAME \
  -p "{\"imagePullSecrets\": [{\"name\": \"ghcr-auth-secret\"}]}" -n $NAMESPACE
```

* We also need to set up the credentials in the Kyverno namespace, to allow Kyverno to download the signature from GitHub Registry.

```sh
kubectl create secret docker-registry regcred --docker-server=ghcr.io --docker-username=${USERNAME} --docker-email=${EMAIL}--docker-password=${PAT_TOKEN} -n kyverno
```

* Add the imagePullSecret to the Kyverno ServiceAccount:

```sh
kubectl patch serviceaccount kyverno \
  -p "{\"imagePullSecrets\": [{\"name\": \"regcred\"}]}" -n kyverno

```

NOTE: This is optional, but we're using this to ensure that the Kyverno ServiceAccount is able to access the secret using the registry credentials.

* We then need to patch/update the Kyverno Deployment to include the imagePullSecrets with the Registry credentials:

```sh
kubectl get deploy kyverno -n kyverno -o yaml | grep containers -A5
      containers:
      - args:
        - --imagePullSecrets=regcred
        env:
        - name: INIT_CONFIG
          value: kyverno
```

NOTE: The imagePullSecrets are used as an argument for the Kyverno binary inside of the deployment/pod, assigning the secret containing the registry credentials, so that it can access the GitHub registry container images and signatures.

Now we're set to install and generate our first keypair with Sigstore tools.

## 4. Installing Cosign and Rekor

In the Sigstore project, one of the most useful tools is [Cosign](https://docs.sigstore.dev/cosign/overview). Cosign supports container signing, verification and storage in an OCI registry. Cosign aims to make signatures invisible infrastructure.

Cosign supports:

* Hardware and KMS signing
* Bring-your-own PKI
* Our free OIDC PKI (Fulcio)
* Built-in binary transparency and timestamping service (Rekor)
* Kubernetes policy enforcement
* Rego and Cuelang integrations for policy definition

In a nutshell, Cosign is a tool within the Sigstore project that greatly simplifies how content is signed and verified by storing signatures from container images and other types in OCI registries.

By default the signatures produced by cosign are stored in the same OCI registry as another tag with a predictive name like:

```sh
ghcr.io/myuser/myimage:sha256-<DIGEST>.sig
```

* Let's install Cosign in our localhost (Fedora35 in my case):

```sh
go install github.com/sigstore/cosign/cmd/cosign@latest
```

NOTE: Check the [Cosign Installation documentation](https://docs.sigstore.dev/cosign/installation/) for another installation methods / platforms.

* Verify of one known container image with the Public Key:

```sh
cosign verify --key https://raw.githubusercontent.com/tektoncd/chains/main/tekton.pub gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.28.1

Verification for gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.28.1 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.

[{"critical":{"identity":{"docker-reference":"gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller"},"image":{"docker-manifest-digest":"sha256:0c320bc09e91e22ce7f01e47c9f3cb3449749a5f72d5eaecb96e710d999c28e8"},"type":"Tekton container signature"},"optional":{}}]
```

It works! This command returns 0 if at least one cosign formatted signature matching the public key is found.

## 5. Generating KeyPair with Cosign

To sign content using cosign, a public/private keypair must be generated. Cosign can use keys stored in Kubernetes Secrets to so sign and verify signatures.

In order to generate a secret you have to pass the cosign generate-key-pair command a k8s://[NAMESPACE]/[NAME] URI specifying the namespace and secret name:

```sh
cosign generate-key-pair k8s://${NAMESPACE}/cosign
```

After generating the key pair, cosign will store it in a Kubernetes secret using your current context. The secret will contain the private and public keys, as well as the password to decrypt the private key.

```sh
kubectl get secret -n $NAMESPACE cosign -o yaml
apiVersion: v1
data:
  cosign.key: xxxx
  cosign.password: xxxx
  cosign.pub: xxxx
immutable: true
kind: Secret
```

The cosign command above prompts the user to enter the password for the private key. The user can either manually enter the password, or set an environment variable COSIGN_PASSWORD.

When verifying an image signature, using cosign verify, the key will be automatically decrypted using the password stored in the Kubernetes secret under cosign.password.

To check more useful tips with Cosign check the [Detailed Usage documentation](https://github.com/sigstore/cosign/blob/main/USAGE.md).

## 6. Signing Image with Cosign and storing the signature in the registry

* Retrieve the Cosign Key from the Cosign secret in your k8s/ocp cluster:

```sh
kubectl get secret -n ${NAMESPACE} cosign -o jsonpath='{.data.cosign\.key}' | base64 -d > cosign.key
```

NOTE: this is only necessary if you're signing images with cosing OUTSIDE of the cluster, because the Cosign private key is stored within the k8s secret.

* Pull an example image (we're using th Ubi8 image in this example), and tag them to use the ghcr.io registry in our Organization:

```sh
podman pull registry.access.redhat.com/ubi8/ubi-minimal:8.5-230
podman tag registry.access.redhat.com/ubi8/ubi-minimal:8.5-230 ghcr.io/${USERNAME}/ubi-minimal:8.5-230
```

* Login to the GitHub registry using the PAT token:

```sh
echo $PAT_TOKEN | podman login ghcr.io -u ${USERNAME} --password-stdin
Login Succeeded!
```

* Push the image to the GitHub registry:

```sh
podman push --remove-signatures ghcr.io/${USERNAME}/ubi-minimal:8.5-230
```

NOTE: We're using the --remove-signatures, because the image is pulled from the registry.access.com. By default the Podman ecosystem treats signatures as an integral part of the image. So "--remove-signatures" [is necessary](https://access.redhat.com/solutions/6568941) to make a copy that doesn’t include the signatures, including copies that don’t preserve the images bit-for-bit-exactly.

* Sign a container and store the signature in the registry:

```sh
cosign sign --key cosign.key ghcr.io/rcarrata/ubi-minimal:8.5-230
Enter password for private key:
Pushing signature to: ghcr.io/rcarrata/ubi-minimal
```

The cosign command above prompts the user to enter the password for the private key. The user can either manually enter the password, or set an environment variable COSIGN_PASSWORD with the value of the password. 

* Signatures are uploaded to an OCI artifact and stored with a predictable name. The signature can be located with the cosign triangulate command:

```sh
cosign triangulate ghcr.io/rcarrata/ubi-minimal:8.5-230
ghcr.io/rcarrata/ubi-minimal:sha256-c8c13c505681f6e926cfe1cd260fd94c7414a273c528a060e2ff69aefa358a8b.sig
```

## 7. Installing Crane for inspect remote signatures

[Crane](https://github.com/google/go-containerregistry/tree/main/cmd/crane) is a tool for interacting with remote images and registries.

Crane has useful tips and tricks than could be helpful for interacting with remote registries. In this case we are using it to inspect the signature generated by cosign and uploaded into the Registry.

* Let's first install Crane in our localhost (I'm using Fedora but you can use whenever [installation method]() that it's suitable for you).

* We then use the crane manifest command and pass on the output of the cosign triangulate (that represent the signature location within the Registry):

```sh
crane manifest $(cosign triangulate ghcr.io/rcarrata/ubi-minimal:8.5-230) | jq -r .
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 233,
    "digest": "sha256:dcab9b63d8e66e98c30876d63b7414c4d22334ff90c55e9b906b4bc36297144d"
  },
  "layers": [
    {
      "mediaType": "application/vnd.dev.cosign.simplesigning.v1+json",
      "size": 244,
      "digest": "sha256:bea87fb8bbda92538339b53614963c6d00a0de4409c05266ad3c6b1ca2dcac6d",
      "annotations": {
        "dev.cosignproject.cosign/signature": "MEUCIQCzVLWZaPQpEzvWbVDQOe/MX1h7fq+9TUP2Tpn6ZAkJaAIgKkj0EGvFfkejl2gWOCPo6Lf/1mtZ2WklmKDT5DqQIk0="
      }
    }
  ]
}
```

As you can see this command outputs the signature json schema so that we can check the digest and annotations. 

## 8. Verify a Container Image against a public key

* Let's now verify our container image against a public key generated by Cosign:

```sh
cosign verify --key cosign.pub ghcr.io/rcarrata/ubi-minimal:8.5-230 | jq -r .

Verification for ghcr.io/rcarrata/ubi-minimal:8.5-230 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
[
  {
    "critical": {
      "identity": {
        "docker-reference": "ghcr.io/rcarrata/ubi-minimal"
      },
      "image": {
        "docker-manifest-digest": "sha256:c8c13c505681f6e926cfe1cd260fd94c7414a273c528a060e2ff69aefa358a8b"
      },
      "type": "cosign container image signature"
    },
    "optional": null
  }
]
```

This command returns 0 if at least one cosign formatted signature matching the public key is found for the image. For information and caveats on other signature formats, refer to the detailed usage below.

Any valid payloads are printed to stdout, in json format. Note that these signed payloads include the digest of the container image, which is how we can be sure these "detached" signatures cover the correct image.

## 9. Verify images in Kubernetes clusters with Kyverno

Now to ensure only the container images signed with our keypair are deployed and no malicious container images enter our environment, we can use [Kyverno](https://kyverno.io/docs/writing-policies/verify-images/). Kyverno is a policy engine designed specifically for Kubernetes.

Kyverno runs as a dynamic admission controller in a Kubernetes cluster. Kyverno receives validating and mutating admission webhook HTTP callbacks from the kube-apiserver and applies matching policies to return results that enforce admission policies or reject requests.

Policy enforcement is captured using Kubernetes events. Kyverno also reports policy violations for existing resources

Two of the useful features of Kyverno that we can implement in our use cases here are as follows:

* Verify container images for software supply chain security
* Inspect image metadata

Focusing in these use cases, the Kyverno **verifyImages** rule uses Cosign to verify container image signatures, attestations and more stored in an OCI registry.

The rule matches an image reference (wildcards are supported) and specifies a public key to be used to verify the signed the image or attestations.

We first use an image verification policy defined in a ClusterPolicy CR of Kyverno.

* Let's define the image to be verified (without any tag or SHA digest), and specify the public key generated by Cosign in earlier steps of this blog:

```sh
export IMAGE="ghcr.io/rcarrata/ubi-minimal"
```

* Then we need to specify a Kyverno ClusterPolicy CR that will check the container image signatures and add the digests:

```sh
cat <<EOF > cpol-image-check.yaml
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
      - image: "$IMAGE:*"
        key: |-
$(cat cosign.pub | sed 's/^/          /')
EOF
```

As you can see, by default,  the validationFailureAction is set to enforced, and the failurePolicy is set to Fail.

The verifyImages rule from the ClusterPolicy in Kyverno checks all the container image signatures that matches, and as you notice we're using a wildcard * to validate all the tags generated.

We also need to specify our public key inorder to verify the signed image.

* Let's check that we are able to deploy a pod using our signed imaged from our GitHub Registry:

```sh
kubectl run myawesomeapp --image=ghcr.io/rcarrata/ubi-minimal:8.5-230
pod/myawesomeapp created
```

* As you can see verifyImage rule in the Kyverno Cluster Policy checks against the public key defined in the step before and everything works ok. 

```sh
kubectl get pod
NAME           READY   STATUS              RESTARTS   AGE
myawesomeapp   0/1     ContainerCreating   0          2s
```

## 10. Testing Kyverno Cluster Policy to ensure that ONLY our signed images can be deploy in K8s/OCP

The policy rule check fails if the signature is not found in the OCI registry, or if the image was not signed using the specified key. And because of the validationFailureAction is set to enforce, Kyverno will not allow to deploy the image to the k8s/ocp clusters.

Let's see that in action!

* First of all, let's push the same image but with different tag and without signing them with our keypair and cosign tool:

```sh
podman pull registry.access.redhat.com/ubi8/ubi-minimal:8.4-212
podman tag registry.access.redhat.com/ubi8/ubi-minimal:8.4-212 ghcr.io/${USERNAME}/ubi-minimal:nosignedandnotsecure
podman push ghcr.io/${USERNAME}/ubi-minimal:nosignedandnotsecure
```

* Let's try to run the pod using an image that is NOT signed by cosign keypair:

```sh
kubectl run hackedpod --image=ghcr.io/${USERNAME}/ubi-minimal:nosignedandnotsecure
Error from server: admission webhook "mutate.kyverno.svc-fail" denied the request:

resource Pod/test/hackedpod was blocked due to the following policies

check-image:
  check-image: 'image verification failed for ghcr.io/rcarrata/ubi-minimal:nosignedandnotsecure:
    signature mismatch'
```

and voilà!

Kyverno save the day (and our Software Supply Chain as well) by preventing the deployment of the pod using a non signed and verified container image.

NOTE: this example uses Kyverno, but other Policy Engines can be used. [compatible with Cosign](https://docs.sigstore.dev/cosign/overview#kubernetes-integrations) such as [OPA](https://github.com/sigstore/cosign-gatekeeper-provider) or [Conaisseur](https://github.com/sse-secure-systems/connaisseur#what-is-connaisseur).

This concludes the first part of my blog post around Image Signing and Verification using Sigstore and Kyverno!

Check out the [second part of this blog post](https://rcarrata.com/kubernetes/sign-images-2/), where we implement a Tekton Pipeline to automate the entire process and see how Kyverno and Sigstore can help secure the software supply chain!

Happy hacking!

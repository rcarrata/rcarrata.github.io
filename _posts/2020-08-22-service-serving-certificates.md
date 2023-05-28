---
layout: post
title: Secure inter-cluster traffic in OpenShift with Service CA Operator
date: 2020-08-22
type: post
published: true
status: publish
categories:
- OpenShift
tags: []
author: rcarrata
comments: true
---

How I can secure the inter-cluster communication within two pods running into OpenShift? How can I
secure one service calling another service using SSL both running OpenShift in a easy way?

Let's dig in!

## 1. Overview

If we want to secure the communication between two pods / services inside OpenShift we have commonly
two options:

* The application manages its own certificates, and the certificates are bundled into the
  application images, and reachable with routes of type passthough (the connection is not
  encrypted by the reverse proxy)

* Utilize OpenShift generation of certificates using the Service CA Operator.

## 2. Prerequisites and environment

This blog post is tested and runs in every OCP4.x environment, but its specifically tested in a
4.4.5 version in AWS.

No specific configuration is needed (out of the box).

## 3. OpenShift Service CA Operator

The openshift-service-ca-operator is an OpenShift Cluster Operator that runs in the cluster by
default:

```
$ oc get co service-ca
 NAME         VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
 service-ca   4.4.12    True        False         False      4d4h
```

The Custom Resource Definition that enables this operator (among others) is servicecas.operator.openshift.io, and can be viewed in a cluster with:

```
$ oc get crd servicecas.operator.openshift.io -o yaml
```

And the servicecas object reflects the configuration of the operator and the managementState
(Managed):

```
$ oc get servicecas cluster --export -o yaml
apiVersion: operator.openshift.io/v1
kind: ServiceCA
metadata:
  generation: 1
  name: cluster
  selfLink: /apis/operator.openshift.io/v1/servicecas/cluster
spec:
  logLevel: "4"
  managementState: Managed
  replicas: 1
  version: 4.0.0
```

The Service CA Operator runs the operator in the openshift-service-ca-operator namespace:

```
oc get pod
NAME                                   READY   STATUS    RESTARTS   AGE
service-ca-operator-5f596775f8-gwlpj   1/1     Running   1          2d7h
```

The OpenShift Service CA Operator runs the following OpenShift controllers:

```
$ oc get pod -n openshift-service-ca
NAME                                            READY   STATUS    RESTARTS   AGE
apiservice-cabundle-injector-64cfdd7645-h4dqf   1/1     Running   3          2d7h
configmap-cabundle-injector-67ffc677d5-prn7b    1/1     Running   3          2d7h
service-serving-cert-signer-b5665b6f5-lx7bl     1/1     Running   3          2d7h
```

## 4. Service CA Certificates in OpenShift 4

For OpenShift 4.x the service ca certificates are managed by the service CA operator.

The key and certificate are in a secret in the namespace openshift-service-ca as signing-key secret:

```
$ oc get secrets -n openshift-service-ca signing-key -o "jsonpath={.data['tls\.crt']}" |  base64 -d | openssl x509 -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1181511372096086528 (0x106592553fdf2200)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = openshift-service-serving-signer@1596638040
        Validity
            Not Before: Aug  5 14:34:00 2020 GMT
            Not After : Oct  4 14:34:01 2022 GMT
        Subject: CN = openshift-service-serving-signer@1596638040
...
...
```

as we can check, the signing key certificate is generated and and issued by the openshift service serving signer.

## 5. Service CA Operator Controllers

The Service CA operator runs the following controllers:

* **Serving Cert Signer**: Issues a signed serving certificate/key pair to services annotated with
  'service.beta.openshift.io/serving-cert-secret-name' via a secret.

* **Configmap cabundle injector**: Watches for configmaps annotated with
  'service.beta.openshift.io/inject-cabundle=true' and adds or updates a data item (key
  "service-ca.crt") containing the PEM-encoded CA signing bundle.

* **Generic cabundle injector**: Watches for apiservices, mutatingwebhookconfig, validatingwebhookconfig and crds annotated with 'service.beta.openshift.io/inject-cabundle=true' and sets the appropriate ca bundle field with a base64url-encoded CA signing bundle

The controllers that we are interested in is the Configmap cabundle injector, because is capable to with an annotation to the configmap, injects the service CA certificate into the service-ca.crt

An important note here, is that the access to this CA certificate allows TLS clients to verify connections to services using this service serving certificates, and validating this way the secure connection inside the cluster.

In order to compare the service-ca.crt that are injected into the configmaps by the service ca operator, to the original signing-key secret generated into the openshift-service-ca, we need to extract the signing-key certificate (in the openshift-service-ca namespace) and one example of the service-ca.crt injected by the service ca operator:

```
$ oc get cm service-ca-bundle  -n openshift-insights -o jsonpath="{.data['service-ca\.crt']}" > /tmp/ca1.crt
$ oc get secrets -n openshift-service-ca signing-key -o jsonpath="{.data['tls\.crt']}" |  base64 -d | openssl x509 > /tmp/ca2.crt
```

If we compare the two extracted certificates with openssl md5, we noticed that are the same file:

```
$ openssl md5 /tmp/ca1.crt
MD5(/tmp/ca1.crt)= 4072c8d1c32d38bb659cc506f14a81d1
$ openssl md5 /tmp/ca2.crt
MD5(/tmp/ca2.crt)= 4072c8d1c32d38bb659cc506f14a81d1
```

this is because the service-ca.crt certificate is automatically injected by the service ca operator,
and the operator injects the same exact cert of the signing-key.

## 6. Benefits of Service Serving Certificates

And how this helps to me?

* With the OpenShift Service CA operator, you can dynamic generate the certificates, allowing
secure connections (TLS) to services that utilize service-service certificates.

This certificates will added automatically using the configmap ca bundle injector annotation:

```
$ oc get cm service-ca-bundle  -n openshift-insights -o yaml | yq .metadata.annotations
 {
     "service.beta.openshift.io/inject-cabundle": "true"
 }
```

This will allow that the consumers of the configmap can then trust service-ca.crt in their TLS
client configurations, allowing connections to these services that uses the service-serving certs.

* Furthermore, another benefit is that the certificate and key dynamically generated by the CA
  operator are automatically replaced when they get close to expiration. The service CA certificate,
  which signs the service certificates, is only valid for one year after OCP is installed.

## 7. Conclusion and Next Steps

OpenShift Service CA Operator helps other operators and elements inside of the cluster to generate
certificates that can secure communication between services, and we can benefit also of this
operator generating dynamically certificates for our microservices / apps running in our cluster.

In the next blog post we will analyse a real example, using the Service CA Operator to secure the
communication between two microservices without using non external service.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

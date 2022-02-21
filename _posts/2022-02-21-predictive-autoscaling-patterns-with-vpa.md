---
layout: post
title: Predictive Autoscaling Patterns using Vertical Pod Autoscaler in Kubernetes
date: 2022-02-21
type: post
published: true
status: publish
categories:
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How you can adjust your application / workloads resource requests automatically, to ensure that your pods stay up during periods of high demand, like Black Friday periods? How can we ensure than as an administrators of our Kubernetes clusters, we're implementing an appropriated cluster resource management, ensuring that the capacity planning is properly defined?

Let's dig in!

## Overview

Determining the proper values for pod resources is challenging. In an organization, it can be difficult to identify a team that has the proper insights regarding the resource requirements in production for a given application.

Translating the idea that the cluster should become aware of the necessary compute resources required into Kubernetes concepts, a controller can be used to perform this task on a set of a set of pods and provide its best recommendation for memory and cpu (and potentially other metrics).

This is what the Vertical Pod Autoscaler (VPA) does.

Vertical Pod Autoscaler (VPA) frees the users from necessity of setting up-to-date resource limits and requests for the containers in their pods.

When configured, it will set the requests automatically based on usage and thus allow proper scheduling onto nodes so that appropriate resource amount is available for each pod. It will also maintain ratios between limits and requests that were specified in initial containers configuration.

## Vertical Pod Autoscaler Operator

The Vertical Pod Autoscaler Operator (VPA) is implemented as an API resource and a custom resource (CR). The CR determines the actions the Vertical Pod Autoscaler Operator should take with the pods associated with a specific workload object, such as a daemon set, replication controller, and so forth, in a project.

The VPA automatically computes historic and current CPU and memory usage for the containers in those pods and uses this data to determine optimized resource limits and requests to ensure that these pods are operating efficiently at all times.

## When to use VPA?

For developers, you can use the VPA to help ensure your pods stay up during periods of high demand by scheduling pods onto nodes that have appropriate resources for each pod.

Administrators can use the VPA to better utilize cluster resources, such as preventing pods from reserving more CPU resources than needed.

The VPA monitors the resources that workloads are actually using and adjusts the resource requirements so capacity is available to other workloads. The VPA also maintains the ratios between limits and requests that are specified in initial container configuration.

## Installing VPA in Kubernetes / OpenShift

To use Vertical Pod Autoscaler, you need to first install them in your Kubernetes / OpenShift clusters.

* **Option 1**: To install VPA in Kubernetes vanilla clusters:

```bash
$ git clone https://github.com/kubernetes/autoscaler.git
./autoscaler/vertical-pod-autoscaler/hack/vpa-up.sh
```

* Check the pods running corresponding with the VPA components:

```bash
kubectl get po -n kube-system
NAME                                        READY   STATUS    RESTARTS   AGE
vpa-admission-controller-5b7857cc48-ppspd   1/1     Running   0          7s
vpa-recommender-6fc8c67d85-gljpl            1/1     Running   0          8s
vpa-updater-75474c9ff-bgp9d                1/1     Running   0          8s

kubectl get crd
verticalpodautoscalers.autoscaling.k8s.io 
```

* **Option 2**: To install VPA in OpenShift using VPA Operator follow the documentation about [installing the Vertical Pod Autoscaler Operator](https://docs.openshift.com/container-platform/4.9/nodes/pods/nodes-pods-vertical-autoscaler.html#nodes-pods-vertical-autoscaler-install_nodes-pods-vertical-autoscaler)

## VPA demo - Automatically adjust requests / limits when Apps are OOMKilled

Let's have some fun with VPA, implementing an use case, that will automatically adjust our request and limits of our application, avoiding to fail with an OOMKilled status when the requests are increased.

### A - Deploying Application Without VPA

Let's first deploy our application without Vertical Pod Autoscaler CR applied.

* Create a project without LimitRange

```bash
PROJECT=test-novpa-uc2-$RANDOM
```

```bash
oc new-project $PROJECT
```

* Delete any preexistent LimitRange:

```bash
kubectl -n $PROJECT delete limitrange --all
```

* Deploy stress application into the ns:

```bash
cat <<EOF | kubectl -n $PROJECT apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress-novpa
spec:
  selector:
    matchLabels:
      app: stress
  replicas: 1
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - name: stress
        image: polinux/stress
        resources:
          requests:
            memory: "100Mi"
          limits:
            memory: "200Mi"
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "250M"]
EOF
```

We defined the requests as 100Mi and the limits with 200Mi for the container stress.

On the other hand, we used the stress image, and as in the [Image Stress Documentation](https://linux.die.net/man/1/stress) is described, we can define the arguments for allocate certain amount of memory:

```md
- -m, --vm N: spawn N workers spinning on malloc()/free()
- --vm-bytes B: malloc B bytes per vm worker (default is 256MB) 
```

So we defined 250M of memory allocation by the stress process, that's more than the limits of the container is defined, **exceeding the Container's memory limit**, and this will produce an OOMKilled.

```sh
kubectl get pod
NAME                            READY   STATUS      RESTARTS   AGE
stress-novpa-7b9459559c-hrgwr   0/1     OOMKilled   0          6s
```

In Kubernetes, every scheduling decision is always made based on the resource requests. Whatever number you put there, the scheduler will use it to allocate place for your pod.

## B - Deploying Application With VPA

Now let's check how we can benefit from the VPA controller in order to avoid the OOMKilled situations, where VPA will adjust and apply the recommended resources (requests and limits) automatically without disrupting our application SLA/SLOs.

* Create a project without LimitRange

```bash
PROJECT=test-vpa-uc2-$RANDOM
```

```bash
oc new-project $PROJECT
```

* Delete any preexistent LimitRange:

```bash
kubectl -n $PROJECT delete limitrange --all
```

* Deploy stress application into the ns:

```bash
cat <<EOF | kubectl -n $PROJECT apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stress
spec:
  selector:
    matchLabels:
      app: stress
  replicas: 1
  template:
    metadata:
      labels:
        app: stress
    spec:
      containers:
      - name: stress
        image: polinux/stress
        resources:
          requests:
            memory: "100Mi"
          limits:
            memory: "200Mi"
        command: ["stress"]
        args: ["--vm", "1", "--vm-bytes", "150M"]
EOF
```

We defined the requests as 100Mi and the limits with 200Mi for the container stress.

On the other hand, we used the stress image, and as in the [Image Stress Documentation](https://linux.die.net/man/1/stress) is described, we can define the arguments for allocate certain amount of memory:

```md
- -m, --vm N: spawn N workers spinning on malloc()/free()
- --vm-bytes B: malloc B bytes per vm worker (default is 256MB) 
```

So we defined 150M of memory allocation by the stress process, that's it's between the request and limits defined.

* Check that the pod is up && running:

```sh
kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
stress-7d48fdb6fb-j46b8   1/1     Running   0          35s

kubectl logs -l app=stress
stress: info: [1] dispatching hogs: 0 cpu, 0 io, 1 vm, 0 hdd
```

* Check that the request and limits generated in Pod

```bash
kubectl get pod -l app=stress -o yaml | grep limit -A1
        limits:
          memory: 200Mi
```

```
kubectl get pod -l app=stress -o yaml | grep requests -A1
        requests:
          memory: 100Mi
```

* Check the metrics of the pod deployed:

```bash
kubectl adm top pod --namespace=$PROJECT --use-protocol-buffers
NAME                      CPU(cores)   MEMORY(bytes)
stress-7d48fdb6fb-j46b8   1019m        115Mi
```

* It is possible to set a range for the autoscaling: minimum and maximum values, for the requests. Apply the VPA with the minAllowed and maxAllowed as described:

```sh
cat <<EOF | kubectl -n $PROJECT apply -f -
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: stress-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: stress
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        minAllowed:
          cpu: 100m
          memory: 50Mi
        maxAllowed:
          cpu: 1000m
          memory: 1024Mi
        controlledResources: ["cpu", "memory"]
EOF
```

So since the only truly important things is the requests parameter, the Vertical Pod Autoscaler will always work with this. Whenever you define vertical autoscaling for your app, you are defining what the requests should be.

* After a couple of minutes, check the VPA to see the Memory and CPU suggested:

```sh
kubectl get vpa
NAME         MODE   CPU   MEM       PROVIDED   AGE
stress-vpa   Auto   1     262144k   True       81s
```

```sh
kubectl get vpa stress-vpa -o jsonpath='{.status}' | jq -r .
{
  "conditions": [
    {
      "lastTransitionTime": "2021-12-20T20:48:09Z",
      "status": "True",
      "type": "RecommendationProvided"
    }
  ],
  "recommendation": {
    "containerRecommendations": [
      {
        "containerName": "stress",
        "lowerBound": {
          "cpu": "746m",
          "memory": "262144k"
        },
        "target": {
          "cpu": "1",
          "memory": "262144k"
        },
        "uncappedTarget": {
          "cpu": "1388m",
          "memory": "262144k"
        },
        "upperBound": {
          "cpu": "1",
          "memory": "1Gi"
        }
      }
    ]
  }
}
```

* **Lower Bound**: when your pod goes below this usage, it will be evicted and downscaled.

* **Target**: this will be the actual amount configured at the next execution of the admission webhook. (If it already has this config, no changes will happen (your pod won’t be in a restart/evict loop). Otherwise, the pod will be evicted and restarted using this target setting.)

* **Uncapped Target**: what would be the resource request configured on your pod if you didn’t configure upper limits in the VPA definition.

* **Upper Bound**: when your pod goes above this usage, it will be evicted and upscaled.

* Let's increase the memory allocation by the stress process in our container in our stress pod, above the defined limit:

```sh
kubectl get pod -l app=stress -n $PROJECT -o yaml | grep limits -A1
        limits:
          memory: 200Mi
```

the memory limit is 200Mi as is defined in the Deployment.

* Increase the memory allocation in the pod patching the arg of the --vm-bytes to 250M:

```sh
oc patch deployment stress --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args/3", "value": "250M" }]'
```

* Check the pods to see if the OOMKilled or Crashloopbackoff state it's in our stress pod:

```sh
kubectl get pod -w
NAME                      READY   STATUS        RESTARTS   AGE
stress-7b9459559c-ntnrv   1/1     Running       0          5s
stress-7d48fdb6fb-j46b8   1/1     Terminating   0          22m
```

* Check the VPA resources and:

```sh
kubectl get pod -l app=stress -o yaml | grep vpa
      vpaObservedContainers: stress
      vpaUpdates: 'Pod resources updated by stress-vpa: container 0: cpu request,
```

* Check that the VPA changed automatically the requests and limits in the POD, but NOT in the deployment or ReplicaSet:

```sh
kubectl get pod -l app=stress -o yaml | grep requests -A2
        requests:
          cpu: "1"
          memory: 262144k
```

```sh
kubectl get pod -l app=stress -o yaml | grep limits -A1
        limits:
          memory: 500Mi
```

So what happens to the limits parameter of your pod? Of course they will be also adapted, when you touch the requests line. The VPA will proportionally scale limits.

As mentioned above, this is proportional scaling: in our default stress deployment manifest, we have the following requests to limits ratio:

* CPU: 100m -> 200m: 1:4 ratio
* memory: 100Mi -> 250Mi: 1:2.5 ratio

So when you get a scaling recommendation, it will respect and keep the same ratio you originally configured, and proportionally set the new values based on your original ratio.

* The deployment of stress app is not changed at all, the VPA just is changing the Pod spec definition:

```sh
kubectl get deployment stress -o yaml | egrep -i 'limits|request' -A1         
         requests:
            memory: "100Mi"
          limits:
            memory: "200Mi"
```

But don’t forget, your limits are almost irrelevant, as the scheduling decision (and therefore, resource contention) will be always done based on the requests.

Limits are only useful when there's resource contention or when you want to avoid uncontrollable memory leaks.

If you want to see this demo in action and the explanation of the VPA modes among other things, check our [DevConf.CZ 2022 talk about VPA](https://www.youtube.com/watch?v=znnHnERjnGs&t=251s&ab_channel=DevConf)

And with that ends this blog post around Vertical Pod Autoscaler in Kubernetes / OpenShift.

Happy autoscaling!

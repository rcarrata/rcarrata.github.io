---
layout: single
title: Deep Dive of Sandbox Containers / Kata Containers in OpenShift
date: 2021-11-10
type: post
published: true
status: publish
categories:
- OpenShift
- Observability
- Networking
- Kubernetes
tags: []
author: rcarrata
comments: true
---

How we can provide more isolation to our containers running on OpenShift? What are Kata Containers and how we can implement them in OpenShift? What are the differences between the regular containers and the kata containers workloads?

Let's Kata in!

## 1. Overview

OpenShift sandboxed containers support for OpenShift Container Platform provides users with built-in support for running Kata Containers as an additional optional runtime. This is particularly useful for users who are wanting to perform the following tasks:

* Run privileged or untrusted workloads.
* Ensure kernel isolation for each workload.
* Share the same workload across tenants.
* Ensure proper isolation and sandboxing for testing software.
* Ensure default resource containment through VM boundaries.

OpenShift sandboxed containers also provides users the ability to choose from the type of workload that they want to run to cover a wide variety of use cases.

You can use the OpenShift sandboxed containers Operator to perform tasks such as installation and removal, updates, and status monitoring.

[![](/images/sandbox.png "sandbox 0")]({{site.url}}/images/sandbox.png)

NOTE: Sandboxed containers are only supported on bare metal.

This Blog Post it's tested in 4.9.5 using the Sandbox Containers Preview 1.1.

NOTE: this blog post is supported by the [Sandbox Container repository](https://github.com/rcarrata/sandbox-containers) located in my personal Github.

## 2. Install Sandbox Containers Operator in OpenShift

We will start this blog post from an Openshift "empty" (fresh installed) because we will install the OpenShift Sandbox Container, version Preview 1.1.

NOTE: Please be aware that this version is currently in Technology Preview. OpenShift sandboxed containers is not intended for production use. Be careful!

Let's install and config the Sandbox Container operator!

* Clone the repository with the OCP objects to the Sandbox Containers:  

```sh
git clone https://github.com/rcarrata/sandbox-containers.git
cd sandbox-containers
```

* Install the subscription for the Sandbox Containers Preview 1.1:

```sh
oc apply -k operator/overlays/preview-1.1/
```

* Check the sandbox containers operator subscription in the sandbox namespace:

```sh
oc get subs -n openshift-sandboxed-containers-operator
NAME                            PACKAGE                         SOURCE             CHANNEL
sandboxed-containers-operator   sandboxed-containers-operator   redhat-operators   preview-1.1
```

* Check the ClusterServiceVersion of the Sandbox container Operator:

```sh
oc get csv -n openshift-sandboxed-containers-operator
NAME                                   DISPLAY                                   VERSION   REPLACES                               PHASE
sandboxed-containers-operator.v1.1.0   OpenShift sandboxed containers Operator   1.1.0     sandboxed-containers-operator.v1.0.2   Succeeded
```

* Check that the Operator is up and running:

```sh
oc get deployments -n openshift-sandboxed-containers-operatoror
NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
sandboxed-containers-operator-controller-manager   1/1     1            1           20s
```

NOTE: Notice that it's in the Phase Succeeded.

## 3. Creating and configuring the KataConfig in OpenShift cluster

You must create one KataConfig custom resource (CR) to trigger the OpenShift sandboxed containers Operator to do the following:

1. Install the needed RHCOS extensions, such as QEMU and kata-containers, on your RHCOS node.
2. Ensure that the runtime, CRI-O, is configured with the correct Kata runtime handlers.
3. Create a RuntimeClass custom resource with necessary configurations for additional overhead caused by virtualization and the required additional processes.

* You can selectively install the Kata runtime on specific workers. Let's select one:

```
NODE_KATA=$(oc get node -l node-role.kubernetes.io/worker= --no-headers=true | awk '{ print $1 }' | head -n1)

echo $NODE_KATA
ocp-8vr6j-worker-0-82t6f
```

* Label the node that will be where the KataConfig will apply:

```
oc label node $NODE_KATA kata=true
```

* In the KataConfig we can see the matchLabel with the exact label defined in the step before:

```sh
apiVersion: kataconfiguration.openshift.io/v1
kind: KataConfig
metadata:
  name: cluster-kataconfig
 spec:
    kataConfigPoolSelector:
      matchLabels:
         kata: 'true'
```

Note: There is a typo in the doc [1] since it uses true as label value instead of ‘true’

* Install the KataConfig using kustomize:

```sh
oc apply -k instance/overlays/default/
```

* You can monitor the values of the KataConfig custom resource by running:

```sh
oc describe kataconfig cluster-kataconfig
Name:         cluster-kataconfig
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  kataconfiguration.openshift.io/v1
Kind:         KataConfig
...
Spec:
  Kata Config Pool Selector:
    Match Labels:
      Kata:  true
Status:
  Installation Status:
    Is In Progress:  True
    Completed:
    Failed:
    Inprogress:
  Prev Mcp Generation:  2
  Runtime Class:        kata
  Total Nodes Count:    1
  Un Installation Status:
    Completed:
    Failed:
    In Progress:
      Status:
  Upgrade Status:
Events:  <none>
```

as you can notice the KataConfig installation it's in progress in One Total Node Count and using the Runtime Class kata

You can check to see if the nodes in the machine-config-pool object are going through a config update.

```
oc get mcp kata-oc
NAME      CONFIG                                              UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
kata-oc   rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8   True      False      False      1              1                   1                     0                      3m
```

If we check the machine config pool we can check that the kata-oc mcp it's based in several machine configs, and in specific there is one that defines the sandbox-containers extensions:

```
oc get mcp kata-oc -o yaml | egrep -i 'kata|sandbox'
  name: kata-oc
    name: rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
      name: 50-enable-sandboxed-containers-extension
      - kata-oc
      kata: "true"
    message: All nodes are updated with rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
    name: rendered-kata-oc-7ce66b5e9e1c51753ebf99c1d9603bd8
      name: 50-enable-sandboxed-containers-extension
```

You can check the machine config that uses the mcp of kata-oc:

```sh
oc get mc | grep sand
50-enable-sandboxed-containers-extension                                                      3.2.0
4m
```

Let's check in detail this MachineConfig described in the step before:

```sh
oc get mc $(oc get mc | awk '{ print $1 }' | grep sand) -o yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
...
  labels:
    app: cluster-kataconfig
    machineconfiguration.openshift.io/role: worker
  name: 50-enable-sandboxed-containers-extension
...
spec:
  config:
    ignition:
      config:
        replace:
          verification: {}
      proxy: {}
      security:
        tls: {}
      timeouts: {}
      version: 3.2.0
    passwd: {}
    storage: {}
    systemd: {}
  extensions:
  - sandboxed-containers
  fips: false
  kernelArguments: null
  kernelType: ""
  osImageURL: ""
```

as you can noticed the extensions have the sandbox-containers. The [RHCOS extensions](https://github.com/openshift/machine-config-operator/blob/master/docs/MachineConfiguration.md#rhcos-extensions) users can enable a limited set of additional functionality on the RHCOS nodes. In 4.8+ only [usbguard and sandboxed-containers](https://docs.openshift.com/container-platform/4.9/post_installation_configuration/machine-configuration-tasks.html#rhcos-add-extensions_post-install-machine-configuration-tasks) are supported extensions

Automatically the Machine Config Operator will apply the MachineConfigPool rendered containing the different MachineConfigs, including the last MC of sandboxed-containers-extension described in the previous step.

```
oc get nodes -l kubernetes.io/os=linux,node-role.kubernetes.io/worker=
NAME                       STATUS                     ROLES    AGE   VERSION
ocp-8vr6j-worker-0-82t6f   Ready,SchedulingDisabled   worker   26h   v1.22.0-rc.0+a44d0f0
ocp-8vr6j-worker-0-kvxr9   Ready                      worker   26h   v1.22.0-rc.0+a44d0f0
ocp-8vr6j-worker-0-sl79n   Ready                      worker   26h   v1.22.0-rc.0+a44d0f0
```

As you can check the first selected worker labeled with the kata: 'true' is automatically installing and configuring the KataConfig extension.

We can check when the Machine Config Operator finishes their work, when the Updating, and Degraded sections of the MCP are in False state:

```sh
# oc get mcp kata-oc
NAME      CONFIG                                              UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
kata-oc   rendered-kata-oc-999617e684ef816391be460254498d18   True      False      False      1              1                   1                     0                      8m33s
```

## 4. Runtimes configurations

Now, let's check the configuration of the CRIO, if it's configured correctly for the Kata runtime handlers.

Let's jump to the worker node labeled with the Kata containers:

```sh
# oc debug node/$NODE_KATA
Starting pod/ocp-8vr6j-worker-0-82t6f-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.126.53
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host bash
[root@ocp-8vr6j-worker-0-82t6f /]#
```

If we check the crio.conf.d folder, we can see the 50-kata file containing the specifics of the Kata runtimes for CRIO:

```
[root@ocp-8vr6j-worker-0-82t6f /]# cat /etc/crio/crio.conf.d/50-kata
[crio.runtime.runtimes.kata]
  runtime_path = "/usr/bin/containerd-shim-kata-v2"
  runtime_type = "vm"
  runtime_root = "/run/vc"
  privileged_without_host_devices = true
```

## 5. Analysis of the RuntimeClass / Kata Runtime

RuntimeClass defines a class of container runtime supported in the cluster. The RuntimeClass is used to determine which container runtime is used to run all containers in a pod.

RuntimeClasses are manually defined by a user or cluster provisioner, and referenced in the PodSpec. In our case we will have the RuntimeClass "kata", because we will demoing the Sandbox Containers use in our OpenShift clusters.

After a while, the Kata runtime is now installed on the cluster and ready for use as a secondary runtime. 

Let's verify that we have a newly created RuntimeClass for Kata on our cluster.

```sh
oc get runtimeClass kata -o yaml
apiVersion: node.k8s.io/v1
handler: kata
kind: RuntimeClass
metadata:
  name: kata
...
overhead:
  podFixed:
    cpu: 250m
    memory: 350Mi
scheduling:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

## 6. Testing Sandbox Containers vs Regular Containers

Let's deploy two identical pods, one using the regular RuntimeClass and the other using the RuntimeClass "kata" to check the differences between them.

Both pods will use the exact same image (net-tools) pulled from Quay.io.


## 6.1 Deploying a regular workload

First, let's generate a new project:

```sh
oc new-project test-kata
```

Let's deploy a simple example using the net-tools image and a sleep command:

```sh
oc apply -f examples/example-net-regular.yaml
```

```sh
apiVersion: v1
kind: Pod
metadata:
  name: example-net-tools-regular
spec:
  nodeSelector:
    kata: 'true'
  containers:
    - name: example-net-tools-regular
      image: quay.io/rcarrata/net-toolbox:latest
      command: ["/bin/bash", "-c", "sleep 99999999999"]
```

if you noticed, the pod have a nodeSelector with the label for ensure that the workloads are running in the labeled kata container worker.

```sh
oc get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE                       NOMINATED NODE   READINESS GATES
example-net-tools-regular   1/1     Running   0          55s   10.128.2.41   ocp-8vr6j-worker-0-82t6f   <none>           <none>
```

## 6.2 Creating a Kata Container workload

* Let's deploy the same pod but with the runtimeClass defined as 'kata':

```sh
oc apply -f examples/example-net-kata.yaml
```

```sh
# cat examples/example-net-kata.yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-net-tools-kata
spec:
  nodeSelector:
    kata: 'true'
  containers:
    - name: example-net-tools-kata
      image: quay.io/rcarrata/net-toolbox:latest
      command: ["/bin/bash", "-c", "sleep 99999999999"]
```

Let's confirm that the pods are running with the runtimeClassName as Kata:

```sh
oc get pod example-net-tools-kata -o jsonpath='{.spec.runtimeClassName}'
kata

oc get pod example-net-tools-kata -o jsonpath='{.spec.runtimeClassName}' -o yaml | grep runtimeClass | tail -1
  runtimeClassName: kata
```

## 7. Analysis of the Kata Containers Workloads and Qemu VMs

As all we know, workloads in Kubernetes/OpenShift runs within a Pod, the minimal representation of a workload in Kubernetes/OpenShift.

In case of the Kata Containers / Sandbox Containers, pods are bound to a QEMU Virtual Machine (qemu-vm), which provides this extra layer of isolation and furthermore extra layer of security.

Each VM runs in a qemu process and hosts a kata-agent process that acts as a supervisor for managing container workloads and processes that are running in those containers.

Let's see how this works comparing the regular pod and the pod using kata container RuntimeClass.

* First let's compare the uptime between the regular pod and the kata container pod:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/uptime
1082.84 1077.77
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/uptime
3074.51 11793.48
```

For what reason is there a difference so significant? The uptime of the standard container kernel is the same as the uptime of the node that is running the container, while the uptime of the sandboxed kata container pod is of its sandbox (VM) that was created at the pod creation time.

* On the other hand, let's compare the kernel command lines from the /proc/cmdline:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cmdline
tsc=reliable no_timer_check rcupdate.rcu_expedited=1 i8042.direct=1 i8042.dumbkbd=1 i8042.nopnp=1 i8042.noaux=1 noreplace-smp reboot=k console=hvc0 console=hvc1 cryptomgr.notests net.ifnames=0 pci=lastbus=0 debug panic=1 nr_cpus=4 scsi_mod.scan=none agent.log=debug
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/cmdline
BOOT_IMAGE=(hd0,gpt3)/ostree/rhcos-9d5687c13259b146f9e918ae7ee8e03f697789ddb69e37058072764a32d96fc4/vmlinuz-4.18.0-305.19.1.el8_4.x86_64 random.trust_cpu=on console=tty0 console=ttyS0,115200n8 ignition.platform.id=qemu ostree=/ostree/boot.0/rhcos/9d5687c13259b146f9e918ae7ee8e03f697789ddb69e37058072764a32d96fc4/0 root=UUID=7a2353f2-a105-4dfa-b800-49030ae7725e rw rootflags=prjquota
```

As we can see in the output of both cmdline commands, the containers are using two different kernel instances running in different environments (a worker node and a lightweight VM).

* On the third place, we will check the number of cores on both containers:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cpuinfo | grep processor | wc -l
1
```

B. Regular Container

```sh
oc exec -ti example-net-tools-regular -- cat /proc/cpuinfo | grep processor | wc -l
4
```

By default, Sandbox Containers are configured to run with one vCPU per VM, and for this reason we're seeing the difference between the container running in Sandbox Containers and the regular containers which shows the total amount of vCPUs available in the worker node.

* Finally let's check the kernel used in each container:

A. Sandboxed Workloads

```sh
oc exec -ti example-net-tools-kata -- uname -r
4.18.0-305.19.1.el8_4.x86_64
```

B. Regular Container

```sh
oc exec -ti example-net-tools-kata -- cat /proc/cpuinfo | grep processor | wc -l
1
```

But they are the same kernel version! But we checked that are different kernel instances in the cmdline before, what's happening?

Despite of the mentioned differences, Kata Containers used by the OpenShift Sandbox Containers it's always running with the very same kernel version on the VM as the underlying RHCOS is running with (OS). The VM image is generated at the startup of the host, ensuring that is compatible with the kernel currently used by the RHCOS host.

## 8. Analysis of the QEMU processes of the Sandbox Containers

Sandbox Containers are containers that are sandboxed by a VM running a QEMU process.

If we check the node where the workload is running, we can verify that this QEMU process is also running there:

* Let's check which node our example kata pod has been assigned to:  

```sh
pod_node_kata=$(oc get pod/example-net-tools-kata -o=jsonpath='{.spec.nodeName}')

echo $pod_node_kata
ocp-8vr6j-worker-0-82t6f
```

as we expected is the same as we defined using the label kata: 'true'.

* Let's extract the containerID CRI:

```sh
oc get pod/example-net-tools-kata -o=jsonpath='{.status.containerStatuses[*].containerID}' | cut -d "/" -f3
fcc17d03b182e0ac3db33f3a19668a7b59d1ef32934e0af0db01a6a482c04056
```

* Let's jump into the node where the kata container is running using the oc debug node:

```sh
oc debug node/$pod_node_kata
chroot /host bash
[root@worker-0 /]#
```

* Check the qemu processes that are running in the node:

```sh
ps aux | grep qemu
sandbox-712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3 -uuid 2428acef-210f-42c5-9def-7c407c5c4042 -machine q35,accel=kvm,kernel_irqchip -cpu host,pmu=off
...
```

as we can see, there is one qemu process running. It is expected to see a qemu process running for each pod running sandbox containers / kata containers on that host/node.

* Let's check the sandboxID with the crictl inspect. We will use the containerID from the earlier step:

```sh
sandboxID=$(crictl inspect fcc17d03b182e0ac3db33f3a19668a7b59d1ef32934e0af0db01a6a482c04056 | jq -r '.info.sandboxID')

echo $sandboxID
712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3
```

* Check the QEMU process again filtering by the sandboxID:

```sh
[root@worker-0 /]# ps aux | grep qemu | grep $sandboxID
root       12325  0.6  4.0 2463548 332908 ?      Sl   19:10   0:15 /usr/libexec/qemu-kiwi -name
sandbox-712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3

[root@worker-0 /]# echo $?
0
```

The QEMU process is indeed running the container we inspected, because the CRI sandboxID is associated with your containerID. The Ids from the crictl inspect that outputs the sandboxID and the QEMU process running the workload are the same.

Finally we have two additional processes that add more overhead:

```
[root@worker-0 /]# ps -ef | grep kata | egrep -i 'containerd|virtio'
root       33678       1  0 Nov09 ?        00:00:10 /usr/bin/containerd-shim-kata-v2 -namespace default -address  -publish-binary /usr/bin/crio -id 712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3
root       33707   33678  0 Nov09 ?        00:00:00 /usr/libexec/virtiofsd --fd=3 -o source=/run/kata-containers/shared/sandboxes/712a4cb4a28dff8655bd92fd6bd5e761173a2b2c23b8b7615a5bbb12ca1b75a3/shared -o cache=auto --syslog -o no_posix_lock -d --thread-pool-size=1
```

1. containerd-shim-kata-v2 is used to communicate with the pod.
2. virtiofsd handles host file system access on behalf of the guest.

And with that we reviewed how to install, configure and use Sandbox Containers in OpenShift.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy Sandboxing!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="You like this blog? It helped? Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
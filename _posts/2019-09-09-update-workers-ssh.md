---
layout: single
title: Update SSH Keys into the nodes of OpenShift4 cluster
date: 2019-09-09
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Administration", "OpenShift", "AWS", "Cloud"]
author: rcarrata
comments: true
---

How to update the ssh keys into your OpenShift4 clusters once is deployed and up&running? And how to
automate this easy and straightforward?

## Overview

By default, the OpenShift 4 installer creates a single user named core with optional SSH keys specified at install time.

This controller supports updating the SSH keys of user core via a MachineConfig object. The SSH keys are updated for all members of the MachineConfig pool specified in the MachineConfig, for example: all worker nodes.

### Components description involved

You will need the following information for the MachineConfig that will be used to update your SSHKeys.

* **machineconfiguration.openshift.io/role**: the MachineConfig that is updated will be applied to all nodes with the role specified here. For example: master or worker

* **sshAuthorizedKeys**: you will need one or more public keys to be assigned to user core. Multiple SSH Keys should begin on different lines and each be preceded by -.

### Updating SSH for workers

* List all the machineconfigs currently in the cluster.

```
oc get machineconfigs
NAME                                                        GENERATEDBYCONTROLLER      IGNITIONVERSION   CREATED
00-master                                                   4.1.0-201905070232-dirty   2.2.0             7d17h
00-worker                                                   4.1.0-201905070232-dirty   2.2.0             7d17h
01-master-container-runtime                                 4.1.0-201905070232-dirty   2.2.0             7d17h
01-master-kubelet                                           4.1.0-201905070232-dirty   2.2.0             7d17h
01-worker-container-runtime                                 4.1.0-201905070232-dirty   2.2.0             7d17h
01-worker-kubelet                                           4.1.0-201905070232-dirty   2.2.0             7d17h
99-master-5b73c25f-7271-11e9-a120-02f119962354-registries   4.1.0-201905070232-dirty   2.2.0             7d17h
99-master-ssh                                                                          2.2.0             7d17h
99-worker-5b782f30-7271-11e9-a120-02f119962354-registries   4.1.0-201905070232-dirty   2.2.0             7d17h
99-worker-ssh                                                                          2.2.0             7d17h
rendered-master-3443b14f13eb6d4f489ae0901a898e80            4.1.0-201905070232-dirty   2.2.0             7d17h
rendered-worker-820426a0516a3ce9aaefeedb44c15990            4.1.0-201905070232-dirty   2.2.0             7d17h
```

* If we take a look about the ssh machineconfig for the workers, named as 99-worker-ssh:

```
oc get machineconfigs 99-worker-ssh -o yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  creationTimestamp: 2019-05-09T15:44:40Z
  generation: 1
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-ssh
  resourceVersion: "2531"
  selfLink: /apis/machineconfiguration.openshift.io/v1/machineconfigs/99-worker-ssh
  uid: 5ba4706c-7271-11e9-a120-02f119962354
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd:
      users:
      - name: core
        sshAuthorizedKeys:
        - |
          ssh-rsa xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    storage: {}
    systemd: {}
  osImageURL: ""
```

This specific machineconfig are where are stored the ssh keys allowed to access to the worker nodes (with the user core). So we need to update them in order to give access to a new ssh-key.

* Export the 99-worker-ssh to edit the SSHAuthorizedKeys.

```
oc get machineconfigs 99-worker-ssh -oyaml --export > update-ssh-worker.yaml
```

* Update the sshAuthorizedKeys for core user in update-ssh-worker.yaml.

```
# update-ssh-worker.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-ssh
spec:
  config:
    passwd:
      users:
      - name: core
        sshAuthorizedKeys:
        - ssh rsa ABC123....
        - ssh rsa {{ NEW_KEY }}
```

* Now with your MachineConfig yaml file (using the example above), apply the changes.

```
oc apply -f update-ssh-worker.yaml
```

* An alternative way to update the ssh-key in the machineconfig is with the command of oc patch:

```
$ oc patch machineconfig 99-worker-ssh --type=json --patch="[{\"op\":\"add\", \"path\":\"/spec/config/passwd/users/0/sshAuthorizedKeys/-\", \"value\":\"$(cat $HOME/.ssh/node.id_rsa.pub)\"}]"
```

A short time after updating the worker configuration the machine config daemon will begin restarting the nodes in the cluster as part of the reconfiguration. It may take 15 minutes or more to cycle through the nodes. Your nodes will show SchedulingDisabled, NotReady and finally Ready. One after the other.


### Deep dive in the node status and the logs of the ssh-key updates

After that the worker ssh is applied, you can check the status with several ways. Let's dig on a bit.

* Check that the 99-worker-ssh MachineConfig is updated:

```
oc get machineconfigs
NAME                                                        GENERATEDBYCONTROLLER      IGNITIONVERSION   CREATED
00-master                                                   4.1.0-201905070232-dirty   2.2.0             7d17h
00-worker                                                   4.1.0-201905070232-dirty   2.2.0             7d17h
01-master-container-runtime                                 4.1.0-201905070232-dirty   2.2.0             7d17h
01-master-kubelet                                           4.1.0-201905070232-dirty   2.2.0             7d17h
01-worker-container-runtime                                 4.1.0-201905070232-dirty   2.2.0             7d17h
01-worker-kubelet                                           4.1.0-201905070232-dirty   2.2.0             7d17h
99-master-5b73c25f-7271-11e9-a120-02f119962354-registries   4.1.0-201905070232-dirty   2.2.0             7d17h
99-master-ssh                                                                          2.2.0             7d17h
99-worker-5b782f30-7271-11e9-a120-02f119962354-registries   4.1.0-201905070232-dirty   2.2.0             7d17h
99-worker-ssh                                                                          2.2.0             15s
rendered-master-3443b14f13eb6d4f489ae0901a898e80            4.1.0-201905070232-dirty   2.2.0             7d17h
rendered-worker-820426a0516a3ce9aaefeedb44c15990            4.1.0-201905070232-dirty   2.2.0             7d17h
```

* Check the nodes, and expect reboot from each of the worker nodes:

```
[rcarrata@asimov Code]$ oc get nodes
NAME                                            STATUS                     ROLES     AGE       VERSION
ip-10-0-130-3.eu-central-1.compute.internal     Ready                      master    7d17h     v1.13.4+43acbc5e5
ip-10-0-143-22.eu-central-1.compute.internal    Ready                      worker    7d17h     v1.13.4+43acbc5e5
ip-10-0-149-110.eu-central-1.compute.internal   Ready                      master    7d17h     v1.13.4+43acbc5e5
ip-10-0-156-23.eu-central-1.compute.internal    Ready,SchedulingDisabled   worker    7d17h     v1.13.4+43acbc5e5
ip-10-0-165-73.eu-central-1.compute.internal    Ready                      master    7d17h     v1.13.4+43acbc5e5
ip-10-0-167-20.eu-central-1.compute.internal    Ready                      worker    7d17h     v1.13.4+43acbc5e5
```

NOTE: the OpenShift Machine Config operator, will handle the reboots and the apply of the updated MachineConfigs.

* On the other hand you can check the Machine Config Daemon logs, that are in charge of rebooting the nodes in order.

```
$ oc logs -f -n openshift-machine-config-operator machine-config-daemon-<same-hash>
```

* Check one of your nodes accessing it with the oc exec command:

```
$ NODE_SSH_POD=$(oc get pod -l app=node-ssh -o jsonpath='{.items[0].metadata.name}')
$ oc exec -it $NODE_SSH_POD -- ssh core@10.0.159.205
```

### Check the ssh access to the worker nodes with your new ssh-key

After connecting to the node, check uptime. Observe that the node has rebooted recently.

* Finally check that we can enter with the new ssh key:

```
[ec2-user@ip-xxxxx ~]$ ssh -i ~/.ssh/my_new_key.pem core@ip-xxxx.eu-central-1.compute.internal
Red Hat Enterprise Linux CoreOS 410.8.20190506.3 Beta
WARNING: Direct SSH access to machines is not recommended.
This node has been annotated with machineconfiguration.openshift.io/ssh=accessed

---
Last login: Fri May 17 12:35:04 2019 from 10.0.2.146
[systemd]
Failed Units: 1
  multipathd.socket
```

## Unsupported Operations

There is some operations that nowadays are not supported by the Machine Config Daemon (the operator that handles part of the ssh-key updates.

The most important are:

* The MCD will not add any new users.
* The MCD will not delete the user core.
* The MCD will not make any changes to any other User fields for user core other than sshAuthorizedKeys.

And that's it! Hope that helps!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
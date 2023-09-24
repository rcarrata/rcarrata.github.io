---
layout: single
title: Collect Sosreport in OpenShift 4
date: 2019-08-08
type: post
published: true
status: publish
categories:
- OpenShift
tags: ["Kubernetes", "security", "Administration", "OpenShift"]
author: rcarrata
comments: true
---

This blog post aims to provide a guide for collect a Sosreport in a node of an OpenShift 4 Cluster.

Because OpenShift 4 uses RHCOS as Operating system for their nodes, the procedure for collect an sosreport for further analysis changed a bit between OpenShift3/Rhel7.x. This procedure is also valid for OpenShift Worker nodes based in RHEL8, instead of RHCOS.

## Access to the OpenShift node

By default, the OpenShift 4 installer creates a single user named core with optional SSH keys specified at install time.

In our case, the ssh-key generated and injected into the cluster at install time could be used, but another ssh-keys can be updated into the OCP nodes following the procedure of [Updating SSH Keys with the MCD](https://github.com/openshift/machine-config-operator/blob/master/docs/Update-SSHKeys.md).

* SSH to the OpenShift cluster specific node

```
$ ssh -i ~/.ssh/bastion ec2-user@bastion.dev.ocp4.example.com

$ ssh -i ~/.ssh/ocp4.pem core@ip-10-148-204-82.eu-central-1.compute.internal
Red Hat Enterprise Linux CoreOS 410.8.20190718.1
WARNING: Direct SSH access to machines is not recommended.

$ sudo -i
```

## Obtain the support-tools container in the node

* Using podman, login with your RH credentials into the node:

```
# podman login registry.redhat.io
Authenticating with existing credentials...
Existing credentials are invalid, please enter valid username and password
Username: rcarrata@redhat.com
Password:
Login Succeeded!
```

For perform the sosreport into the node, a specific container could be used: [support-tools](https://access.redhat.com/containers/?tab=overview&get-method=red-hat-login#/registry.access.redhat.com/rhel8/support-tools).
The Red Hat Enterprise Linux Support Tools image contains tools to analyze the host system including sosreport, strace, and tcpdump.

* Pull the support-tools container to the specific node (in our case a Master node):

```
# podman pull registry.redhat.io/rhel8/support-tools
Trying to pull registry.redhat.io/rhel8/support-tools...Getting image source signatures
Copying blob e61d8721e62e: 0 B / 67.75 MiB [-----------------------------------]
Copying blob e61d8721e62e: 9.35 MiB / 67.75 MiB [====>-------------------------]
Copying blob e61d8721e62e: 67.21 MiB / 67.75 MiB [=============================]
Copying blob e61d8721e62e: 67.75 MiB / 67.75 MiB [==========================] 3s
Copying blob c585fd5093c6: 1.47 KiB / 1.47 KiB [============================] 3s
Copying blob 77392c39ffcb: 8.67 MiB / 8.67 MiB [============================] 3s
Copying config 23a6cff4874d: 4.36 KiB / 4.36 KiB [==========================] 0s
Writing manifest to image destination
Storing signatures
23a6cff4874d03f84c7a787557b693afd58a1fb1f1123d5c9d254f785771c8fa
```

* Execute the container with specific runlabel:

```
# podman container runlabel RUN registry.redhat.io/rhel8/support-tools
Command: /proc/self/exe run -it --name support-tools --privileged --ipc=host --net=host --pid=host -e HOST=/host -e NAME=support-tools -e IMAGE=registry.redhat.io/rhel8/support-tools:latest -v /run:/run -v /var/log:/var/log -v /etc/machine-id:/etc/machine-id -v /etc/localtime:/etc/localtime -v /:/host registry.redhat.io/rhel8/support-tools:latest
```

## Execute the sosreport for collect data

* Once into the support-tools container, perform the sosreport command to collect the data:

```
bash-4.4# sosreport

sosreport (version 3.6)

This command will collect diagnostic and configuration information from
this Red Hat Enterprise Linux system and installed applications.

An archive containing the collected information will be generated in
/host/var/tmp/sos.x55wf_r3 and may be provided to a Red Hat support
representative.

Any information provided to Red Hat will be treated in accordance with
the published support policies at:

  https://access.redhat.com/support/

The generated archive may contain data considered sensitive and its
content should be reviewed by the originating organization before being
passed to any third party.

No changes will be made to system configuration.

Press ENTER to continue, or CTRL-C to quit.

Please enter the case id that you are generating this report for []: xxxxxxxx

 Setting up archive ...
 Setting up plugins ...
 Running plugins. Please wait ...

  Starting 6/78  cgroups         [Running: auditd block boot cgroups]                     caught exception in plugin method "cgroups.collect()"
writing traceback to sos_logs/cgroups-plugin-errors.txt
  Starting 18/78 grub2           [Running: cgroups chrony crio dracut grub2]              caught exception in plugin method "grub2.collect()"
writing traceback to sos_logs/grub2-plugin-errors.txt
  Starting 69/78 system          [Running: cgroups chrony grub2 logs selinux system]      [plugin:system] _copy_dir: Too many levels of symbolic links copying '/host/proc/sys/fs/binfmt_misc'
  Starting 70/78 systemd         [Running: cgroups chrony grub2 logs selinux systemd]     caught exception in plugin method "systemd.collect()"
writing traceback to sos_logs/systemd-plugin-errors.txt
  Finishing plugins              [Running: cgroups grub2 systemd]                         kaged]
Creating compressed archive...

Your sosreport has been generated and saved in:
  /host/var/tmp/sosreport-ip-10-148-204-82-xxxxxxxx-2019-08-08-jnsdcyp.tar.xz

The checksum is: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Please send this file to your support representative.
```

* The sosreport file is located into the /host/var/tmp/ folder into the container:

```

bash-4.4# ls -lhrt /host/var/tmp/sosreport-2019-08-08-jnsdcyp.tar.xz
-rw-------. 1 root root 11M Aug  8 10:00 /host/var/tmp/sosreport-ip-10-148-204-82-02444698-2019-08-08-jnsdcyp.tar.xz
```

* And outside the container (and due to the scc that the container was executed) the sosreport is located into the /var/tmp/ folder of our RHCOS node:

```
# ll /var/tmp/sosreport-2019-08-08-jnsdcyp.tar.xz
-rw-------. 1 root root 11233928 Aug  8 10:00 /var/tmp/sosreport-2019-08-08-jnsdcyp.tar.xz
```

* Just scp/rsync them to any location and after that can be uploaded or analysed for obtain more information.

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
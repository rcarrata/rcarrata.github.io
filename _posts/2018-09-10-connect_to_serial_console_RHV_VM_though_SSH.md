---
layout: post
title: Connect to Serial Console RHV VM through SSH
date: 2018-10-06
type: post
published: true
status: publish
categories:
- Blog
- Personal
tags: []
author: rcarrata
comments: true
---

Imagine that you have one or several VMs deployed on top of your deployment of RHV, and you have not
access with ssh configured (because is a brand new VM without network config settings, or just
because you have some issues and you have not direct ssh connection).

In this case, you must to connect with a Serial console, through for example the Remote Viewer, to
configure or just for check if the VM is doing ok.

But sometimes, when you have a lot of latency between your source point of connection (like your
laptop) and your destination VM through the serial console port, you must to struggle to enter one
command.

To avoid this, you can connect through the serial console of any VM directly from the RHV Manager,
directly with your terminal (without using Remote Viewer or similar).

Connect to the port 2222 of rhevm and use the user "ovirt-vmconsole":

```
~ # ssh -p 2222 rhevm -l ovirt-vmconsole
Available Serial Consoles:
00 HostedEngine[10b4bd9c-d5e6-4663-b3e9-97a39f0b414c]
01 ceph0[700891dc-48d4-4951-981b-2298ad59ace9]
02 ceph1[774ff020-e1be-449b-982a-11c9de2a2d21]
03 ceph2[88a57a12-99cb-44ee-8998-6ad5b48dd21a]
...
08
cephpocnode1[aacf1c46-5a7d-44aa-b953-7cff00d390fb]‍‍‍‍‍‍‍‍
```

NOTE: in the connection through the 2222 port, use -l option instead of @, because its a serial
port.

After that you will be prompted with the whole list of VMs that are available in your cluster.

Select the VM that you want to connect:

```
SELECT> 8

Red Hat Enterprise Linux Server 7.5 (Maipo)
Kernel 3.10.0-862.3.2.el7.x86_64 on an x86_64

cephpocnode1 login: root
Password:
Last login: Tue Jul 31 09:17:21 on ttyS0
[root@cephpocnode1 ~]#‍‍‍‍‍‍‍‍‍
```

And that's it! You have a brand new serial console connection through ssh, without using other stuff
than your favourite terminal console (and avoiding the latency of course).

Enjoy!


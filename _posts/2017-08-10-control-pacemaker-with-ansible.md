---
layout: single
title: Control Pacemaker Resources with Ansible
date: 2017-08-10
type: post
published: true
status: publish
categories:
- Ansible
tags: []
author: rcarrata
comments: true
---

When an OpenStack installation is configured in High Availability across the several pieces it is divided into (haproxies, controllers, backends, etc.), there are a large number of resources that you must control to perform an upgrade or execute some operations.
 
For example, if you want to perform an upgrade of the controllers, you can stop them, perform the upgrade, and start them again one by one, without losing quorum or service.

This is a correct way to execute, but... why not automatize the process?
 
To control all the resources (the Pacemaker and systemd resources that OpenStack uses for proper execution), we developed several playbooks/tasks that can be used to start or stop all the resources (not only the clustered ones, but also the systemd resources of the cnodes, for example).
 
These playbooks can be found in my github: [Pcs Control](https://github.com/rcarrata/pcs_control)
 
For example, to execute the entire playbook that stops and starts the computes, controllers, cinder_controllers, and haproxies, run:

```
ansible-playbook -v -i hosts/<inventory> pcs_control.yml --ask-pass
``` 

To execute the playbook for only one specific group like computes or controllers, run:
 
```
ansible-playbook -v -i hosts/<inventory> -l compute pcs_control.yml --ask-pass
ansible-playbook -v -i hosts/<inventory> -l controller pcs_control.yml --ask-pass
ansible-playbook -v -i hosts/<inventory> -l haproxy pcs_control.yml --ask-pass
```

With this, you can automatically control the resources of OpenStack, making it much less painful to manage each service.
 
In future posts, I will show how this works with a full OpenStack cluster with high availability.

Hope that helps.
 
*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="rcarrata" data-color="#FFDD00" data-emoji=""  data-font="Cookie" data-text="Buy me a coffee :)" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
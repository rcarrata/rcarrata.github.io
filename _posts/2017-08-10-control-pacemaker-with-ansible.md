---
layout: post
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

When an Openstack is installed and configured in High Availability in the several pieces that is divided (haproxies, controllers, backends, etc) there is a large number of resources that you must to control for perform an upgrade or execute some operations.
 
For example, if you want to perform an upgrade of the controllers, you can stop the controllers, perform the upgrade and started again, one by one, without losing quorum and obviously service.

This is a correct way to execute, but... why not automatize the process?
 
For control all the resources (the pacemaker and systemd resources that uses Openstack for the proper execution), we developed several playbooks/tasks that we can be used, that allows you to start or stop all the resources (not only the clustered, also the systemd resources for example of the cnodes).
 
These playbooks can be found in my github: [Pcs Control](https://github.com/rcarrata/pcs_control)
 
For example, for execute the entire playbook that stops and starts the computes, controllers, cinder_controllers and haproxies execute:

```
ansible-playbook -v -i hosts/<inventory> pcs_control.yml --ask-pass
``` 

For only execute the playbook for control one specific group like computes or controllers, execute:
 
```
ansible-playbook -v -i hosts/<inventory> -l compute pcs_control.yml --ask-pass
ansible-playbook -v -i hosts/<inventory> -l controller pcs_control.yml --ask-pass
ansible-playbook -v -i hosts/<inventory> -l haproxy pcs_control.yml --ask-pass
```

So, with this you can automatically control the resources of Openstack, making more painless manage each service.
 
In next posts, I will show how works with a fully cluster of Openstack with high availaibility.

Hope that helps.
 
BR.
 
Rober 


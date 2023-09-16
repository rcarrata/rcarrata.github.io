---
layout: single
title: Share Vault between Ansible Core Roles
date: 2018-11-10
type: post
published: true
status: publish
categories:
- Ansible
- Security
- Automation
tags: []
author: rcarrata
comments: true
---


Sometimes you need to share the Ansible Vault between the different Ansible Roles, because you want to integrated within Tower or just in different Workflow Templates.

Also it can be useful if the playbook is executed by Ansible Tower, or just in the CLI without more options than the vault pass and the ID.

Create a role containing the vault

```
mkdir roles/vault1/defaults
```

Create the vault

```
ansible-vault create --new-vault-id=vault1 roles/vault1/defaults/main.yml
```

Modify the playbooks that execute the roles that need your vault:

```
vi playbook.yml
---
- name: "Testing Vault"
  hosts: localhost
  gather_facts: no
  roles:
    - vault1
    - other_role
```

And that's it!

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Enjoy!

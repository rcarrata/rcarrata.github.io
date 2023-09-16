---
layout: single
title: Integrate Ansible Tower with IDM
date: 2018-09-10
type: post
published: true
status: publish
categories:
- Ansible
- security
- Automation
tags: []
author: rcarrata
comments: true
---

This is a way to configure Ansible Tower 3.2.5 authentication with a idM.

* Background:

Ansible Tower version and host

```
[root@ansible-tower /]# rpm -qa | grep tower-server
ansible-tower-server-3.2.3-1.el7.x86_64
```

Ansible Tower hostname: tower.local.net

IdM (Red Hat Identity Management) version and host

```
[root@idm ~]# ipa --version

VERSION: 4.5.4, API_VERSION: 2.228

Domain name: local.net

IDM Host: idm.local.net

Admin User: admin
```

* IdM Configuration:

We are going to create a IdM user/group and configure LDAP Auth in Ansible Tower. I assumed that the idM server is installed and running, otherwise install it following this doc [1]

Create IdM user "tower" and group "tower-admins" and add the user to the group

```
[root@idm ~]# ipa user-add tower

[root@idm ~]# ipa group-add tower_users

[root@idm ~]# ipa group-add-member tower_users --users=tower
```

Once the user/group is created, we are going to look for the new user and check if it belongs to the tower admins group or not

```
[root@idm ~]# ldapsearch -p 389 -h idm.local.net -D "uid=admin,cn=users,cn=accounts,dc=local,dc=net" -w pass -b "dc=local,dc=net" -s sub "(objectclass=*)"
```

```
[root@idm ~]# ldapsearch -p 389 -h idm.local.net -D "uid=admin,cn=users,cn=accounts,dc=local,dc=net" -w pass -b "dc=local,dc=net" -s sub "uid=tower"

# extended LDIF
#
# LDAPv3
# base <dc=local,dc=net> with scope subtree
# filter: uid=tower
# requesting: ALL
#
# tower, users, compat, local.net
dn: uid=tower,cn=users,cn=compat,dc=local,dc=net
objectClass: posixAccount
objectClass: ipaOverrideTarget
objectClass: top
gecos: tower tower
cn: tower tower
uidNumber: 1413800006
gidNumber: 1413800006
loginShell: /bin/sh
homeDirectory: /home/tower
ipaAnchorUUID:: OklQQTpsb2NhbC5uZXQ6MzFhM2VlY2MtOGViMC0xMWU4LTk0NjItMDAxYTRhMT
 YwMTY3
uid: tower
# tower, users, accounts, local.net
dn: uid=tower,cn=users,cn=accounts,dc=local,dc=net
displayName: tower tower
uid: tower
krbCanonicalName: tower@LOCAL.NET
objectClass: top
objectClass: person
objectClass: organizationalperson
objectClass: inetorgperson
objectClass: inetuser
objectClass: posixaccount
objectClass: krbprincipalaux
objectClass: krbticketpolicyaux
objectClass: ipaobject
objectClass: ipasshuser
objectClass: ipaSshGroupOfPubKeys
objectClass: mepOriginEntry
loginShell: /bin/sh
initials: tt
gecos: tower tower
sn: tower
homeDirectory: /home/tower
mail: tower@local.net
krbPrincipalName: tower@LOCAL.NET
givenName: tower
cn: tower tower
ipaUniqueID: 31a3eecc-8eb0-11e8-9462-001a4a160167
uidNumber: 1413800006
gidNumber: 1413800006
mepManagedEntry: cn=tower,cn=groups,cn=accounts,dc=local,dc=net
memberOf: cn=ipausers,cn=groups,cn=accounts,dc=local,dc=net
memberOf: cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net
# search result
search: 2
result: 0 Success
# numResponses: 3
# numEntries: 2
```

```
[root@idm ~]# ldapsearch -p 389 -h idm.local.net -D "uid=admin,cn=users,cn=accounts,dc=local,dc=net" -w pass -b "dc=local,dc=net" -s sub "(&(objectClass=ipausergroup)(|(member=uid=tower,cn=users,cn=accounts,dc=local,dc=net)))"

# extended LDIF
#
# LDAPv3
# base <dc=local,dc=net> with scope subtree
# filter: (&(objectClass=ipausergroup)(|(member=uid=tower,cn=users,cn=accounts,dc=local,dc=net)))
# requesting: ALL
#
# ipausers, groups, accounts, local.net
dn: cn=ipausers,cn=groups,cn=accounts,dc=local,dc=net
objectClass: top
objectClass: groupofnames
objectClass: nestedgroup
objectClass: ipausergroup
objectClass: ipaobject
description: Default group for all users
cn: ipausers
ipaUniqueID: 6e2eaffc-8aa6-11e8-9114-001a4a160167
member: uid=jordi,cn=users,cn=accounts,dc=local,dc=net
member: uid=romario,cn=users,cn=accounts,dc=local,dc=net
member: uid=tower,cn=users,cn=accounts,dc=local,dc=net
# tower_users, groups, accounts, local.net
dn: cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net
objectClass: top
objectClass: groupofnames
objectClass: nestedgroup
objectClass: ipausergroup
objectClass: ipaobject
objectClass: posixgroup
cn: tower_users
ipaUniqueID: 5f4d36b2-8eb0-11e8-8e25-001a4a160167
gidNumber: 1413800007
member: uid=tower,cn=users,cn=accounts,dc=local,dc=net
# search result
search: 2
result: 0 Success
# numResponses: 3
# numEntries: 2
```

* Ansible Tower Configuration:

We are going to configure LDAP settings via tower-cli command as follows

```
[root@tower]# tower-cli setting modify AUTH_LDAP_SERVER_URI ldap://idm.local.net

[root@tower]# tower-cli setting modify AUTH_LDAP_BIND_DN uid=admin,cn=users,cn=accounts,dc=local,dc=net

[root@tower]# tower-cli setting modify AUTH_LDAP_BIND_PASSWORD password

[root@tower]# tower-cli setting modify AUTH_LDAP_START_TLS false

[root@tower]# tower-cli setting modify AUTH_LDAP_CONNECTION_OPTIONS "{u'OPT_NETWORK_TIMEOUT': 30, u'OPT_REFERRALS': 0}"

[root@tower]# tower-cli setting modify AUTH_LDAP_USER_SEARCH "[u'cn=users,cn=accounts,dc=local,dc=net', u'SCOPE_SUBTREE', u'(uid=%(user)s)']"

[root@tower]# tower-cli setting modify AUTH_LDAP_GROUP_SEARCH "[u'cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net', u'SCOPE_SUBTREE', u'(objectClass=ipausergroup)']"

[root@tower]# tower-cli setting modify AUTH_LDAP_GROUP_TYPE NestedMemberDNGroupType

[root@tower]# tower-cli setting modify AUTH_LDAP_REQUIRE_GROUP cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net

[root@tower]# tower-cli setting modify AUTH_LDAP_USER_ATTR_MAP "{u'first_name': u'givenName', u'last_name': u'sn', u'email': u'mail'}"
```

Once configured check it via tower-cli command line or WebUI

```
[root@ansible-tower]# tower-cli setting list -f human | grep AUTH_LDAP

AUTH_LDAP_SERVER_URI ldap://idm.local.net
AUTH_LDAP_BIND_DN uid=admin,cn=users,cn=accounts,dc=local,dc=net
AUTH_LDAP_BIND_PASSWORD $encrypted$
AUTH_LDAP_START_TLS false
AUTH_LDAP_CONNECTION_OPTIONS {u'OPT_NETWORK_TIMEOUT': 30, u'OPT_REFERRALS': 0}
AUTH_LDAP_USER_SEARCH [u'cn=users,cn=accounts,dc=local,dc=net', u'SCOPE_SUBTREE', u'(uid=%(user)s)']
AUTH_LDAP_USER_DN_TEMPLATE None
AUTH_LDAP_USER_ATTR_MAP {u'first_name': u'givenName', u'last_name': u'sn', u'email': u'mail'}
AUTH_LDAP_GROUP_SEARCH [u'cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net', u'SCOPE_SUBTREE', u'(objectClass=ipausergroup)']
AUTH_LDAP_GROUP_TYPE NestedMemberDNGroupType
AUTH_LDAP_REQUIRE_GROUP cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net
AUTH_LDAP_DENY_GROUP None
AUTH_LDAP_USER_FLAGS_BY_GROUP {}
AUTH_LDAP_ORGANIZATION_MAP {}
AUTH_LDAP_TEAM_MAP {}

[root@ansible-tower]# ansible-tower-service restart
```


And after that, we chech it in Ansible Tower Web UI > settings > Configure Tower > SUB CATEGORY > LDAP

Configure LOG DEBUG LEVEL in /etc/tower/conf.d/ldap.py config file

```
[root@ansible-tower]#  less /etc/tower/conf.d/ldap.py

LOGGING['handlers']['tower_warnings']['level'] = 'DEBUG'
```

And finally try to login with "tower" user via WebUI and at the same time check /var/log/tower/tower.log log file, as an example:

```
# tailf /var/log/tower/tower.log

2018-07-24 04:49:39,027 DEBUG django_auth_ldap search_s('cn=users,cn=accounts,dc=local,dc=net', 2, '(uid=%(user)s)') returned 1 objects: uid=tower,cn=users,cn=accounts,dc=local,dc=net
2018-07-24 04:49:39,201 DEBUG django_auth_ldap search_s('cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net', 2, '(&(objectClass=ipausergroup)(|(member=uid=jordi,cn=users,cn=accounts,dc=local,dc=net)))') returned 1 objects: cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net
2018-07-24 04:49:39,258 DEBUG django_auth_ldap search_s('cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net', 2, '(&(objectClass=ipausergroup)(|(member=cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net)))') returned 0 objects:
2018-07-24 04:49:39,260 DEBUG django_auth_ldap uid=tower,cn=users,cn=accounts,dc=local,dc=net is a member of cn=tower_users,cn=groups,cn=accounts,dc=local,dc=net
2018-07-24 04:49:39,278 DEBUG django_auth_ldap Populating Django user jordi
2018-07-24 04:49:39,316 INFO awx.api.views User tower logged in
2018-07-24 04:49:42,754 DEBUG awx.main.scheduler Running Tower task manager.
2018-07-24 04:49:42,760 DEBUG awx.main.scheduler Starting Scheduler
2018-07-24 04:49:50,678 DEBUG awx.main.tasks Last scheduler run was: 2018-07-23 19:49:20.666694+00:00
```

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Enjoy!


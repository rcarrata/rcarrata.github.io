---
layout: single
title: Making accesible from our laptop an OpenShift 4 UPI Libvirt/KVM Lab
date: 2020-04-21
type: post
published: true
status: publish
categories:
- OpenShift
- Networking
- Administration
- Kubernetes
tags: []
author: rcarrata
comments: true
---


How to expose certain ports of our VMs, to for example expose our Load Balancer port balancing
itself the port of the API of OpenShift4? How to access to the DNS configured in our helper node
and not reachable from outside?

Let's start with the always fancy world of iptables / linux routing!

### Overview

For deploy OpenShift 4 with UPI in Baremetal we will use our small lab and KVM and libvirt for
manage the networking, storage, and the own VMs (the process is out of scope of this article).

Due to the not possibility in Baremetal (and due to is a lab), we need to have available a
DNS (bind), LB (haproxy), and several utils more to provide the enough requirements and tools for
the correct work of our OpenShift cluster. This is usually installed (in lab only, NOT use in prod
:D), in a helper node with a well known and fixed IP.

Once is installed, we can access inside of this VM of helper, to the OpenShift cluster... but how
about access from anywhere, and not restrained inside of the helper VM?

For doing this, we will play with iptables, linux routing, dnsmasq and some tricks :D

Let's dig in!

### Scenario: Lab analysis

As always, we need to have our cluster installed and up && running, remember that all are VMs
running with libvirt inside of the hipervisor, in a nat network:

```
[root@hypervisor ~]# virsh list
 Id   Name           State
------------------------------
 6    ocp4-master0   running
 7    ocp4-master1   running
 8    ocp4-master2   running
 9    ocp4-worker0   running
 10   ocp4-worker1   running
 11   ocp4-worker2   running
 16   ocp4-aHelper   running
```

The network libvirt is used with a nat and with in my case the net ip address of 192.167.7.0/24:

```
[root@hypervisor ~]# virsh net-dumpxml openshift4 | grep -E 'forward|ip'
  <forward mode='nat'>
  </forward>
  <ip address='192.168.7.1' netmask='255.255.255.0'>
  </ip>
```

The helper node have the ip address of 192.168.7.77 (within the nat ip addresses described before):

```
# ssh 192.168.7.77
Last login: Mon Apr 20 09:48:53 2020 from 192.168.7.1
[root@helper ~]#
```

### Prerequisites

For doing this lab, and access to our cluster from outside we need to have installed some prereqs.

In the hypervisor remove firewalld and install iptables-service:

```
[root@hypervisor ~]# yum remove firewalld -y
[root@hypervisor ~]# yum install -y iptables-service
[root@hypervisor ~]# systemctl enable iptables
```

Backup the original iptables, flush the current ones, and save them into the /etc/sysconfig/iptables,
for make it permanently.

```
[root@hypervisor ~]# cp -p /etc/sysconfig/iptables /etc/sysconfig/iptables.bak
[root@hypervisor ~]# iptables -F
[root@hypervisor ~]# iptables-save > /etc/sysconfig/iptables
[root@hypervisor ~]# systemctl start iptables
```

### Enable IP Packet Forwarding

By default any modern Linux distributions will have IP Forwarding disabled. To enable IP packet forwarding enable it into the /etc/sysctl.conf:

```
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
```

Enable and verify your settings:

```
/sbin/sysctl -p
```

### Rules for access from our VMs to Internet (Masquerade / SNAT)

For giving access to the internet into our VMs, libvirt is configuring automatically an SNAT/MASQUERADE:

* SNAT (Change of Source Ip Direction - Source NAT): the public IP that is doing the substitution to the Source IP
  is static
* IP MASQUERADE: Dir public IP that is doing the substitution to the source is dinamic.

To manual configuration, use the iptables with postrouting and masquerade options:

```
[root@hypervisor ~]# iptables -t nat -A POSTROUTING -s 192.168.7.0/24 -o eno1 -j MASQUERADE
```

NOTE: because of the own nature of the SNAT, the NAT will change the source IP, but this is
performed just BEFORE to send the packet to Internet, so need to be defined into the POSTROUTING
chain

```
[root@hypervisor ~]# iptables -L POSTROUTING -n -t nat
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  192.168.7.0/24       0.0.0.0/0
```

NOTE: We only are interested in the nat table and we can filter also by POSTROUTING, due to in this
step the POSTROUTING chains will be configured.

Inside of the VM check that we have access to outside:

```
[root@helper ~]# ping -c1 www.bbc.co.uk
PING www.bbc.net.uk (212.58.237.253) 56(84) bytes of data.
64 bytes from 212.58.237.253 (212.58.237.253): icmp_seq=1 ttl=42 time=81.2 ms

--- www.bbc.net.uk ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 81.242/81.242/81.242/0.000 ms
```

### Rules to port forwarding and reach our OCP cluster from our laptop (DNAT)

In this case, for expose our ports through our hipervisor and reach them from the outside. This is
very useful for example, in the case that our laptop only reaches the hipervisor, but not the VM directly
because is in a NAT libvirt network.

For doing this, we can use iptables with DNAT rules or Destination NAT Rules (can be also
be understanding as Port Forwarding). The Destination NAT, will be originated outside of our VM, and
will have modified the Destination IP address during the connection.

For implement this DNAT we will use iptables and the PREROUTING table. In this case, we want to
forward the port of the dns (53), to be reachable from outside of the VM (and furthermore outside of
the hipervisor).

The example of the iptables command is:

```
iptables -t nat -A PREROUTING -p <tcp/udp> --dport <dest_port> -i <hipervisor_internet_iface> -j DNAT --to <VM_Dest_IP>
```

```
[root@hypervisor ~]# iptables -t nat -A PREROUTING -p udp --dport 53 -i eno1 -j DNAT --to 192.168.7.77
[root@hypervisor ~]# iptables -t nat -A PREROUTING -p tcp --dport 53 -i eno1 -j DNAT --to 192.168.7.77
```

NOTE: In this case we are using the PREROUTING, because we need to modify the incoming packets
BEFORE to take an routing decision. Also the DNAT needs to be specified in this case.

Also, we need to implement the DNAT, forwarding the port 6443 and 443 that is used in this case by the
Haproxy for load balancing our OCP4 Api and Apps routers:

```
[root@hypervisor ~]# iptables -t nat -A PREROUTING -p tcp --dport 6443 -i eno1 -j DNAT --to 192.168.7.77
[root@hypervisor ~]# iptables -t nat -A PREROUTING -p tcp --dport 443 -i eno1 -j DNAT --to 192.168.7.77
[root@hypervisor ~]# iptables -t nat -A PREROUTING -p tcp --dport 80 -i eno1 -j DNAT --to 192.168.7.77
```

Check the iptables nat chain filtering by PREROUTING:

```
[root@hypervisor ~]# iptables -t nat -L PREROUTING -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:6443 to:192.168.7.77
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 to:192.168.7.77
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 to:192.168.7.77
DNAT       udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 to:192.168.7.77
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:53 to:192.168.7.77
```

From outside of the hypervisor we can telnet each port that we included in the DNAT rules:

```
[rcarrata@laptop]$ telnet xx.1.8.85 6443
Connected to xx.1.8.85.
Escape character is '^]'

[rcarrata@laptop]$ telnet xx.1.8.85 443
Connected to xx.1.8.85.
Escape character is '^]'.

[rcarrata@laptop]$ telnet xx.1.8.85 53
Connected to xx.1.8.85.
Escape character is '^]'.
```

It works!

Save the iptables permanently into the iptables file and restart for ensure that are applied:

```
[root@hypervisor ~]# iptables-save > /etc/sysconfig/iptables
[root@hypervisor ~]# systemctl restart iptables
```

### Configure dnsmasq in localhost for access to the cluster by DNS

NetworkManager needs to be configured to use dnsmasq for DNS.

Ensure that the dnsmasq is installed:

```
[rcarrata@laptop ~] sudo dnf install dnsmasq
```

Add a file to /etc/NetworkManager/conf.d to enable use of dnsmasq:

```
[rcarrata@laptop ~]$ sudo tee /etc/NetworkManager/conf.d/use-dnsmasq.conf &>/dev/null <<EOF
[main]
dns=dnsmasq
EOF
```

Add dns entries for the ocp4 cluster:

```
$ tee external-ocp4.conf &>/dev/null <<EOF
address=/apps.<domain_ocp4_cluster>/$SERVER_IP
address=/api.<domain_ocp4_cluster>/$SERVER_IP
EOF
```

In our case, we substitute for the Hypervisor IP that is capable now to port forward to the VM of the helper,
and through the haproxy to the cluster of OpenShift4 in our VMs:

```
[rcarrata@laptop ~]$ external-ocp4.conf
address=/apps.ocp4.rglab.com/xx.1.8.85
address=/api.ocp4.rglab.com/xx.1.8.85

$ export SERVER_IP=”xx.1.8.85”
$ sed -i "s/SERVER_IP/$SERVER_IP/g" external-ocp4.conf
$ sudo cp external-ocp4.conf /etc/NetworkManager/dnsmasq.d/external-ocp4.conf
$ sudo systemctl reload NetworkManager
```

Finally check that is possible to reach the OCP Api from our laptop:

```
[rcarrata@laptop ~]$ oc login https://api.ocp4.rglab.com:6443
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by
others.
Use insecure connections? (y/n): y

Authentication required for https://api.ocp4.rglab.com:6443 (openshift)
Username: kubeadmin
Password:
Login successful.

You have access to 54 projects, the list has been suppressed. You can list all projects with 'oc
projects'

Using project "openshift-marketplace".
```

And voilà! Now we can access the helper vm and therefore the cluster from our laptop :)

*NOTE: Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.*

Happy OpenShifting!


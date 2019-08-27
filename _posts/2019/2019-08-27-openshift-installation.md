---
layout: post
title:	"Openshift Installation"
date:	2019-08-27 12:00:00
categories:
    - blog
tags:
    - openshift
    - okd
    - install
    - hetzner
---
see: https://www.server-world.info/en/note?os=CentOS_7&p=openshift311&f=1

	
Install OpenShift Origin which is the Open Source implementation of Red Hat OpenShift.
This example is based on the environment like follows.

-----------+-----------------------------+-----------------------------+------------
           |10.0.0.25                    |10.0.0.51                    |10.0.0.52
+----------+---------------+      +----------+------------+      +----------+------------+
|  [ master.keepcalm.ch ]  |      | [ node1.keepcalm.ch ] |      | [ node1.keepcalm.ch ] |
|     (Master Node)        |      |    (Compute Node)     |      |    (Compute Node)     |
|     (Infra Node)         |      |                       |      |                       |
|     (Compute Node)       |      |                       |      |                       |
+--------------------------+      +-----------------------+      +-----------------------+


## Hetzner Cloud
You will need an account at: [https://console.hetzner.cloud](https://console.hetzner.cloud)

Additionally you will need the `hcloud` command line tool: [https://github.com/hetznercloud/cli/releases](https://github.com/hetznercloud/cli/releases)

Register your account with `hcloud context create` …. Also see: [https://github.com/hetznercloud/cli#getting-started](https://github.com/hetznercloud/cli#getting-started)

Local configuration
You will need to create a file called config in this directory. You may use the file config.example as a basis for this.

```
marzel > ~  hcloud server create --name master.keepcalm.ch --type cx31 --image centos-7 --ssh-key k8s-monkey --datacenter hel1-dc2
hcloud server create --name node1.keepcalm.ch --type cx11 --image centos-7 --ssh-key k8s-monkey --datacenter hel1-dc2
hcloud server create --name node2.keepcalm.ch --type cx11 --image centos-7 --ssh-key k8s-monkey --datacenter hel1-dc2
```

```
marzel > ~ hcloud server list                                                                                                                                                         25.6s  Tue Aug 27 21:31:31 2019
ID        NAME                 STATUS    IPV4             IPV6                      DATACENTER
3197545   master.keepcalm.ch   running   95.217.2.138     2a01:4f9:c010:524e::/64   hel1-dc2
3197546   node1.keepcalm.ch    running   95.217.9.223     2a01:4f9:c010:59b9::/64   hel1-dc2
3197547   node2.keepcalm.ch    running   95.217.9.247     2a01:4f9:c010:5a82::/64   hel1-dc2
```

## DNS
Update `DNS`  entries.
```
*                   420     IN      CNAME       apps.console
@                   1800    IN      A           95.217.2.138
apps.console        300     IN      A           95.217.2.138
console             300     IN      A           95.217.2.138
master              300     IN      A           95.217.2.138
node1               300     IN      A           95.217.9.223 
node2               300     IN      A           95.217.9.247
```

### Check DNS Entries
Let``s test if the `DNS` are registert. `for x in apps.console.keepcalm.ch console.keepcalm.ch master.keepcalm.ch node1.keepcalm.ch node2.keepcalm.ch; nslookup $x; end`
```bash
marzel > ~  for x in apps.console.keepcalm.ch console.keepcalm.ch master.keepcalm.ch node1.keepcalm.ch node2.keepcalm.ch; nslookup $x; end
Server:		2a02:1205:502f:7030:ead1:1bff:feac:8fc0
Address:	2a02:1205:502f:7030:ead1:1bff:feac:8fc0#53

Non-authoritative answer:
Name:	apps.console.keepcalm.ch
Address: 95.217.2.138

Server:		2a02:1205:502f:7030:ead1:1bff:feac:8fc0
Address:	2a02:1205:502f:7030:ead1:1bff:feac:8fc0#53

Non-authoritative answer:
Name:	console.keepcalm.ch
Address: 95.217.2.138

Server:		2a02:1205:502f:7030:ead1:1bff:feac:8fc0
Address:	2a02:1205:502f:7030:ead1:1bff:feac:8fc0#53

Non-authoritative answer:
Name:	master.keepcalm.ch
Address: 95.217.2.138

Server:		2a02:1205:502f:7030:ead1:1bff:feac:8fc0
Address:	2a02:1205:502f:7030:ead1:1bff:feac:8fc0#53

Non-authoritative answer:
Name:	node1.keepcalm.ch
Address: 95.217.9.223

Server:		2a02:1205:502f:7030:ead1:1bff:feac:8fc0
Address:	2a02:1205:502f:7030:ead1:1bff:feac:8fc0#53

Non-authoritative answer:
Name:	node2.keepcalm.ch
Address: 95.217.9.247
``` 

## Install OKD 3.11
On All Nodes, Create a user for installation to be used in Ansible and also grant root privileges to him.
On All Nodes, install OpenShift Origin 3.11 repository and Docker and so on.
``` 
marzel > ~  for x in  master.keepcalm.ch node1.keepcalm.ch node2.keepcalm.ch; ssh root@$x; end 

[root@master ~]# useradd origin 
[root@master ~]# passwd origin 
[root@master ~]# echo -e 'Defaults:origin !requiretty\norigin ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/openshift 
[root@master ~]# chmod 440 /etc/sudoers.d/openshift
[root@master ~]# yum -y install centos-release-openshift-origin311 epel-release docker git pyOpenSSL
[root@master ~]# systemctl start docker 
[root@master ~]# systemctl enable docker  
[root@master ~]# exit

[root@node1 ~]# useradd origin 
[root@node1 ~]# passwd origin 
[root@node1 ~]# echo -e 'Defaults:origin !requiretty\norigin ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/openshift 
[root@node1 ~]# chmod 440 /etc/sudoers.d/openshift
[root@node1 ~]# yum -y install centos-release-openshift-origin311 epel-release docker git pyOpenSSL
[root@node1 ~]# systemctl start docker 
[root@node1 ~]# systemctl enable docker
[root@node1 ~]# exit

[root@node2 ~]# useradd origin 
[root@node2 ~]# passwd origin 
[root@node2 ~]# echo -e 'Defaults:origin !requiretty\norigin ALL = (root) NOPASSWD:ALL' | tee /etc/sudoers.d/openshift 
[root@node2 ~]# chmod 440 /etc/sudoers.d/openshift
[root@node2 ~]# yum -y install centos-release-openshift-origin311 epel-release docker git pyOpenSSL
[root@node2 ~]# systemctl start docker 
[root@node2 ~]# systemctl enable docker    
[root@node2 ~]# exit    
```

On Master Node, login with a user created above and set SSH keypair with no pass-phrase.
``` 
marzel > ~ ssh origin@master.keepcalm.ch  
origin@master.keepcalm.ch's password:
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8): No such file or directory
[origin@master ~]$ ssh-keygen -q -N ""
Enter file in which to save the key (/home/origin/.ssh/id_rsa):
[origin@master ~]$ vi ~/.ssh/config
```
```bash
# create new ( define each node )
Host master
    Hostname master.keepcalm.ch
    User origin
Host node1
    Hostname node1.keepcalm.ch
    User origin
Host node2
    Hostname node2.keepcalm.ch
    User origin
```

```
[origin@master ~]$ chmod 600 ~/.ssh/config
```

transfer public-key to other nodes
```
[origin@master ~]$ ssh-copy-id node1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/origin/.ssh/id_rsa.pub"
The authenticity of host 'node1.keepcalm.ch (95.217.9.223)' can't be established.
ECDSA key fingerprint is SHA256:89gWWsiDU+pkMP28UZN9Uxt8uUhFNHPDaAu11enxmmo.
ECDSA key fingerprint is MD5:08:62:d9:88:a4:32:99:a0:63:4c:79:69:ac:e3:86:3f.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
origin@node1.keepcalm.ch's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'node1'"
and check to make sure that only the key(s) you wanted were added.

```

And also for the `node2` and `master`
``` 
[origin@master ~]$ ssh-copy-id node2
[origin@master ~]$ ssh-copy-id master
```

On Master Node, login with a user created above and run Ansible Playbook for setting up OpenShift Cluster.
``` 
[origin@master ~]$ sudo yum -y install openshift-ansible
[origin@master ~]$ sudo vi /etc/ansible/hosts
```

```yaml
# add follows to the end
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
# admin user created in previous section
ansible_ssh_user=origin
ansible_become=true
openshift_deployment_type=origin

# use HTPasswd for authentication
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
# define default sub-domain for Master node
openshift_master_default_subdomain=apps.keepcalm.ch
# allow unencrypted connection within cluster
openshift_docker_insecure_registries=172.30.0.0/16

[masters]
master.keepcalm.ch openshift_schedulable=true containerized=false

[etcd]
master.keepcalm.ch

[nodes]
# defined values for [openshift_node_group_name] in the file below
# [/usr/share/ansible/openshift-ansible/roles/openshift_facts/defaults/main.yml]
master.keepcalm.ch openshift_node_group_name='node-config-master-infra'
node1.keepcalm.ch openshift_node_group_name='node-config-compute'
node1.keepcalm.ch openshift_node_group_name='node-config-compute'

# if you'd like to separate Master node feature and Infra node feature, set like follows
# master.keepcalm.ch openshift_node_group_name='node-config-master'
# node1.keepcalm.ch openshift_node_group_name='node-config-compute'
# node2.keepcalm.ch openshift_node_group_name='node-config-infra'
```
run Prerequisites Playbook
``` 
[origin@master ~]$ ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml 
```

![prerequisites](/images/posts/2019/openshift-install/prerequisites.png)

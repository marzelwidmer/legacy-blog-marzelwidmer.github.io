---
layout: post
title: Install OKD on Hetzner Cloud
date: 2019-01-13
#last_modified:
description: Install Single Node OKD on Hetzner Cloud # Add post description (optional)
img: 2019/hetzner-okd/cloudcomputing.jpg # Add image post (optional)
tags: [Blog, Kubernetes, Hetzner, Infrastructure, OpenShift]
author: # Add name author (optional)
---

Inspiration from [Installation of OKD 3.10 from start to finish](https://www.youtube.com/watch?v=ZkFIozGY0IA){:target="_blank"}

# Install OKD on Hetzner Cloud
## Clone Repository

```bash
$ git clone https://github.com/c3smonkey/hetzner-okd-ansible
```
## Create VM

This will install a _Cluster Admin_ you can configure, it will also create a default _Cluster User_ with user name `developer`
and password `developer`.   

First you need a account on [Hetzner Cloud]{https://console.hetzner.cloud}{:target="_blank"}
Then you install the [Hettzner CLI](https://github.com/hetznercloud/cli){:target="_blank"} because we will create the VM's from the `CLI`
The installation of a `OKD` takes around 25-30min. 

When you check the [Hetzner VM's](https://www.hetzner.de/cloud){:target="_blank"} you will se there are many different from `CX11` with around 2.96 EUR
till `CX51` with around 35.58 EUR each month. I was starting with the `CX11` and end up now with the `CX41` this have enough performance to work with.

So now let's start setup our installation with :
```bash
$ hcloud server create --name <YOUR_DOMAIN> --type cx41 --image centos-7 --ssh-key <YOUR_HETZNER_SSH_KEY> --datacenter hel1-dc2
```
 
if you have already one and want it just rest and re-install the cluster you can user the following command.
> ⚠️ **re-install**: hcloud server rebuild <your-domain> --image centos-7

## Configure Domain
Update your domain stuff in the `okd.yml` file.
```yaml
  vars:
    - okd_domain: "your-domain"
    - okd_admin_username: "your-admin-user"
    - okd_admin_password: "your-admin-user-password"
```    

Check _`ansible.cfg`_ file your private key file have the same name, otherwise update it
```bash
$ private_key_file =  ~/.ssh/id_rsa_hetzner
```

## Playbook
### Call playbook
```bash
$ ansible-playbook -i inventory.ini okd.yml
```

Wait for the summary...

```bash
Sunday 13 January 2019  09:45:58 +0100 (0:00:07.390)       0:27:30.351 ********
===============================================================================
common : Install OKD on Hetzner -------------------------------------- 1631.71s
~/hetzner-okd-ansible/roles/common/tasks/main.yml:2----------------------------
common : Read installation log file ------------------------------------- 7.39s
~//hetzner/hetzner-okd-ansible/roles/common/tasks/main.yml:32 -----------------
common : Install Git applications --------------------------------------- 3.89s
~/hetzner/hetzner-okd-ansible/roles/common/tasks/main.yml:2 -------------------
common : Transfer the installation script ------------------------------- 3.04s
~/hetzner/hetzner-okd-ansible/roles/common/tasks/main.yml:14 ------------------
common : Clone OKD installation repository ------------------------------ 2.72s
~/hetzner/hetzner-okd-ansible/roles/common/tasks/main.yml:10 ------------------
common : Check Git version ---------------------------------------------- 1.47s
~/hetzner/hetzner-okd-ansible/roles/common/tasks/main.yml:7 -------------------
```


# Ansible hints

## Playbook syntax check
```bash
$ ansible-playbook -i inventory.ini okd.yml --syntax-check
```
## Uptime
```bash
$ ansible hetzner-vm -a uptime
```
## Ping
```bash
$ ansible hetzner-vm -m ping
```
## Ansible Long version
### Ping
```bash
$ ansible -i inventory.ini hetzner-vm -m ping --verbose --user root
```
### Call localhost with ansible
#### Localhost Ping
```bash
$ ansible --connection local -i inventory.ini hetzner-vm -m ping --verbose --user root
```

#### Localhost Uptime
```bash
$ ansible --connection local -i inventory.ini hetzner-vm -a uptime --verbose --user root
```
#### hcloud server list

```bash
$ ansible --connection local -i inventory.ini local -a 'hcloud server list' --verbose --user root
```

## SSL
To configure your certificate put the following inside your inventory files under [OSEv3:vars]
```properties
# Cert
openshift_master_overwrite_named_certificates=true 
openshift_master_named_certificates=[{"certfile": "/etc/path-to/cert.pem", "keyfile": "/etc/path-to/key.pem", "names": ["console.okd.domain.io"], "cafile": "/etc/path-to/ca.cer"}] 
openshift_hosted_router_certificate={"certfile": "/etc/path-to/cert.pem", "keyfile": "/etc/path-to/key.pem", "cafile": "/etc/path-to/ca.cer"} 
```

If you setup your cert the first time, you must run the following playbook:
```bash
$ /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

For renewal, the following is enough, but only if the domain name don't change:
```bash
$  /usr/share/ansible/openshift-ansible/playbooks/redeploy-certificates.yml
```

More details check this out [Certbot - let`s Encrypt/](http://blog.marcelwidmer.org/certbot-letsencrypt/)

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/



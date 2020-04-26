---
layout: post
title: Install OKD on Hetzner Cloud
date: 2019-01-13
last_modified: 2020-04-26
description: Install Single Node OKD on Hetzner Cloud # Add post description (optional)
img: 2019/hetzner-okd/cloudcomputing.jpg # Add image post (optional)
tags: [Blog, Kubernetes, Hetzner, OpenShift]
author: # Add name author (optional)
---
Inspiration from [Installation of OKD 3.10 from start to finish](https://www.youtube.com/watch?v=ZkFIozGY0IA){:target="_blank"}

Create a `Hetzner VM` with the CLI https://github.com/hetznercloud/cli

# hcloud (Hetzner CLI)
Let's check first some `hcloud` command that we can use later to create a `VM` with the right size and in the `Datacenter` we want.


![hetzner-preis](/assets/img/2019/hetzner-okd/hetzner-preis.png)


```bash
hcloud server create --name <YOUR_DOMAIN> --type <SERVER-TYPE> --image <IMAGE> --ssh-key <YOUR_HETZNER_SSH_KEY> --datacenter <DATACENTER>
```

## Server Type
```bash
hcloud server-type list
ID   NAME        CORES   MEMORY     DISK     STORAGE TYPE
1    cx11        1       2.0 GB     20 GB    local
2    cx11-ceph   1       2.0 GB     20 GB    network
3    cx21        2       4.0 GB     40 GB    local
4    cx21-ceph   2       4.0 GB     40 GB    network
5    cx31        2       8.0 GB     80 GB    local
6    cx31-ceph   2       8.0 GB     80 GB    network
7    cx41        4       16.0 GB    160 GB   local
8    cx41-ceph   4       16.0 GB    160 GB   network
9    cx51        8       32.0 GB    240 GB   local
10   cx51-ceph   8       32.0 GB    240 GB   network
11   ccx11       2       8.0 GB     80 GB    local
12   ccx21       4       16.0 GB    160 GB   local
13   ccx31       8       32.0 GB    240 GB   local
14   ccx41       16      64.0 GB    360 GB   local
15   ccx51       32      128.0 GB   600 GB   local
22   cpx11       2       2.0 GB     40 GB    local
23   cpx21       3       4.0 GB     80 GB    local
24   cpx31       4       8.0 GB     160 GB   local
25   cpx41       8       16.0 GB    240 GB   local
26   cpx51       16      32.0 GB    360 GB   local
```

## Image
```bash
hcloud image list
ID         TYPE     NAME           DESCRIPTION    IMAGE SIZE   DISK SIZE   CREATED
1          system   ubuntu-16.04   Ubuntu 16.04   -            5 GB        2 years ago
2          system   debian-9       Debian 9       -            5 GB        2 years ago
3          system   centos-7       CentOS 7       -            5 GB        2 years ago
168855     system   ubuntu-18.04   Ubuntu 18.04   -            5 GB        2 years ago
5924233    system   debian-10      Debian 10      -            5 GB        9 months ago
8356453    system   centos-8       CentOS 8       -            5 GB        6 months ago
9032843    system   fedora-31      Fedora 31      -            5 GB        5 months ago
15512617   system   ubuntu-20.04   Ubuntu 20.04   -            5 GB        2 days ago
```

## Datacenter
```bash
hcloud datacenter list
ID   NAME        DESCRIPTION          LOCATION
2    nbg1-dc3    Nuremberg 1 DC 3     nbg1
3    hel1-dc2    Helsinki 1 DC 2      hel1
4    fsn1-dc14   Falkenstein 1 DC14   fsn1
``` 

## Create Server
Let's create a Server `cx41` with CentOS `centos-7` (ecause of some issues with `centos-8`) on a data center in Nuremberg `nbg1-dc3`.
When the Server is we will get a Public IP `116.203.16.100`.   


```bash
hcloud server create --name keepcalm.ch --type cx41 --image centos-8 --ssh-key ~/.ssh/id_rsa_hetzner.pub --datacenter nbg1-dc3

6s [=====================================] 100.00%
Waiting for server 5577861 to have started
 ... done
Server 5577861 created
IPv4: 116.203.16.100
```


### Rebuild Server 
if you have already one and want it just rest and re-install the cluster you can user the following command.
```bash
hcloud server rebuild keepcalm.ch --image centos-7
```

### Delete Server
```bash
hcloud server delete keepcalm.ch
```

# Configure DNS
Configure now the following `DNS` entries in your `DNS` provider with the `IP` from `Hetzner` in my case I have [gandi.net](https://www.gandi.net/en).

```text
@ 10800 IN SOA ns1.gandi.net. hostmaster.gandi.net. 1585063961 10800 3600 604800 10800
* 420 IN CNAME apps.console
@ 1800 IN A 116.203.16.100
apps.console 300 IN A 116.203.16.100
console 300 IN A 116.203.16.100
```

# Install OKD
Now you can connect with `SSH` to your `VM`

```bash
ssh -i ~/.ssh/id_rsa_hetzner root@keepcalm.ch
The authenticity of host 'keepcalm.ch (116.203.16.100)' can't be established.
ECDSA key fingerprint is SHA256:mc7itrGp4777okbKnKDKPDlQwkMi0e4awyh6cfssNXM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'keepcalm.ch,116.203.16.100' (ECDSA) to the list of known hosts.
Last failed login: Sun Apr 26 12:02:57 CEST 2020 from 51.77.212.235 on ssh:notty
There were 3 failed login attempts since the last successful login.
[root@keepcalm ~]#
```

## Update System and install GIT
```bash
yum -y update && yum -y install git
```

## Clone Git Repo
```bash
git clone https://github.com/marzelwidmer/installcentos.git
```

## Update user-custom-exports script
Update `user-custom-exports.sh` script with your domain specific settings.
```bash
cd installcentos && vi user-custom-exports.sh
``` 

```text
#!/bin/bash
export DOMAIN="keepcalm.ch"
export USERNAME="admin"
export PASSWORD="password"
export MAIL="marzelwidmer@gmail.com"

export SCRIPT_REPO=""
export IP=""
export DISK=""
```

## Load user-custom-exports script
Execute the shell script to load the variables 
```bash
. user-custom-exports.sh
```

## Start install script
Now we can execute the installation script and press enter till the question `Do you wish to enable HTTPS with Let``s Encrypt` 
There we choose `1) Yes`. 

```bash
root@keepcalm installcentos]# ./install-openshift.sh
Domain to use: (keepcalm.ch):
Username: (admin):
Password: (password):
OpenShift Version: (3.11):
IP: (0):
API Port: (8443):
Do you wish to enable HTTPS with Let's Encrypt?
Warnings:
  Let's Encrypt only works if the IP is using publicly accessible IP and custom certificates.
  This feature doesn't work with OpenShift CLI for now.
1) Yes
2) No
```

The following questions we answer with yes. 

```bash
Complete!
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing to share your email address with the Electronic Frontier
Foundation, a founding partner of the Let's Encrypt project and the non-profit
organization that develops Certbot? We'd like to send you email about our work
encrypting the web, EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: yes
Starting new HTTPS connection (1): supporters.eff.org
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for apps.keepcalm.ch
dns-01 challenge for keepcalm.ch
dns-01 challenge for keepcalm.ch

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: yes
```


> ⚠️ **Important Steps**: Be sure you update the `DNS TXT` records with the prompted values 

The following values you have to add in you `DNS`.
So let's open again the `DNS` Admin console from your provider. 

```bash 
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.apps.keepcalm.ch with the following value:

xj9uF017asqGXhuS1VPhA3idwmUq__UgUTscTMYgbxI

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

In my case something like so :

![_acme-challenge.apps](/assets/img/2019/hetzner-okd/acme.png)

You can check the `DNS` with [https://dnschecker.org/](https://dnschecker.org/#TXT/_acme-challenge.apps.keepcalm.ch)
Check with `host` command.
```bash
host -t txt _acme-challenge.apps.keepcalm.ch
_acme-challenge.apps.keepcalm.ch descriptive text "xj9uF017asqGXhuS1VPhA3idwmUq__UgUTscTMYgbxI"
```

Your `DNS` will look like this:
```text
@ 10800 IN SOA ns1.gandi.net. hostmaster.gandi.net. 1587897751 10800 3600 604800 10800
* 420 IN CNAME apps.console
@ 1800 IN A 116.203.16.100
_acme-challenge 300 IN TXT "H2S720yowtzUBVKq3EbH6W8rmgg7qzF0EEVF_IZOz9c"
_acme-challenge 300 IN TXT "dpMeonbQDeHkbJopDOuXw1j4NtFHXzEvr8CGutiRX8w"
_acme-challenge.apps 300 IN TXT "xj9uF017asqGXhuS1VPhA3idwmUq__UgUTscTMYgbxI"
apps.console 300 IN A 116.203.16.100
console 300 IN A 116.203.16.100
```

```bash
Before continuing, verify the record is deployed.
(This must be set up in addition to the previous challenges; do not remove,
replace, or undo the previous challenge tasks yet. Note that you might be
asked to create multiple distinct TXT records with the same name. This is
permitted by DNS standards.)

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
Waiting for verification...
Resetting dropped connection: acme-v02.api.letsencrypt.org
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/keepcalm.ch/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/keepcalm.ch/privkey.pem
   Your cert will expire on 2020-07-25. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```

Now you have to wait some minutes till the installation is finished.

```bash
created volume 199
persistentvolume/vol200 created
created volume 200
******
* Your console is https://console.keepcalm.ch:8443
* Your username is admin
* Your password is password
*
* Login using:
*
$ oc login -u admin -p password https://console.keepcalm.ch:8443/
******
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-public
    kube-service-catalog
    kube-system
    management-infra
    openshift
    openshift-console
    openshift-infra
    openshift-logging
    openshift-metrics-server
    openshift-monitoring
    openshift-node
    openshift-sdn
    openshift-template-service-broker
    openshift-web-console

Using project "default".
[root@keepcalm installcentos]#
```

After the installation is finish you can on : [https://console.keepcalm.ch:8443/console/catalog](https://console.keepcalm.ch:8443/console/catalog)
You have now a `OKD` instance with Let's Encrypt.

![hetzner-preis](/assets/img/2019/hetzner-okd/console.png)


# Renewal Let's Encrypt 
to renewal the Let's Encrypt you can log in to your `VM` and execute the following scrupts 
two ansible scripts in the `installcentos/` folder. 
```bash
 . user-custom-exports.sh
 ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml
 ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml
```



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/



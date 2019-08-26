---
layout: post
title:	"Openshift Certbot Let`s Encrypt"
date:	2019-08-24 12:00:00
categories:
    - blog
tags:
    - openshift
    - okd
    - letsencrypt
    - ssl
---

## Check Certificate
Let`s check first if there a certificate already for our domain [https://crt.sh/?q=c3smonkey.ch](https://crt.sh/?q=c3smonkey.ch) 

![ImageStreams](/images/posts/2019/openshift-certbot-certificate/crt.png)

    
## Check EPEL Reposittory
Check if the ``epel.repo`` is enabled.
```bash
vi /etc/yum.repos.d/epel.repo
```

```yaml
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

```
change ``enabled=0`` to ``enabled=1`` and quite and save the file with `:qw`


## Login to the Cluster 
Login to the Cluster with the `oc login` command. Check if you are in the project ``default`` other wise change to the ``default`` project with `oc project default`
```bash
[root@c3smonkey ~]# oc login
Authentication required for https://console.c3smonkey.ch:8443 (openshift)
Username: monkey
Password: 
Login successful.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    development
    jenkins
```
Verify if you are in the ``default`` project namespace.
```bash
[root@c3smonkey ~]# oc project
Using project "default" on server "https://console.c3smonkey.ch:8443".
```
To create our certificate with ``certbot`` we take the temporary webserver approach. For this we have to scale done one ``pod`` on the cluster 
who are running already on the port ``80``.

### Search Router and scale it down.

With the `oc get dc` we can check if the router is running.
```bash
[root@c3smonkey ~]# oc get dc
NAME               REVISION   DESIRED   CURRENT   TRIGGERED BY
docker-registry    1          1         1         config
registry-console   1          1         1         config
router             1          1         1         config
```
Now we can scale it down with `oc scale --replicas=0 dc router`
```bash
oc scale --replicas=0 dc router
deploymentconfig.apps.openshift.io/router scaled
```
Lets verify if the router not running before we install and run the ``certbot`` with the `oc get dc` command like before.

```bash
[root@c3smonkey ~]# oc get dc
NAME               REVISION   DESIRED   CURRENT   TRIGGERED BY
docker-registry    1          1         1         config
registry-console   1          1         1         config
router             1          0         0         config
```

Lets install the `certbot` now with the `yum install certbot` command.
```bash
[root@c3smonkey ~]# yum install certbot
```

Check if the DNS server is setup is correct. Important `*` ``CNAME`` is pointing to `apps.console` 
We need this because whne we deploy new application this will create a URL under apps and the namespace.
   
```bash
*                420    IN      CNAME   apps.console
@               1800    IN      A       95.216.193.150
apps.console     300    IN      A       95.216.193.150
console          300    IN      A       95.216.193.150
```

Here you can verify the domain names [dnschecker.org](https://dnschecker.org/#A/c3smonkey.ch)

```
[root@c3smonkey ~]# certbot certonly --server https://acme-v02.api.letsencrypt.org/directory \
    --standalone \
    -d jenkins-jenkins.apps.c3smonkey.ch \
    -d console.c3smonkey.ch \
    -d c3smonkey.ch \
    -d grafana-openshift-monitoring.apps.c3smonkey.ch \
    -d apps.c3smonkey.ch \
    -d hawkular-metrics.apps.c3smonkey.ch
```


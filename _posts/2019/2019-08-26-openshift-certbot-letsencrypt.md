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

    


## Edit epel repo
vi /etc/yum.repos.d/epel.repo
change value enabled from  to 1
```
enabled=1
```
##Search Route
```
oc project default
oc get dc
```
##Scale down Routes
```
oc scale --replicas=0 dc router
```

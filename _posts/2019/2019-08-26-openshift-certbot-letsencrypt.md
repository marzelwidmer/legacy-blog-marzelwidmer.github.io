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

![crt](/images/posts/2019/openshift-certbot-certificate/crt.png)

    
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
Ok let`s create some certificate for the following domain names
- c3smonkey.ch
- console.c3smonkey.ch
- jenkins-jenkins.apps.c3smonkey.ch
- grafana-openshift-monitoring.apps.c3smonkey.ch
- apps.c3smonkey.ch
- hawkular-metrics.apps.c3smonkey.ch

```
[root@c3smonkey ~]# certbot certonly --server https://acme-v02.api.letsencrypt.org/directory \
    --standalone \
    -d console.c3smonkey.ch \
    -d jenkins-jenkins.apps.c3smonkey.ch \
    -d c3smonkey.ch \
    -d grafana-openshift-monitoring.apps.c3smonkey.ch \
    -d apps.c3smonkey.ch \
    -d hawkular-metrics.apps.c3smonkey.ch
```
After this command you see something like this

![certbot-result](/images/posts/2019/openshift-certbot-certificate/certbot-result.png)

The certificate are located under ```/etc/letsencrypt/live/console.c3smonkey.ch/```
``` 
root@c3smonkey installcentos]# ls -lisa /etc/letsencrypt/live/console.c3smonkey.ch/
total 12
1258906 4 drwxr-xr-x. 2 root root 4096 Aug 26 21:06 .
1258902 4 drwx------. 3 root root 4096 Aug 26 21:06 ..
1258915 4 -rw-r--r--. 1 root root  692 Aug 26 21:06 README
1258907 0 lrwxrwxrwx. 1 root root   44 Aug 26 21:06 cert.pem -> ../../archive/console.c3smonkey.ch/cert1.pem
1258909 0 lrwxrwxrwx. 1 root root   45 Aug 26 21:06 chain.pem -> ../../archive/console.c3smonkey.ch/chain1.pem
1258910 0 lrwxrwxrwx. 1 root root   49 Aug 26 21:06 fullchain.pem -> ../../archive/console.c3smonkey.ch/fullchain1.pem
1258908 0 lrwxrwxrwx. 1 root root   47 Aug 26 21:06 privkey.pem -> ../../archive/console.c3smonkey.ch/privkey1.pem
```

Now we can scale up our route with `oc scale --replicas=1 dc router` and verify it with `oc get dc`
``` 
[root@c3smonkey]# oc scale --replicas=1 dc router
deploymentconfig.apps.openshift.io/router scaled
[root@c3smonkey]# oc get dc
NAME               REVISION   DESIRED   CURRENT   TRIGGERED BY
docker-registry    1          1         1         config
registry-console   1          1         1         config
router             1          1         1         config
[root@c3smonkey installcentos]#
```

Now we have to patch the installation `inventory.ini` file for this we will take the `vi`
We will add the following section on the bottom of the file. 

``` 
openshift_master_overwrite_named_certificates=true
openshift_master_named_certificates=[{"certfile": "/etc/letsencrypt/live/console.c3smonkey.ch/fullchain.pem", "keyfile": "/etc/letsencrypt/live/console.c3smonkey.ch/privkey.pem","names": ["c3smonkey.ch", "console.c3smonkey.ch", "jenkins-jenkins.apps.c3smonkey.ch", "grafana-openshift-monitoring.apps.c3smonkey.ch", "apps.c3smonkey.ch", "hawkular-metrics.apps.c3smonkey.ch"] }]
```
The meaning of this configuration is basically that we override the master cluster certificate with the flag `true` in `openshift_master_overwrite_named_certificates=true`
and add all named certificates we created with the `certbot` command from before. 

- c3smonkey.ch
- console.c3smonkey.ch
- jenkins-jenkins.apps.c3smonkey.ch
- grafana-openshift-monitoring.apps.c3smonkey.ch
- apps.c3smonkey.ch
- hawkular-metrics.apps.c3smonkey.ch
    
``` 
[root@c3smonkey ~]# vi installcentos/inventory.ini

[OSEv3:children]
masters
nodes
etcd

[masters]
95.216.193.150 openshift_ip=95.216.193.150 openshift_schedulable=true

[etcd]
95.216.193.150 openshift_ip=95.216.193.150

[nodes]
95.216.193.150 openshift_ip=95.216.193.150 openshift_schedulable=true openshift_node_group_name="node-config-all-in-one"

[OSEv3:vars]
openshift_additional_repos=[{'id': 'centos-paas', 'name': 'centos-paas', 'baseurl' :'https://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin311', 'gpgcheck' :'0', 'enabled' :'1'}]

ansible_ssh_user=root
enable_excluders=False
enable_docker_excluder=False
ansible_service_broker_install=False

containerized=True
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

deployment_type=origin
openshift_deployment_type=origin

template_service_broker_selector={"region":"infra"}
openshift_metrics_image_version="v3.11"
openshift_logging_image_version="v3.11"
openshift_logging_elasticsearch_proxy_image_version="v1.0.0"
openshift_logging_es_nodeselector={"node-role.kubernetes.io/infra":"true"}
logging_elasticsearch_rollout_override=false
osm_use_cockpit=true

openshift_metrics_install_metrics=True
openshift_logging_install_logging=False

openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_file='/etc/origin/master/htpasswd'

openshift_public_hostname=console.c3smonkey.ch
openshift_master_default_subdomain=apps.c3smonkey.ch
openshift_master_api_port=8443
openshift_master_console_port=8443

openshift_master_overwrite_named_certificates=true
openshift_master_named_certificates=[{"certfile": "/etc/letsencrypt/live/console.c3smonkey.ch/fullchain.pem", "keyfile": "/etc/letsencrypt/live/console.c3smonkey.ch/privkey.pem","names": ["c3smonkey.ch", "console.c3smonkey.ch", "jenkins-jenkins.apps.c3smonkey.ch", "grafana-openshift-monitoring.apps.c3smonkey.ch", "apps.c3smonkey.ch", "hawkular-metrics.apps.c3smonkey.ch"] }]
```

let us change the directory first and jump in the installation directory `cd installcentos`
```
[root@c3smonkey ~]# cd installcentos/
```

Now let`s apply two ansible script but for this lets search the ansible scripts we need with `cat install-openshift.sh | grep ansible`
``` 
[root@c3smonkey installcentos]# cat install-openshift.sh | grep ansible
curl -o ansible.rpm https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.6.5-1.el7.ans.noarch.rpm
yum -y --enablerepo=epel install ansible.rpm
[ ! -d openshift-ansible ] && git clone https://github.com/openshift/openshift-ansible.git
# cd openshift-ansible && git fetch && git checkout release-${VERSION} && git checkout e7f05191a1 && cd ..
cd openshift-ansible && git fetch && git checkout release-${VERSION} && cd ..
# https://github.com/openshift/openshift-ansible/issues/11069
ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml
ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml
```

The first one we will apply is the `prerequisites` playbook to check if everything is still good.
``` 
[root@c3smonkey installcentos]# ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml
```

![prerequisites](/images/posts/2019/openshift-certbot-certificate/prerequisites.png)


Lets play the second playbook who will replace the certificate we just created.
``` 
[root@c3smonkey installcentos]# ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml
```

Now reboot the cluster with `shutdown -r now`
```
[root@c3smonkey installcentos]# shutdown -r now
```

![crt](/images/posts/2019/openshift-certbot-certificate/crt-lets-encrypt.png)


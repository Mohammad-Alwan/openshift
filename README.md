firewall-cmd --add-port=1936/tcp --add-port=9000-9999/tcp --add-port=10250-10259/tcp --add-port=10256/tcp --add-port=4789/udp --add-port=6081/udp --add-port=9000-9999/udp --add-port=500/udp --add-port=4500/udp --add-port=30000-32767/tcp --add-port=30000-32767/udp --add-port=6443/tcp --add-port=2379-2380/tcp --add-port 8080/tcp --add-port 53/udp --add-port 53/tcp --add-port 22623/tcp --permanent

https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.15.0/openshift-client-linux-4.15.0.tar.gz
https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.15.0/openshift-install-linux-4.15.0.tar.gz

========================================================================

#####Deploy Openshift 4.14#####
firewall-cmd --add-port={8080,80,443,22623,6443}/tcp --add-port=53/udp --permanent

vi /etc/selinux/config
###
...
SELINUX=permissive
...
SELINUXTYPE=targeted
...
###
setenforce 0

firewall-cmd --add-port=1936/tcp --add-port=9000-9999/tcp --add-port=10250-10259/tcp \
--add-port=10256/tcp --add-port=4789/udp --add-port=6081/udp --add-port=9000-9999/udp \
--add-port=500/udp --add-port=4500/udp --add-port=30000-32767/tcp --add-port=30000-32767/udp \
--add-port=6443/tcp --add-port=2379-2380/tcp --add-port 8080/tcp --add-port 53/udp \
--add-port 53/tcp --add-port 22623/tcp --permanent

firewall-cmd --reload

yum install chrony dnsmasq haproxy httpd 
systemctl enable --now chronyd dnsmasq httpd haproxy
timedatectl set-ntp true
setsebool -P haproxy_connect_any=1
semanage port -a -t http_port_t -p tcp 8080
semanage fcontext -a -t httpd_sys_content_t "/var/www(/.*)?"
restorecon -Rv /var/www/
vi /etc/httpd/conf/httpd.conf
###
...
#Change Listen port from default 80 to 8080 (prevent conflict port)
Listen 8080
...
###

systemctl restart httpd

getsebool -a | grep -i haproxy

vi /etc/dnsmasq.conf
####
interface=<interface_name> ##rhel 9 issue
srv-host=_etcd-server-ssl._tcp.ocp-redhat.test.lab,etcd-0.ocp-redhat.test.lab,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp-redhat.test.lab,etcd-1.ocp-redhat.test.lab,2380,0,10
srv-host=_etcd-server-ssl._tcp.ocp-redhat.test.lab,etcd-2.ocp-redhat.test.lab,2380,0,10
no-dhcp-interface=
no-hosts
addn-hosts=/etc/hosts
domain=ocp-redhat.test.lab
local=/ocp-redhat.test.lab/
server=192.168.1.4
#server=ip_dns_internal
address=/ocp-redhat.test.lab/10.19.4.34
###
systemctl restart dnsmasq

vi /etc/hosts
###
#Bastion
10.19.4.34 bastion.ocp-redhat.test.lab bastion
#Master
10.19.4.35 bootstrap.ocp-redhat.test.lab bootstrap
10.19.4.36 master0.ocp-redhat.test.lab master0
10.19.4.36 etcd-0.ocp-redhat.test.lab
10.19.4.37 master1.ocp-redhat.test.lab master1
10.19.4.37 etcd-1.ocp-redhat.test.lab
10.19.4.38 master2.ocp-redhat.test.lab master2
10.19.4.38 etcd-2.ocp-redhat.test.lab
#Worker
10.19.4.39 worker0.ocp-redhat.test.lab worker0
10.19.4.40 worker1.ocp-redhat.test.lab worker1
##infra (for monitoring and logging usage only with elasticsearch and grafana)
#10.19.4.201 infra0.clustername.fqdn infra0
#10.19.4.202 infra1.clustername.fqdn infra1
#api
10.19.4.34 api-int.ocp-redhat.test.lab api-int
10.19.4.34 api.ocp-redhat.test.lab api
###

vi /etc/haproxy/haproxy.cfg
###
global
       log         127.0.0.1 local2
       pidfile     /var/run/haproxy.pid
       maxconn     4000
       daemon

defaults
       mode                    http
       log                     global
       option                  dontlognull
       option http-server-close
       option                  redispatch
       retries                 3
       timeout http-request    10s
       timeout queue           1m
       timeout connect         10s
       timeout client          1m
       timeout server          1m
       timeout http-keep-alive 10s
       timeout check           10s
       maxconn                 3000

frontend stats
       bind *:1936
       mode            http
       log             global
       maxconn 10
       stats enable
       stats hide-version
       stats refresh 30s
       stats show-node
       stats show-desc Stats for ocp4 cluster
       stats auth admin:ocp4
       stats uri /stats

listen api-server-6443
       bind *:6443
       mode tcp
       server bootstrap bootstrap.ocp-redhat.test.lab:6443 check inter 1s backup
       server master0 master0.ocp-redhat.test.lab:6443 check inter 1s
       server master1 master1.ocp-redhat.test.lab:6443 check inter 1s
       server master2 master2.ocp-redhat.test.lab:6443 check inter 1s

listen machine-config-server-22623
       bind *:22623
       mode tcp
       server bootstrap bootstrap.ocp-redhat.test.lab:22623 check inter 1s backup
       server master0 master0.ocp-redhat.test.lab:22623 check inter 1s
       server master1 master1.ocp-redhat.test.lab:22623 check inter 1s
       server master2 master2.ocp-redhat.test.lab:22623 check inter 1s

listen ingress-router-443
       bind *:443
       mode tcp
       balance source
       server worker0 worker0.ocp-redhat.test.lab:443 check inter 1s
       server worker1 worker1.ocp-redhat.test.lab:443 check inter 1s

listen ingress-router-80
       bind *:80
       mode tcp
       balance source
       server worker0 worker0.ocp-redhat.test.lab:80 check inter 1s
       server worker1 worker1.ocp-redhat.test.lab:80 check inter 1s

###
systemctl restart haproxy

wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.15/openshift-client-linux-4.14.15.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.14.15/openshift-install-linux.tar.gz
tar xvf openshift-client-linux-4.14.15.tar.gz
tar xvf openshift-install-linux.tar.gz
mkdir -p installation/yaml_collection/ && mkdir -p installation/ocp4/ && mkdir -p installation/backup/
cp oc kubectl openshift-install /usr/bin/

ssh-keygen

vi installation/ocp4/install-config.yaml
###
apiVersion: v1
baseDomain: test.lab
compute: 
- hyperthreading: Enabled 
  name: worker
  replicas: 0 
controlPlane: 
  hyperthreading: Enabled 
  name: master
  replicas: 3 
metadata:
  name: ocp-redhat
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OVNKubernetes
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
fips: false 
pullSecret: '{"auths": ...}' 
sshKey: 'ssh-ed25519 AAAA...'
###

cp installation/ocp4/install-config.yaml installation/backup/
openshift-install create manifests --dir installation/ocp4/

vi installation/ocp4/manifests/cluster-scheduler-02-config.yaml
###
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: null
  name: cluster
spec:
  mastersSchedulable: false #Locate the mastersSchedulable parameter and ensure that it is set to false, but If you wanna deploy something on master nodes, like localvolume, pv, pc, storageclass, and others  should have this value masterScheduleable: true 
  policy:
    name: ""
status: {}
###

openshift-install create ignition-configs --dir installation/ocp4/

systemctl restart httpd
chmod 755 installation/ocp4/*.ign
cp installation/ocp4/*.ign /var/www/html/
curl -kv http://10.19.4.34:8080/bootstrap.ign 

nslookup bootstrap.ocp-redhat.test.lab


### Setting Network and Hostname in Boostrap, Master & Worker Node###
sudo -i
nmtui ---> setting network and hostname or set hostname use 'hostnamectl set-hostname bootstrap.ocp-redhat.test.lab'

###Deploying Boostrap, Master & Worker from Ignition###
#for bootstrap node
coreos-installer install --copy-network --insecure-ignition --ignition-url=http://10.19.4.34/bootstrap.ign /dev/sda 
#for master node
coreos-installer install --copy-network --insecure-ignition --ignition-url=http://10.19.4.34/master.ign /dev/sda
#for worker node
coreos-installer install --copy-network --insecure-ignition --ignition-url=http://10.19.4.34/worker.ign /dev/sda

openshift-install wait-for bootstrap-complete --dir=installation/ocp4/ --log-level=debug
cp installation/ocp4/auth/kubeconfig ~/.kube/config
while true; do oc get csr -o name | xargs oc adm certificate approve; sleep 30; done

##Access Console##
oc get route
oc get routes console -n openshift-console -o jsonpath='{"https://"}{.spec.host}{"\n"}'

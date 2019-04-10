# Openshift Hackathon
> We'll install a HA Origin cluster with Container Native Storage.

## 1. Pre-req
* Digital ocean account
    * if you dont have one register here https://m.do.co/c/bc801d22c7b9

## 2. Add your public key to Digital ocean
* ```ssh-keygen```  on your machine and  copy  `~/.ssh/id_rsa.pub`


## 3. Provision master-lb loadbalancer in singapore region
* When provisioning masters-lb  make sure 80,443 are forwarded
* for certificate on https port,  use `passthrough` mechanism.
* We will add master vm's later in the process.

## 4. Provision infra-lb loadbalancer in singapore region
* When provisioning infra-lb  make sure 80,443 are forwarded
* for certificate on https port,  use `passthrough` mechanism.
* We will add node vm's later in the process.

## 5. Provision 3 master vms on Digital Ocean with name  master1,master2,master3 
* Make sure to add your public key that is generated in step #2 as authentication mechanism to VM's 
* Select private networking checkbox on
* Take 8GB Mem  (lastest centos) size for all three VM's
* for each master add additional docker storage volume of size 30GB
* Add these 3 vm's to masters-lb

## 6. Provision 2 infra vms on Digital Ocean with name  infra1,infra2 
* Make sure to add your public key that is generated in step #2 as authentication mechanism to VM's 
* Take 8GB Mem  (lastest centos) size for all  VM's
* Select private networking checkbox on
* for each infra vm add additional docker storage volume of size 30GB
* Add these two vm's to infra LB.

## 7. Provision 2 compute vms on Digital Ocean with name  compute1,compute2 
* Make sure to add your public key that is generated in step #2 as authentication mechanism to VM's 
* Take 8GB Mem  (lastest centos) size for all  VM's
* for each compute vm add additional docker storage volume of size 50GB

## 8. Preparing sample inventory
* Use the inventory https://github.com/debianmaster/okd-hackathon.git  as reference for your inventory file.
* `cd okd-hackathon`
* make a hard link of hosts file in this repo to   /etc/ansible/hosts  using  `sudo ln hosts /etc/ansible/hosts`
* Update inventory file with your own ip values.  Since we dont have a working dns we'll use nip.io  appended to ip address.



## 9. Checks before Starting installation.

### 9.0 Make sure ansible environment is set right.
```shell=
easy_install pip
pip install -U pyopenssl
export ANSIBLE_HOST_KEY_CHECKING=False

ansible all -m ping
```

### 9.1 Make sure latest packages are installed on these vm's
```shell=
ansible all -a "rpm -ivh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm"
ansible all -a "yum install ansible -y"
```

### 9.1 Make sure /etc/hosts does not include an entry like `127.0.0.1 vmshortname`
```shell=
ansible all -a "cat /etc/hosts"
ansible all -m lineinfile -a "path=/etc/hosts regexp={{inventory_hostname}} state=absent"
```
> This will make sure controller,api pods are able to reach out to etc without dns resolution issues.
> Otherwise controllers will try to make an attempt to connect to etcd on  127.0.0.1:2379 which is not accurate.

### 9.1 Make sure Network Manager and DnsMasq are installed and started prior to installation
```shell=
ansible all -a "yum install NetworkManager -y"
ansible all -a "systemctl restart  NetworkManager"

ansible all -a "yum install dnsmasq -y"
ansible all -a "systemctl restart  dnsmasq"
```

### 9.2 Make sure selinux is to enforcing 
```shell=
ansible all  -a "getenforce"
ansible all  -a "setenforce 0"
```

### 9.3 Make sure all VMs are in same time zone and in sync
```shell=
ansible all  -a "date"
ansible all -m shell -a "timedatectl set-timezone UTC"
```

### 9.4  Needed for logging
```shell=
ansible all  -a "sysctl -w vm.max_map_count=262144"
```
### 9.5 Make sure VM's are able to reach out to internet
```shell=
ansible all  -a "ping -c 1 google.com"
```
### 9.6 Take a backup of /etc/resolv.conf incase of install issues we can restore dns state of VM
```sh
ansible all  -a "cp -f /etc/resolv.conf /etc/resolv.conf.upstream"
```
> Tip :  DONOT run this unless required.     to restore /etc/resolv.conf  use following command
> `#ansible all  -a "cp -f /etc/resolv.conf.upstream /etc/resolv.conf"`

### 9.7   Make sure firewalld is disabled.  (We'll use iptables instead of firewalld)
```shell=
ansible all -a "systemctl status firewalld"
ansible all -a "systemctl stop firewalld"
ansible all -a "systemctl disable firewalld"
```

### 9.10 Make sure hostname and hostname -f are all FQDN
```shell=
ansible all -a "hostname"
ansible all -a "hostname -f "
```
>  For your digital ocean VM this might be different.  since we dont have a working dns, and we need a full FQDN  
>  We'll rest the hosntames with their   nip.io  equivalent dns names.   so its consistent everywhere.

#### 9.10.1  Change hosntames of VM's so  `hostname`  and `hostname -f`  are same.
```shell=
ansible all -a "hostnamectl set-hostname  {{inventory_hostname}}"
```
>  you can skip 9.10.1  if your VM's  hostname and hostname -f are both pointing to same FQDN.

### 9.11 Make sure VM's can reachout to themselves and responding with correct similar interface ip on all vm's
```shell=
ansible all -a "ping -c 1 {{inventory_hostname}}"
```

### 9.12 Your VM's may have two network interfaces. openshift will use an interface which satisfies this condition
```shell=
ansible all -a "ip -4 route get 8.8.8.8"
```

>  if you need to change this interface you need to use this command
>  `#ansible all -a "route add -net 8.8.8.8 netmask 255.255.255.255 <interfacename>"`

### 9.13 For the target interface make sure NM_CONTROLLED, PEERDNS, ip_forward are set as below.
> change the interface name if necessary
```shell=
ansible all -m lineinfile -a "path=/etc/sysconfig/network-scripts/ifcfg-eth0 line=NM_CONTROLLED='yes'"
ansible all -m lineinfile -a "path=/etc/sysconfig/network-scripts/ifcfg-eth0 line=PEERDNS='yes'"
ansible all -m sysctl -a "name=net.ipv4.ip_forward value=1 sysctl_set=yes state=present reload=yes"
ansible all -m sysctl -a "name=net.ipv6.conf.all.disable_ipv6 value=1 sysctl_set=yes state=present reload=yes"
ansible all -m sysctl -a "name=net.ipv6.conf.default.disable_ipv6 value=1 sysctl_set=yes state=present reload=yes"
```

### 9.14 Intall rpm dependencies
```shell
ansible all -m shell -a 'yum install  epel-release wget git net-tools bind-utils yum-utils iptables-services bridge-
utils bash-completion kexec-tools sos psacct -y'
ansible masters -m shell -a "yum install httpd-tools -y"
ansible masters -m shell -a "yum install centos-release-openshift-origin311.noarch -y"
```
> Need this for most of the storage clients
```shell
ansible all -m shell -a "yum install samba-client samba-common cifs-utils iscsi-initiator-utils -y"
ansible all -a "systemctl restart iscsid"
ansible all -a "systemctl status iscsid"
```

### 9.15 Install docker
```
ansible all -m shell -a "yum install docker -y"
```

### 9.16 See if you need restarting of server,  restart all hosts if necessary
```
ansible all -m shell -a "/usr/bin/needs-restarting -r"
#ansible all -a "systemctl reboot"
```
### 9.17  Make sure latest rpm repo is copied to all hosts 
> create file named origin.repo 
```shell=
[origin-repo]
name=Origin RPMs
baseurl=http://mirror.centos.org/centos/7/paas/x86_64/openshift-origin/
enabled=1
gpgcheck=0
```
```shell=
ansible all -m copy -a "src=origin.repo dest=/etc/yum.repos.d/origin.repo"
```


### Install the cluster
```shell=
ansible-playbook ~/openshift-ansible/playbooks/byo/openshift_facts.yml
ansible-playbook ~/openshift-ansible/playbooks/prerequisites.yml
ansible-playbook ~/openshift-ansible/playbooks/deploy_cluster.yml
```




































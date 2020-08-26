---
title: Jenkins  HA Setup
categories:
- DevOps
# feature_image: "https://picsum.photos/2560/600?image=872"
---

## To achieve High Availability and DR setup for Jenkins master.


### Steps:

1. NFS file storage server.
2. Cloud instances for Jenkins Primary and secondary master servers.
3. HA proxy Instance.

### Network:

Since all the instances used here are part of a cloud VPC and can ping each other, we used private IP's for all the connection configuration.

* NFS (10.0.0.2)
* Jenkins primary (10.0.0.3)
* Jenkins secondary (10.0.0.4)

### Setup:

# NFS file store server

This is with respect to the centos / RedHat cloud instance. Install NFS client with the below commands.
```console
yum install nfs-utils
```
create a directory inside NFS file storage instance, which will be used as Jenkins shared file storage
```console
mkdir /var/lib/jenkins
```
modify the user permissions and ownership for the file storage directory so that there won't be any file permission related issues in the future.
```console
chmod -R 755 /var/lib/jenkins
chown nfsnobody:nfsnobody /var/lib/jenkins
```
enable all the services related to NFS.
```console
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```
Open the exports file
```console
vim /etc/exports
Copy the below line to the end of the file, so that the NFS server knows whereabouts of the NFS clients.
/var/lib/jenkins      10.0.0.3(rw,sync,no_root_squash,no_all_squash)
/var/lib/jenkins      10.0.0.4(rw,sync,no_root_squash,no_all_squash)
```
restart the NFS server to apply your configuration.
```console
systemctl restart nfs-server
```
Apply the below commands to allow firewall.
```console
firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload
```

# Jenkins primary and secondary instances

To install the latest version of Jenkins master server on both the primary and secondary instances execute the below commands on both the machines.
```console
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

##### Make sure Java installed on the instance
```console
yum install jenkins
```
Before starting the Jenkins service, set up an NFS client on the machines to attach the NFS volumes.
```console
yum install nfs-utils
mkdir -p /mnt/nfs/var/lib/jenkins
nano /etc/fstab
10.0.0.2:/var/lib/jenkins    /mnt/nfs/var/lib/jenkins   nfs defaults 0 0
vim /etc/sysconfig/jenkins
JENKINS_HOME="/mnt/nfs/var/lib/jenkins"
#save
chown -R jenkins:jenkins /var/lib/jenkins/jobs
restart Jenkins service
systemctl restart jenkins
```

# HA proxy server
On the HA proxy instance execute the below commands for the installation.
```console
yum update 
yum install haproxy
```
Edit the HA proxy configuration in order to achieve active-passive routing of traffic for our High Available Jenkins master instance.
```console
vim /etc/haproxy/haproxy.cfg
```
Add below lines to the bottom of the haproxy.cfg" file
```console
frontend ft_jenkins
 bind *:80
 default_backend bk_jenkins
backend bk_jenkins
 server jenkins1 jenkins1_publicip:8080 check
 server jenkins2 jenkins2_publicip:8080 check backup
```
Restart server
```console
Restart the HA proxy server
systemctl restart haproxy
```
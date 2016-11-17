# create-ldap-KDC-server

this guide provide a step-by-step instruction on how to creare a openLDAP and MIT KDC environment for testing HDP security.

###Pre-install
install pre-requisits on this server. 
rng-tools help to generate radnom operation which is used by MIT KDC to initilize the database.

```
yum -y install wget ntp firewalld git httpd openssl rng-tools
ntpdate pool.ntp.org 
systemctl disable firewalld
systemctl stop firewalld
systemctl enable ntpd rngd
systemctl start ntpd rngd

setenforce 0

```

modify selinux config to disable it

```
vi /etc/selinux/config

```
Disable transparent huge page
```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
###Install MIT KDC server

First yum install MIT KDC and get you server name for krb5.conf
```
yum -y install krb5-server krb5-libs krb5-auth-dialog krb5-workstation
hostname -f
```
modify krb5.conf file so it contains you realm name as well as the server name 
```
vi /etc/krb5.conf
default_ccache_name = FILE:/tmp/krb5cc_%{uid}
```
```
vi /var/kerberos/krb5kdc/kdc.conf
```
Set the administrator for this KDC server
```
vi /var/kerberos/krb5kdc/kadm5.acl
```
create the database for KDC server
```
kdb5_util create -s -r FIELD.HORTONWORKS.COM

systemctl enable krb5kdc kadmin
systemctl start krb5kdc kadmin
systemctl status krb5kdc
systemctl status kadmin
```
Then add admin for KDC server and restart the server
```
kadmin.local 
addprinc admin/admin
addprinc rangeradmin
addprinc administrator
addprinc hadoopadmin
listprincs

systemctl restart krb5kdc kadmin
```

make sure you can run kadmin with created admin users 
```
kadmin -p admin/admin
kadmin -p rangeradmin
kadmin -p hadoopadmin
kadmin -p administrator
```

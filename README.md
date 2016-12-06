# create openLDAP KDC server
This guide is the first step of a serie to setup a secured HDP cluster with integrated Ranger and Atlas enabled. It gives you the ability to test all security and governance features in HDP stacks. 

The part provide a step-by-step instruction on how to creare a openLDAP, MIT KDC and MySQL (optional) environment on centOS 7 VM for testing HDP security and governance features. It assumes you have a centOS 7 VM with minimum of 4GB memory.

###Pre-install
First enable root user on centOS VM which is disabled by default. 
```
sudo vi /root/.ssh/authorized_keys
--make ssh key available, delete everything before ssh-rsa

sudo vi /etc/ssh/sshd_config
PermitRootLogin yes
```
Install pre-requisits on the server. rng-tools help to generate radnom operation which is used by MIT KDC to initilize the database.

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
--change the following line
SELINUX=disabled
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
```
Below is modfiied krb5.conf 
```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = FIELD.HORTONWORKS.COM
 default_ccache_name = FILE:/tmp/krb5cc_%{uid}

[realms]
 FIELD.HORTONWORKS.COM = {
  kdc = qwang-kdc-ldap.field.hortonworks.com
  admin_server = qwang-kdc-ldap.field.hortonworks.com
 }

[domain_realm]
 .field.hortonworks.com = FIELD.HORTONWORKS.COM
 field.hortonworks.com = FIELD.HORTONWORKS.COM
```
Next update kdc.conf
```
vi /var/kerberos/krb5kdc/kdc.conf
```
Here is modified kdc.conf 
```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 FIELD.HORTONWORKS.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```
Set the administrators in kadm5.acl 
```
vi /var/kerberos/krb5kdc/kadm5.acl
```
kadm5.acl after update
```
*/admin@FIELD.HORTONWORKS.COM   *
hadoopadmin@FIELD.HORTONWORKS.COM       *
rangeradmin@FIELD.HORTONWORKS.COM       *
```
Next create KDC database. This will take very long time without rng-tools. Make sure you note down the password 
```
kdb5_util create -s -r FIELD.HORTONWORKS.COM
```
Restart KDC server and kadmin and confirm both are running before proceed to next step
```
systemctl enable krb5kdc kadmin
systemctl start krb5kdc kadmin
systemctl status krb5kdc
systemctl status kadmin
```
Add principles for admin and test users to KDC. You must run this under root
```
kadmin.local 
```
Inside kadmin console, add admin principles and test users with default password as "passwords" (change this if you want to sue other password). You should see the newly created principles with listprincs
```
addprinc -pw password admin/admin
addprinc -pw password mktg1
addprinc -pw password mktg2
addprinc -pw password mktg3
addprinc -pw password hr1
addprinc -pw password hr2
addprinc -pw password hr3
addprinc -pw password legal1
addprinc -pw password legal2
addprinc -pw password legal3
addprinc -pw password finance1
addprinc -pw password finance2
addprinc -pw password finance3
addprinc -pw password sales1
addprinc -pw password sales2
addprinc -pw password sales3
addprinc -pw password ali
addprinc -pw password rangeradmin
addprinc -pw password xapolicymgr
addprinc -pw password hadoopadmin

listprincs
quit
```
then restart KDC and kadmin
```
systemctl restart krb5kdc kadmin
```
make sure you can run kadmin with created admin users 
```
kadmin -p admin/admin
kadmin -p rangeradmin
kadmin -p hadoopadmin
```
This concludes the installation of MIT KDC server.

###Install openLDAP server and create test users/groups
yum install openLDAP and supported components
```
yum -y install openldap-servers openldap-clients krb5-server-ldap epel-release

cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG 
chown ldap. /var/lib/ldap/DB_CONFIG 

systemctl enable slapd
systemctl start slapd 
```
Records the hashed password from the following command
```
slappasswd
note this down and use in later steps: {SSHA}E+Vgldng4xOrWfqfwOBAfdUSWkV/zWt3
```
create chrootpw.ldif with following content
```
dn: olcDatabase={0}config,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}E+Vgldng4xOrWfqfwOBAfdUSWkV/zWt3
```
And update the root password and load schemas
```
ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif 
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif 
```
Fill chdomain.ldif with follwing content 
```
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=admin,dc=field,dc=hortonworks,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=field,dc=hortonworks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=field,dc=hortonworks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}E+Vgldng4xOrWfqfwOBAfdUSWkV/zWt3

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=admin,dc=field,dc=hortonworks,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=admin,dc=field,dc=hortonworks,dc=com" write by * read
```
Then update LDAP domain
```
ldapmodify -Y EXTERNAL -H ldapi:/// -f chdomain.ldif 
```
Create domainanduser.ldif with following content
```
dn: dc=field,dc=hortonworks,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: Qi Wang

dn: cn=admin,dc=field,dc=hortonworks,dc=com
objectClass: organizationalRole
cn: admin
description: Directory Manager

dn: ou=Users,dc=field,dc=hortonworks,dc=com
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=field,dc=hortonworks,dc=com
objectClass: organizationalUnit
ou: Groups

dn: cn=marketing,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: marketing
objectclass: posixgroup
gidNumber: 75000001
memberuid: mktg1
memberuid: mktg2
memberuid: mktg3
memberuid: ali

dn: cn=hr,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: hr
objectclass: posixgroup
gidNumber: 75000002
memberuid: hr1
memberuid: hr2
memberuid: hr3
memberuid: ali

dn: cn=legal,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: legal
objectclass: posixgroup
gidNumber: 75000003
memberuid: legal1
memberuid: legal2
memberuid: legal3
memberuid: ali

dn: cn=finance,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: finance
objectclass: posixgroup
gidNumber: 75000004
memberuid: finance1
memberuid: finance2
memberuid: finance3
memberuid: ali

dn: cn=sales,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: sales
objectclass: posixgroup
gidNumber: 75000005
memberuid: sales1
memberuid: sales2
memberuid: sales3
memberuid: ali

dn: cn=admins,ou=Groups,dc=field,dc=hortonworks,dc=com
cn: admins
objectclass: posixgroup
gidNumber: 75000006
memberuid: rangeradmin
memberuid: xapolicymgr

dn: uid=ali,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: ali
sn: Bajwa
uid: ali
homedirectory:/home/ali
uidNumber: 75000010
gidNumber: 75000005
userPassword:password

dn: uid=mktg1,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: mktg1
sn: Smith
uid: mktg1
homedirectory:/home/mktg1
uidNumber: 75000011
gidNumber: 75000001
userPassword:password

dn: uid=mktg2,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: mktg2
sn: Smith
uid: mktg2
homedirectory:/home/mktg2
uidNumber: 75000012
gidNumber: 75000001
userPassword:password

dn: uid=mktg3,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: mktg3
sn: Smith
uid: mktg3
homedirectory:/home/mktg3
uidNumber: 75000013
gidNumber: 75000001
userPassword:password

dn: uid=hr1,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: hr1
sn: Smith
uid: hr1
homedirectory:/home/hr1
uidNumber: 75000014
gidNumber: 75000002
userPassword:password

dn: uid=hr2,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: hr2
sn: Smith
uid: hr2
homedirectory:/home/hr2
uidNumber: 75000015
gidNumber: 75000002
userPassword:password

dn: uid=hr3,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: hr3
sn: Smith
uid: hr3
homedirectory:/home/hr3
uidNumber: 75000016
gidNumber: 75000002
userPassword:password

dn: uid=legal1,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: legal1
sn: Smith
uid: legal1
homedirectory:/home/legal1
uidNumber: 75000017
gidNumber: 75000003
userPassword:password

dn: uid=legal2,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: legal2
sn: Smith
uid: legal2
homedirectory:/home/legal2
uidNumber: 75000018
gidNumber: 75000003
userPassword:password

dn: uid=legal3,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: legal3
sn: Smith
uid: legal3
homedirectory:/home/legal3
uidNumber: 75000019
gidNumber: 75000003
userPassword:password

dn: uid=finance1,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
uid: finance1
cn: finance1
sn: Smith
homedirectory:/home/finance1
uidNumber: 75000020
gidNumber: 75000004
userPassword:password

dn: uid=finance2,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: finance2
sn: Smith
uid: finance2
homedirectory:/home/finance2
uidNumber: 75000021
gidNumber: 75000004
userPassword:password

dn: uid=finance3,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: finance3
sn: Smith
uid: finance3
homedirectory:/home/finance3
uidNumber: 75000022
gidNumber: 75000004
userPassword:password

dn: uid=sales1,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: sales1
sn: Smith
uid: sales1
homedirectory:/home/sales1
uidNumber: 75000023
gidNumber: 75000005
userPassword:password

dn: uid=sales2,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: sales2
sn: Smith
uid: sales2
homedirectory:/home/sales2
uidNumber: 75000024
gidNumber: 75000005
userPassword:password

dn: uid=sales3,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: sales3
sn: Smith
uid: sales3
homedirectory:/home/sales3
uidNumber: 75000025
gidNumber: 75000005
userPassword:password

dn: uid=rangeradmin,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: Ranger
sn: Admin
uid: rangeradmin
homedirectory:/home/rangeradmin
uidNumber: 75000026
gidNumber: 75000006
userPassword:password

dn: uid=xapolicymgr,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: XAPolicy
sn: Manager
uid: xapolicymgr
homedirectory:/home/xapolicymgr
uidNumber: 75000027
gidNumber: 75000006
userPassword:password

dn: uid=hadoopadmin,ou=Users,dc=field,dc=hortonworks,dc=com
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
objectclass:posixaccount
cn: Ranger
sn: Admin
uid: hadoopadmin
homedirectory:/home/rangeradmin
uidNumber: 75000028
gidNumber: 75000006
userPassword:password
```
Create users and groups for testing
```
ldapadd -x -D cn=admin,dc=field,dc=hortonworks,dc=com -W -f domainanduser.ldif
```

Next we will enable ldaps://
```
openssl req -x509 -newkey rsa:4096 -keyout /etc/pki/tls/private/openldap.key -out /etc/pki/tls/certs/openldap.crt -days 3650 -nodes -subj "/CN=qwang-kdc-ldap.field.hortonworks.com"
cp /etc/pki/tls/private/openldap.key /etc/pki/tls/certs/openldap.crt /etc/pki/tls/certs/ca-bundle.crt /etc/openldap/certs/ 
chown ldap. /etc/openldap/certs/openldap.key /etc/openldap/certs/openldap.crt /etc/openldap/certs/ca-bundle.crt
```
create mod_ssl.ldif with following content
```
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ca-bundle.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/openldap.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/openldap.key
```
Then update cert locations
```
ldapmodify -Y EXTERNAL -H ldapi:/// -f mod_ssl.ldif 
```
Update /etc/sysconfig/slapd so ldaps:// is enabled
```
SLAPD_URLS="ldapi:/// ldap:/// ldaps:///"
```
restart sladp
```
systemctl restart slapd 
```

test with ldapserach
```
ldapsearch -W -D "cn=admin,dc=field,dc=hortonworks,dc=com" -b "dc=field,dc=hortonworks,dc=com"
```
About this LDAP directory
LDAP url
```
ldap://qwang-kdc-ldap.field.hortonworks.com:389
```
search base is
```
dc=field,dc=hortonworks,dc=com
```
All test users are inside this container and it could be used as user search base as well
```
ou=Users,dc=field,dc=hortonworks,dc=com
```
All groups are inside this container and it could be used as group search base as well
```
ou=Groups,dc=field,dc=hortonworks,dc=com
```
bind user/Manager DN
```
cn=admin,dc=field,dc=hortonworks,dc=com
```
This concludes the installation of openLDAP server

###Install MySQL server
Get MySQL repo and yum install MySQL
```
wget http://repo.mysql.com/mysql-community-release-el7.rpm
rpm -ivh mysql-community-release-el7.rpm
yum -y install mysql mysql-server mysql-libs mysql-connector-java*
systemctl enable mysqld.service
systemctl start mysqld.service
```
Login into MySQL (password is empty) and create schemas and users needed by HDP
```
mysql -u root -p

CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost';
CREATE USER 'ambari'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%';
FLUSH PRIVILEGES;
CREATE DATABASE ambari;

CREATE USER 'hive'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'localhost';
CREATE USER 'hive'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'hive'@'%';
FLUSH PRIVILEGES;
CREATE DATABASE hive;

CREATE USER 'oozie'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'oozie'@'%';
FLUSH PRIVILEGES;
CREATE DATABASE oozie;

CREATE USER 'rangerdba'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost';
CREATE USER 'rangerdba'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;

EXIT;

```
After Ambari installation, copy the schema over and apply to MySQL
```
mysql -u root -p ambari < /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql
```

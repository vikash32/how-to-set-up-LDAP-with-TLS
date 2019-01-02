# LDAP With TLS (Copy Paste Of Below Command is Suffice)
### Install openldap, openldap-servers and openldap-clients, here openldap-client is optional on ldap servers.
`sudo yum install -y openldap openldap-clients openldap-servers`

### Change to directory 
`cd /etc/openldap/`
### Generate password you want for LDAP Admin Account i am using "abc123"
```
sudo slappasswd
New password:
Re-enter new password:
shavalue: {SSHA}JzauI3NmCfK8moytqu9sOJaG7yljfGCl
```
### Change to config directory
`cd slapd.d/cn\=config`

### update olcRootPW with the password generated above; olcDatabase\=\{2\}hdb.ldif configuration file.
```
vi olcDatabase\=\{2\}hdb.ldif
olcRootPW: {SSHA}JzauI3NmCfK8moytqu9sOJaG7yljfGCl
```

### update dc value in file 'olcDatabase\=\{1\}monitor.ldif 'as per your org domain name , i am using "example" 
```
vi olcDatabase\=\{1\}monitor.ldif
al,cn=auth" read by dn.base="cn=Manager,dc=example,dc=com" read by * none
```
### Test your configuration by executing below command, ignore if there is checksum error is appearing.
`slaptest -u`

### Start your Ldap, Still we are not using TLS as of moment
```
systemctl start slapd
systemctl enable slapd
```
### Copy sample DB_CONFIG.example to DB_CONFIG ,it is important to create user,org etc
```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown ldap:ldap /var/lib/ldap/*
```
### now execute below command to add predefined schema
```
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```
### It is time to create LDAP Manager create a file "base.ldif" with below content
```
dn: dc=example,dc=com
dc: example
objectClass: top
objectClass: domain

dn: cn=ldapadm ,dc=example,dc=com
objectClass: organizationalRole
cn: Manager
description: LDAP Manager

dn: ou=Employees,dc=example,dc=com
objectClass: organizationalUnit
ou: Employees

dn: ou=Visitors,dc=example,dc=com
objectClass: organizationalUnit
ou: Visitors
```
### Now create LDAP Manager  
`ldapadd -x -W -D "cn=Manager,dc=example,dc=com" -f ./base.ldif`

#### Now create a user let say "vikash", i have created file "vikash.ldif" with below content
```
dn: uid=vikash,ou=Employees,dc=example,dc=com
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: shadowAccount
cn: vikash
uid: vikash
uidNumber: 1032
gidNumber: 1032
homeDirectory: /home/vikash
loginShell: /bin/bash
gecos: Vikash
userPassword: {crypt}x
shadowLastChange: 17058
shadowMin: 0
shadowMax: 99999
shadowWarning: 7
```
### Create user "vikash"
```
ldapadd -x -W -D "cn=Manager,dc=example,dc=com" -f ./vikash.ldif
ldappasswd -s vikash32# -W -D "cn=Manager,dc=example,dc=com" -x "uid=vikash,ou=Employees,dc=example,dc=com"
```

### You can verify that user is created or not by executing below command
`ldapsearch -x cn=vikash -b dc=example,dc=com`

### Till now our LDAP is not listening secure connection. 
#### To configure to use TLS , create a pem certificate, Please note: at the time of giving CN name, always provide complete domain name of ldap server ex: ldapserver1.example.com
```
cd /etc/openldap/
openssl req -new -x509 -nodes -out /etc/openldap/example.pem -keyout /etc/openldap/example.key -days 365
```

### Add below line in file slapd.d/cn\=config/olcDatabase\=\{2\}hdb.ldif 
```
vi olcDatabase\=\{2\}hdb.ldif
olcTLSCertificateFile: /etc/openldap/example.pem
olcTLSCertificateKeyFile: /etc/openldap/example.key
```

### Now open  ldap.conf and add below line
```
vi /etc/openldap/ldap.conf
TLS_CACERT /etc/openldap/example.pem
```
### Set up permission of key file
```
chown -R ldap. example.pem example.key
```
### Add ldaps:/// in /etc/sysconfig/slapd service file 
```
vi /etc/sysconfig/slapd
ldaps:///
```
### restart the service again
`systemctl restart slapd`
systemctl enable slapd



!--------- And that is all --------------!

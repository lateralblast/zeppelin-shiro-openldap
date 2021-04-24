![alt tag](https://raw.githubusercontent.com/lateralblast/zeppelin-shiro-openldap/master/cat_zeppelin.png)

Zeppelin Shiro OpenLDAP
=======================

Zeppelin authentication via OpenLDAP - Shiro configuration

Introduction
------------

The instructions for configuring OpenLDAP authentication for Zeppelin via Shiro are not very clear.
These instructions are designed as an attempt to make the process easier.

These instructions are for the default OOTB Zeppelin install using HTTP on port 8080.
You should enable and configure HTTPS. Details for doing this are not included here.

Similarly these instructions are for connecting to OpenLDAP in simple mode on port 389.
You should configure LDAP and SSL. Details for this are not included here.

One of the benefits of the LDAP authentication over PAM is that you can remap the group names.

If users don't need system access or to upload files manually for processing,
then you could set the user shell to /bin/false to restrict access file read.

Requirements
------------

Software:

- Zeppelin
- OpenLDAP

Configuration:

- A valid hostname and FQDN
- A group for Zeppelin users
  - a group name e.g. zepusers with a group id (GID) e.g. 2001
  - Preferably same group name for LDAP and roles in Zeppelin/Shiro (shiro.ini)
    - Alternate group names can be used
- A group for Zeppelin admins
  - a group name e.g. zepadmins with a group id (GID) e.g. 2002
  - Preferable same group name for LDAP and roles in Zeppelin/Shiro (shiro.ini)
    - Alternate group names can be used
- Zeppelin users and ids
  - A user name e.g. zepuser with a user id (UID) e.g. 2001
- Zeppelin admins and ids
  - A user name e.g. zepadmin with a user id (UID) e.g. 2002

Instructions
------------

Set hostname:

```
$ sudo hostnamectl ubuntu.home.arpa
```

Create users and groups if users need the ability to manually upload files:

```
$ sudo groupadd zepusers
$ sudo groupadd zepadmins
$ sudo useradd -s /bin/bash -d /home/zepserver -m -G user zepserver
$ sudo useradd -s /bin/bash -d /home/zepuser -m -G user zepusers
$ sudo useradd -s /bin/bash -d /home/zepadmin -m -G admin zepadmins
$ sudo passwd zepserver
$ sudo passwd zepuser
$ sudo passwd zepadmin
```

Example group file entries:

```
user:x:2001:zepuser
admin:x:2002:zepadmin
zepuser:x:1001:
zepadmin:x:1002:
```

Example passwd file entries:

```
zepuser:x:1001:1001::/home/zepuser:/bin/bash
zepadmin:x:1002:1002::/home/zepadmin:/bin/bash
```

Install OpenLDAP:

```
$ sudo apt update
$ sudo apt -y install slapd ldap-utils
```

Create a file with base DN entries for groups:

```
$ cat basedn.ldif
dn: ou=zepusers,dc=home,dc=arpa
objectClass: organizationalUnit
ou: zepusers

dn: ou=zepgroups,dc=home,dc=arpa
objectClass: organizationalUnit
ou: zepgroups
```

Import entries for groups:

```
$ ldapadd -x -D cn=admin,dc=home,dc=arpa -W -f basedn.ldif
```

Create passwords for user and admin account(s):

```
$ slappasswd
New password: 
Re-enter new password: 
{SSHA}Cz/Dr9UpTJIkRmKokr5kpQLMmwhjxlLe
```

Create a file with entries for users and admins:

```
$ cat ldapusers.ldif
dn: uid=zepuser,ou=zepusers,dc=home,dc=arpa
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: zepuser
sn: ZepUser
userPassword: {SSHA}Cz/Dr9UpTJIkRmKokr5kpQLMmwhjxlLe
loginShell: /bin/bash
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/zepuser

dn: uid=zepadmin,ou=zepusers,dc=home,dc=arpa
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: zepadmin
sn: ZepAdmin
userPassword: {SSHA}Cz/Dr9UpTJIkRmKokr5kpQLMmwhjxlLe
loginShell: /bin/bash
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/zepadmin
```

Import entries for users:

```
$ ldapadd -x -D cn=admin,dc=home,dc=arpa -W -f ldapusers.ldif
```

Create group membership entries:

```
$ cat ldapgroups.ldif
dn: cn=zepusers,ou=zepgroups,dc=home,dc=arpa
objectClass: posixGroup
cn: zepusers
gidNumber: 2001
memberUid: zepuser

dn: cn=zepusers,ou=zepgroups,dc=home,dc=arpa
objectClass: posixGroup
cn: zepadmins
gidNumber: 2002
memberUid: zepadmin
```

Import group entries for users:

```
$ ldapadd -x -D cn=admin,dc=home,dc=arpa -W -f ldapgroups.ldif
```


Install Zeppelin:

```
$ wget https://downloads.apache.org/zeppelin/zeppelin-0.9.0/zeppelin-0.9.0-bin-all.tgz 
$ tar -xpf zeppelin-0.9.0-bin-all.tgz
```

Copy shiro config template and edit it:

```
$ cp ./zeppelin-0.9.0-bin-all/conf/shiro.ini.template ./zeppelin-0.9.0-bin-all/conf/shiro.ini
$ vi ./zeppelin-0.9.0-bin-all/conf/shiro.ini
```

Comment out user entries in shiro.ini, e.g.:

```
[users]
# List of users with their password allowed to access Zeppelin.
# To use a different strategy (LDAP / Database / ...) check the shiro doc at http://shiro.apache.org/configuration.html#Configuration-INISections
# To enable admin user, uncomment the following line and set an appropriate password.
#admin = password1, admin
#user1 = password2, role1, role2
#user2 = password3, role3
#user3 = password4, role2
```

Add LDAP entries to main section in shiro.ini, e.g.:

```
[main]
ldapRealm = org.apache.zeppelin.realm.LdapRealm
ldapRealm.userDnTemplate = uid={0},ou=zepusers,dc=home,dc=arpa
ldapRealm.searchBase = dc=home,dc=arpa
ldapRealm.userSearchBase = ou=zepusers,dc=home,dc=arpa
ldapRealm.groupSearchBase = ou=zepgroups,dc=home,dc=arpa
ldapRealm.contextFactory.url = ldap://127.0.0.1:389
ldapRealm.contextFactory.authenticationMechanism = simple
ldapRealm.userObjectClass = posixAccount
ldapRealm.groupObjectClass = posixGroup
ldapRealm.authorizationEnabled = true
ldapRealm.memberAttribute = memberUid
ldapRealm.memberAttributeValueTemplate=uid={0},ou=zepusers,dc=home,dc=arpa
ldapRealm.rolesByGroup = zepadmins:admin,zepusers:user
ldapRealm.allowedRolesForAuthentication = admin,user
ldapRealm.userSearchAttributeName = uid
securityManager.realms = $ldapRealm
```

Unlike PAM, with LDAP you can remap the LDAP group names to the Zeppelin ones using ldapRealm.rolesByGroup:

```
ldapRealm.rolesByGroup = zepadmins:admin,zepusers:user
```

Have admin entry in roles section in shiro.ini, e.g.:

```
[roles]
admin = *
```

Make sure you include groups in roles in urls section in shiro.ini, e.g.:

```
[urls]
/api/version = anon
/api/cluster/address = anon
# Allow all authenticated users to restart interpreters on a notebook page.
# Comment out the following line if you would like to authorize only admin users to restart interpreters.
#/api/interpreter/setting/restart/** = authc
/api/interpreter/** = authc, roles[zepadmins]
/api/notebook-repositories/** = authc, roles[zepadmins]
/api/configurations/** = authc, roles[zepadmins]
/api/credential/** = authc, roles[zepadmins]
/api/admin/** = authc, roles[zepadmins]
#/** = anon
/** = authc
```

The authc entry above allows anyone with a valid user account and password on the system to have user access via PAM.
The roles entry allows only people in the zepadmins group to have administrative access to administrative URLS.

License
-------

This documentation is licensed as CC-BA (Creative Commons By Attrbution)

http://creativecommons.org/licenses/by/4.0/legalcode

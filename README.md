# OpenLDAP Setup


My process for setting up an openLDAP server on a linux node for testing. I utilize the LDIF files from my [SokoviaProject](https://github.com/Twizzlerific/sokoviaProject) repository to populate the users and groups. This is based on Marvel characters so my admin user is named [oneaboveall](https://marvel.fandom.com/wiki/One_Above_All_(Multiverse)). 

---

<!-- toc -->

- [Install and start](#Install-and-start)
- [Configure LDAP DB](#Configure-LDAP-DB)
- [Configure LDAP Domain](#Configure-LDAP-Domain)
- [Setting up KDC](#Setting-up-KDC)

<!-- tocstop -->

---

# Install and start
Installing packages:
```
$ yum install openldap openldap-servers openldap-clients sudo -y
```

Starting the LDAP Server
```
$ sudo systemctl start slapd
$ sudo systemctl enable slapd
$ sudo systemctl status slapd
```

Creating the Admin Password
```
$ slappasswd yourdesiredpassword
>> {SHA}someencodedbitshere/V91
```


Applying the Admin Password
```
# vi ldaprootpasswd.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SHA}someencodedbitshere/V91
```

```
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f ldaprootpasswd.ldif
```

# Configure LDAP DB

Copy the template files
```
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
chown -R ldap:ldap /var/lib/ldap/DB_CONFIG
sudo systemctl restart slapd
```

Import Schemas
```
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif 
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```


Load memberOf module
```
vi memberOfLoad.ldif
dn: cn=module,cn=config
cn: module 
objectClass: olcModuleList
olcModulePath: /usr/lib64/openldap
olcModuleLoad: memberof.la
```
```
ldapadd -Y EXTERNAL -H ldapi:/// -f memberOfLoad.ldif
```

Apply memberOf overlay
```
vi memberOfOverldap.ldif
dn: olcOverlay=memberof,olcDatabase={1}hdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcMemberOf
olcOverlay: memberof
olcMemberOfRefint: TRUE
```

```
ldapadd -Y EXTERNAL -H ldapi:/// -f memberOfOverldap.ldif
```


# Configure LDAP Domain
Here we will start customizing our LDAP server by defining our domain and admin user within that domain. 

In my example, I want the domain to be akin to "KS.COM" so I set it to _dc=ks,dc=com_

```
# vi ldapdomain.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=oneaboveall,dc=ks,dc=com" read by * none

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=oneaboveall,dc=ks,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SHA}someencodedbitshere/V91

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=oneaboveall,dc=ks,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=oneaboveall,dc=ks,dc=com" write by * read
```

```
ldapmodify -Y EXTERNAL -H ldapi:/// -f ldapdomain.ldif
```


Creating base entries

```
# vi baseldapdomain.ldif
dn: dc=ks,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: ks com
dc: ks

dn: cn=oneaboveall,dc=ks,dc=com
objectClass: organizationalRole
cn: oneaboveall
description: The One Above All

dn: ou=Users,dc=ks,dc=com
objectClass: organizationalUnit
ou: Users

dn: ou=Marvel,ou=Users,dc=ks,dc=com
objectClass: organizationalUnit
ou: Users

dn: ou=Groups,dc=ks,dc=com
objectClass: organizationalUnit
ou: Groups
```

```
ldapadd -x -D cn=oneaboveall,dc=ks,dc=com -W -f baseldapdomain.ldif
Enter LDAP Password:
```

Import group entries:
```
ldapadd -x -D cn=oneaboveall,dc=ks,dc=com -W -f groups.ldif
Enter LDAP Password:
```

Import User entries with hashed password 'marvel':
```
ldapadd -x -D cn=oneaboveall,dc=ks,dc=com -W -f groups.ldif
Enter LDAP Password:
```

---

If we hit invalid password error we can update the password for the admin user:

```
dn: olcDatabase={0}config,cn=config
#olcRootDN: cn=oneaboveall,dc=ks,dc=com
changetype: modify
replace: olcRootPW
olcRootPW: {SHA}someencodedbitshere/V91
```

If we hit any errors with the user passwords, we can put the password in clear text or replace the previous string using vi and a regex making sure to escape special characters:

```
:%s/\{SHA\}Q\+Z6wjVspBWOdyOBF4Re5Yp\/0NI\=/YOUR_NEW_PASSWORD/g
```
---
__DISCLAIMER__:

Characters, names, and such are all owned by Marvel Comics. This project is solely intended for educational purposes. 

---
DiBtP ("Done is Better than Perfect!")

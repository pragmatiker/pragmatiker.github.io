---
layout: post
title: "Running OpenLDAP with docker"
date: 2024-07-13 12:00:00 -0000
author: "TimDude"
categories:
  - IT Sec 
  - ssh certificates
tags:
  - docker
  - keycloak
  - openldap
---

Not really necessary, since Keycloak brings its own User Management, still I want to build an external User Database in LDAP.
Since most Companies will have Active Directory / LDAP, I wanted to include something similar in this lab.

## Add OpenLDAP to docker-compose of the Lab
Edit /opt/docker/ssh_poc/docker-compose.yml
~~~
version: '3.8'

services:
  openldap:
    image: osixia/openldap
    container_name: openldap
    ports:
      - "389:389"
    volumes:
      - openldap_config:/etc/ldap/slapd.d
      - openldap_db:/var/lib/ldap
    environment:
      LDAP_ADMIN_PASSWORD: admin

volumes:
  openldap_config
  openldap_db
~~~

## Start the Lab with systemcl
Run the container
~~~
systemctl restart docker-compose-ssh_poc.service
~~~

## Feeding initial Data into the Directory

If you can install ldap-utils on the host OS, you dont need to jump into the container.

Jump to container shell
~~~
docker exec -it openldap /bin/bash ;
~~~

From container shell:
~~~
# // on LDAP docker:
printf "%s" """\
dn: ou=groups,dc=example,dc=org
objectClass: organizationalunit
objectClass: top
ou: groups
description: groups of users

dn: ou=users,dc=example,dc=org
objectClass: organizationalunit
objectClass: top
ou: users
description: users

dn: cn=dev,ou=groups,dc=example,dc=org
objectClass: groupofnames
objectClass: top
description: testing group for dev
cn: dev
member: uid=lsmith,ou=users,dc=example,dc=org
member: uid=tlaxman,ou=users,dc=example,dc=org
member: uid=lwilson,ou=users,dc=example,dc=org

dn: cn=prod,ou=groups,dc=example,dc=org
objectClass: groupofnames
objectClass: top
description: testing group for prod
cn: prod
member: uid=ataylor,ou=users,dc=example,dc=org
member: uid=tbrown,ou=users,dc=example,dc=org
member: uid=devans,ou=users,dc=example,dc=org

dn: uid=tlaxman,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: tlaxman
givenName: Tim
sn: Laxman
cn: Tim Laxman
mail: tim.laxman@example.org
userPassword: tim2024

dn: uid=lsmith,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: lsmith
givenName: Laura
sn: Smith
cn: Laura Smith
mail: laura.smith@example.org
userPassword: laura1204

dn: uid=lwilson,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: lwilson
givenName: Luke
sn: Wilson
cn: Luke Wilson
mail: luke.wilson@example.org
userPassword: mrwilson1257

dn: uid=ataylor,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: ataylor
givenName: Ashley
sn: Taylor
cn: Ashley Taylor
mail: ashley.taylor@example.org
userPassword: asht1034

dn: uid=tbrown,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: tbrown
givenName: Thomas
sn: Brown
cn: Thomas Brown
mail: thomas.brown@example.org
userPassword: ThoBro123

dn: uid=devans,ou=users,dc=example,dc=org
objectClass: inetOrgPerson
objectClass: organizationalPerson
objectClass: person
objectClass: top
uid: devans
givenName: Dave
sn: Evnas
cn: Dave Evans
mail: dave.evans@example.org
userPassword: DopeDave
""" > ldap_seed.ldif ; # // an example ldif with changes to apply
~~~

Send ldif to server
~~~
ldapadd -x -H ldap://localhost -W -D "cn=admin,dc=example,dc=org" -f ldap_seed.ldif
~~~

See if you can search for the new entries
~~~
ldapsearch -x -H ldap://localhost -W -D "cn=admin,dc=example,dc=org" -b "dc=example,dc=org"
~~~

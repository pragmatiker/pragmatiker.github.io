---
layout: post
title: "Add LDAP user federation to Keycloak"
date: 2024-07-22 12:00:00 -0000
author: "TimDude"
categories: ssh_poc  keycloak openldap
tags: Keycloak LDAP federation openldap
---

It is time to set up a Backend were we can get ans store users. This is not strictly necessary.
However there are some interessting things to learn on the way, so I think it is worth taking a dive into the matter.

## First we add a new realm
1. Login to the admin web console
2. Click the Realm Drop Down
3. Create Realm "LAB"
4. Save

## Add the Federation
1. Go to the side Menue on the top left, select Realm "LAB"
2. Click on "User federation" in the side menue
3. Add new Provider "LDAP"
4. Edit the Information provided below
5. Click SAVE
6. Select the "OpenLDAP federation"
7. Top right "action" drop down, select "sync all users"

You should now have 6 users synced
   
~~~
UI display name: OpenLDAP
Vendor: other
Connection URL: ldap://openldap
Bind DN: cn=admin,dc=example,dc=org
Bind Credentials: #the admin password

Edit mode: WRITEABLE
Users DN: ou=users,dc=example,dc=org
Username LDAP attribute: uid
RDN LDAP attribute: uid
UUID LDAP attribute: entryUUID
User object classes: top, person,organizationalPerson,inetOrgPerson
~~~

## Add a Mapping for the Groups
1. Click on User federation
2. Selct the "OpenLDAP" federation
3. Select the "Mappings" tab at the Top
4. Click "Add Mapper"
5. Edit in, the informatio provded below
6. Click "Save"
7. Select "OpenLDAP Groups" Mapper
8. Top right "action" drop down, select "Sync LDAP groups to Keycloak"

You should now have 2 groups synced.

~~~
Name: OpenLDAP Groups
Mapper Type: group-ldap-mapper
LDAP Groups DN: ou=groups,dc=example,dc=org
Group Object Classes: groupOfNames
Membership LDAP Attribute: member
Membership Attribute Type: DN
Membership User LDAP Attribute: uid
User Groups Retrieve Strategy: LOAD_GROUPS_BY MEMBER_ATTRIBUTE
~~~

## Add a Mapping to get fullname from LDAP CN
1. Selct the "OpenLDAP" federation
2. Select the "Mappings" tab at the Top
3. Click "Add Mapper"
4. Edit in, the information provided below
5. Click "Save"
   
~~~
Name: fullNametoCN
Mapper type: full-name-ldap-mapper
LDAP Full Name Attribute: cn
~~~

## Edit User Attribute mappings
For each of the following mappers, click the name, and set the "Read Only" flag to "Off" (this enables 2-way sync between Keycloak and OpenLDAP). Also check, that the "User Model Attribute" is mapped to the right "LDAP Attribute"

| Mapping | User Model Attribute| LDAP Attribute|
|-------|------|------|
| email | email | mail |
| first name | firstName | givenName |
| last name | firstName | sn |
| Username | username | uid |


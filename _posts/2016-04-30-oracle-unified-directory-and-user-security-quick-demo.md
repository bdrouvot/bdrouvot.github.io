---
layout: post
title: 'Oracle Unified Directory and user security: quick demo'
date: 2016-04-30 15:48:05.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Other
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _oembed_104d5c4092662d801721d6766f1ccbea: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="640" height="400" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_104d5c4092662d801721d6766f1ccbea: '1462023876'
  _publicize_done_2435436: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/YonAxgRtW1B
  _publicize_job_id: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6132188588219129856&type=U&a=iGVx
  _oembed_1f8e0fbeac568055486402514995d3e7: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="634" height="396" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _wpas_done_8482624: '1'
  _wpas_done_2077996: '1'
  _oembed_a88f3cb477d1da6d3e8eed3a9dfc7075: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="500" height="313" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_a88f3cb477d1da6d3e8eed3a9dfc7075: '1462027690'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/726422903026122752";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _oembed_time_1f8e0fbeac568055486402514995d3e7: '1462027693'
  _publicize_done_8471251: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _oembed_96a8a723aa36b1fac29152591a0788c0: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="840" height="525" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_96a8a723aa36b1fac29152591a0788c0: '1495265933'
  _oembed_ffde1fc011d07a31f249afd695de1bef: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="1100" height="688" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_ffde1fc011d07a31f249afd695de1bef: '1499532286'
  _oembed_80772b5708688a0a9ffd57f81b6d3a37: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="656" height="410" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_80772b5708688a0a9ffd57f81b6d3a37: '1499532340'
  _oembed_4be389408d98a15394bfb97acb00ceb8: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/164819949"
    width="739" height="462" frameborder="0" title="oud" webkitallowfullscreen mozallowfullscreen
    allowfullscreen></iframe></div>
  _oembed_time_4be389408d98a15394bfb97acb00ceb8: '1500799786'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/04/30/oracle-unified-directory-and-user-security-quick-demo/"
---

Introduction
------------

Oracle Unified Directory (OUD) can be used to centrally manage database users across the enterprise.

It allows us to manage roles and privileges across various databases registered with the directory.

Users connect to the database by providing credentials that are stored in Oracle Unified Directory, then the database executes LDAP search operations to query user specific authentication and authorization information.

This post does not cover the OUD installation.

Setup Steps
-----------

So, once the OUD has been installed the remaining steps are:

1.  Register the database into the OUD.
2.  Create global roles and global users into the database.
3.  Create groups and users into the OUD.
4.  Link OUD groups with databases roles.

Let's setup.

### Step 1: Register the database into the OUD

    >$ cat $ORACLE_HOME/network/admin/ldap.ora
    DIRECTORY_SERVERS=(oud:1389:1636)
    DEFAULT_ADMIN_CONTEXT="dc=bdt,dc=com"
    DIRECTORY_SERVER_TYPE=OID

    >$ cat register_database_oud.ksh
    dbca -silent -configureDatabase -sourceDB PBDT -registerWithDirService true -dirServiceUserName "cn=orcladmin" -dirServicePassword "bdtbdt" -walletPassword "monster123#"

    >$ ksh ./register_database_oud.ksh
    Preparing to Configure Database
    6% complete
    13% complete
    66% complete
    Completing Database Configuration
    100% complete
    Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/PBDT/PBDT20.log" for further details.

### Step 2: Create global roles and global users into the database

    SQL> !cat prepare_oud_users.sql
    create role org_dba identified globally;
    grant dba to org_dba;
    create role org_connect identified globally;
    grant create session to org_connect;
    create user org_user identified globally;

    SQL> @prepare_oud_users.sql

    Role created.


    Grant succeeded.


    Role created.


    Grant succeeded.


    User created.

As you can see:

-   DBA has been granted to the ORG\_DBA role.
-   CREATE SESSION has been granted to the ORG\_CONNECT role.
-   The user ORG\_USER has not been granted any privileges.

### Step 3: Create groups and users into the OUD

As an example, a ldif file has been created to:

-   create 2 users: bdt\_dba and bdt\_connect.
-   create 2 groups: DBA\_GROUP and CONNECT\_GROUP.
-   assign bdt\_dba to the DBA\_GROUP and bdt\_connect to the CONNECT\_GROUP.

<!-- -->

    >$ cat newgroupsusers.ldif
    dn: cn=groups,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: groupOfNames
    cn: groups

    dn: cn=users,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: groupOfNames
    cn: users

    dn: cn=bdt_connect,cn=users,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: person
    objectclass: organizationalPerson
    objectclass: inetOrgPerson
    cn: bdt_connect
    sn: bdt_connect
    uid: bdt_connect
    userpassword: bdtconnect

    dn: cn=CONNECT_GROUP,cn=groups,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: groupOfNames
    cn: CONNECT_GROUP
    member: cn=bdt_connect,cn=users,dc=bdt,dc=com

    dn: cn=bdt_dba,cn=users,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: person
    objectclass: organizationalPerson
    objectclass: inetOrgPerson
    cn: bdt_dba
    sn: bdt_dba
    uid: bdt_dba
    userpassword: bdtdba

    dn: cn=DBA_GROUP,cn=groups,dc=bdt,dc=com
    changetype: add
    objectclass: top
    objectclass: groupOfNames
    cn: DBA_GROUP
    member: cn=bdt_dba,cn=users,dc=bdt,dc=com

and then launch:

    $> cat create_ldap_groups_users.ksh
    ldapadd -h oud -p 1389 -D "cn=orcladmin" -w bdtbdt -f ./newgroupsusers.ldif -v

    $> ksh ./create_ldap_groups_users.ksh

So that the users and groups have been created into the OUD.

A graphical view of what has been done into the OUD (thanks to the [Apache Directory Studio](https://directory.apache.org/studio/)) is:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2016-04-30-at-13-48-21.png" class="aligncenter size-full wp-image-3065" width="640" height="294" alt="Screen Shot 2016-04-30 at 13.48.21" />

### Step 4: Link OUD groups with database roles.

    >$ ksh ./mapdb_ldap.ksh
    + echo 'Mapping User'
    Mapping User
    + eusm createMapping database_name=PBDT realm_dn='dc=bdt,dc=com' map_type=SUBTREE map_dn='cn=users,dc=bdt,dc=com' schema=ORG_USER ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + echo 'Create Enterprise role'
    Create Enterprise role
    + eusm createRole enterprise_role=PBDT_dba_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + eusm createRole enterprise_role=PBDT_connect_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + echo 'Link Roles'
    Link Roles
    + eusm addGlobalRole enterprise_role=PBDT_dba_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' database_name=PBDT global_role=ORG_DBA dbuser=system dbuser_password=bdtbdt dbconnect_string=dprima:1521:PBDT ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + eusm addGlobalRole enterprise_role=PBDT_connect_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' database_name=PBDT global_role=ORG_CONNECT dbuser=system dbuser_password=bdtbdt dbconnect_string=dprima:1521:PBDT ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + echo 'Grant Roles'
    Grant Roles
    + eusm grantRole enterprise_role=PBDT_dba_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' group_dn='cn=DBA_GROUP,cn=groups,dc=bdt,dc=com' ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt
    + eusm grantRole enterprise_role=PBDT_connect_role domain_name=OracleDefaultDomain realm_dn='dc=bdt,dc=com' group_dn='cn=CONNECT_GROUP,cn=groups,dc=bdt,dc=com' ldap_host=oud ldap_port=1389 ldap_user_dn='cn=orcladmin' ldap_user_password=bdtbdt

So that, for this database only, there is a mapping between:

-   The database ORG\_DBA role (created in step 2) and the OUD DBA\_GROUP group (created in step 3).
-   The database ORG\_CONNECT role (created in step 2) and the OUD CONNECT\_GROUP group (created in step 3).

Authentication and authorization results:
-----------------------------------------

You can view the result into this video:

\[embed\]https://vimeo.com/164819949\[/embed\]

As you can see:

-   There is no bdt\_dba nor bdt\_connect oracle users into the database.
-   I logged in with the bdt\_dba OUD user, then was connected as ORG\_USER into the database and have been able to query the dba\_users view.
-   I logged in with the bdt\_connect OUD user, then was connected as ORG\_USER into the database and (as expected) have not been able to query the dba\_users view due to the lack of permission.

Remark
------

Frank Van Bortel already covered this subject into this [blog post](http://vanbortel.blogspot.fr/2013/06/oracle-unified-directory-111210-tns-and.html).

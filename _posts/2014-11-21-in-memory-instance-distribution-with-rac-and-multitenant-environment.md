---
layout: post
title: In-Memory Instance distribution with RAC and Multitenant Environment
date: 2014-11-21 13:46:40.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- In-Memory
- Rac
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/hr91Dnqg2hw
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/9SbFyYQG4Q
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5941542018600701952&type=U&a=IZJo
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/"
---

In a previous post (see [In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data](http://bdrouvot.wordpress.com/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/ "In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data")) I provided a way to get rid of the default In-Memory Instance distribution on a RAC (non CDB) database.

**<span style="text-decoration:underline;">To summarize:</span>**

-   Thanks to the hidden parameter “*\_inmemory\_auto\_distribute*” we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.
-   By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
-   With “***\_inmemory\_auto\_distribute***” set to **false**, then we are able to populate one instance (or all the instances) **with all the data**.

Now, let's have a look at the In-Memory distribution on a **Multitenant RAC database** and let's check **if we can influence (get rid of the default behaviour)** how the In-Memory is distributed across all the Instances per PDB.

<span style="text-decoration:underline;">**By default the IMCS distribution for a PDB is the following:**</span>

<span style="text-decoration:underline;">**First case:**</span> The PDB is open **on one** Instance only:

    SYS@CDB$ROOT> alter pluggable database BDTPDB open instances=('BDT12c3_1');

    Pluggable database altered.

    -- Now connect to the BDTPDB PDB as bdt/bdt
    SYS@CDB$ROOT> @connect_user_pdb.sql
    Enter value for user: bdt
    Enter value for pwd: bdt
    Enter value for pdb: BDTPDB
    Connected.

    -- Let's enable the inmemory attribute on the BDT table
    BDT@BDTPDB> alter table bdt inmemory ;

    Table altered.

    -- Trigger the IMCS population
    BDT@BDTPDB> select count(*) from bdt;

      COUNT(*)
    ----------
      44591000

    -- check the distribution
    BDT@BDTPDB> SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
    FROM gv$im_user_segments s, v$pdbs c
    WHERE s.segment_name = 'BDT'
    and c.con_id=s.con_id; 

       INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- ------------------------------ --------- ------------- ---------- -------------------
             1 BDTPDB                         COMPLETED     695074816 1879048192                   0

As you can see **the Instance 1 contains all the data** for this PDB (as BYTES\_NOT\_POPULATED=0)

<span style="text-decoration:underline;">**Second case:**</span> The PDB is open **on all** the Instances:

    -- Open the BDTPDB PDB on all instances
    SYS@CDB$ROOT> alter pluggable database BDTPDB open instances=all;

    Pluggable database altered.

    -- Now connect to the BDTPDB PDB as bdt/bdt
    SYS@CDB$ROOT> @connect_user_pdb.sql
    Enter value for user: bdt
    Enter value for pwd: bdt
    Enter value for pdb: BDTPDB
    Connected.  

    -- Let's enable the inmemory attribute on the BDT table
    BDT@BDTPDB> alter table bdt inmemory ;

    Table altered.

    -- Trigger the IMCS population
    BDT@BDTPDB> select count(*) from bdt;

      COUNT(*)  
    ----------  
      44591000

    -- check the distribution
    BDT@BDTPDB> SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
    FROM gv$im_user_segments s, v$pdbs c
    WHERE s.segment_name = 'BDT'
    and c.con_id=s.con_id;


       INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- ------------------------------ --------- ------------- ---------- -------------------
             1 BDTPDB                         COMPLETED     376307712 1879048192           802889728
             2 BDTPDB                         COMPLETED     298909696 1879048192          1006559232

We can see that (for this PDB), by default, **no instance contains all data** as the BYTES\_NOT\_POPULATED column is  greater than 0.

Now let’s set the hidden parameter **“*\_inmemory\_auto\_distribute*” to false** **(at the PDB level)**:

    BDT@BDTPDB> alter system set "_inmemory_auto_distribute"=false;
    alter system set "_inmemory_auto_distribute"=false
    *
    ERROR at line 1:
    ORA-65040: operation not allowed from within a pluggable database

So, we **can't set this hidden** parameter at the PDB level.

Ok, let's set it at the **CDB level**:

    SYS@CDB$ROOT> alter system set "_inmemory_auto_distribute"=false;

    System altered.

then re-trigger the population on instance 1 (for this PDB) and check the distribution:

    SYS@CDB$ROOT> @connect_user_pdb.sql
    Enter value for user: bdt
    Enter value for pwd: bdt
    Enter value for pdb: BDTPDB
    Connected.  
    BDT@BDTPDB> select count(*) from bdt;

      COUNT(*)  
    ----------  
      44591000

    BDT@BDTPDB> SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
    FROM gv$im_user_segments s, v$pdbs c
    WHERE s.segment_name = 'BDT'
    and c.con_id=s.con_id;

       INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- ------------------------------ --------- ------------- ---------- -------------------
             2 BDTPDB                         COMPLETED     297861120 1879048192          1006559232
             1 BDTPDB                         COMPLETED     675151872 1879048192                   0

**BINGO!** Look at the Instance 1, **it now contains all the data** in its IMCS (as BYTES\_NOT\_POPULATED=0) for this PDB.

If I connect on the Instance 2 (for this PDB) and launch the query to trigger the population, I’ll get:

       INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- ------------------------------ --------- ------------- ---------- -------------------
             1 BDTPDB                         COMPLETED     675151872 1879048192                   0
             2 BDTPDB                         COMPLETED     672006144 1879048192                   0

So, **both instances now contain all the data for this PDB.  
**

<span style="text-decoration:underline;">**Remarks:**</span>

-   As I said into the previous post: I used an hidden parameter, so you should get oracle support approval to use it.
-   I guess (because I can't test) that in a Multitenant and Engineered Systems context it is also possible to have all the data appear in the IM column store on every node for a PDB (thanks to the *duplicate* attribute): As Christian Antognini show us into [this blog post](http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/) for non CDB.

<span style="text-decoration:underline;">**Conclusion:**</span>

The conclusion is the same than in [my previous post](http://bdrouvot.wordpress.com/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/ "In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data") (with non CDB):

-   Thanks to the hidden parameter “*\_inmemory\_auto\_distribute*” we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.
-   By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
-   With “***\_inmemory\_auto\_distribute***” set to **false**, then we are able to populate one instance (or all the instances) **with all the data**.

<span style="text-decoration:underline;">And we can add that:</span>

-   If the PDB is open on one instance only then this instance contains all the data.
-   If the PDB is open on all the instances then we have to set “***\_inmemory\_auto\_distribute***” to **false to have at least one instance that contains all the data.**
-   Each PDB population **will behave the same** (distributed or not) and inherit from the CDB setting (As we **can not set the hidden** parameter at the **PDB** level).

 

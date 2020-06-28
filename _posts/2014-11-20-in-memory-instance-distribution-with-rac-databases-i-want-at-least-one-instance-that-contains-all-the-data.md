---
layout: post
title: 'In-Memory Instance distribution with RAC databases: I want at least one instance
  that contains all the data'
date: 2014-11-20 11:49:54.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- In-Memory
- Rac
tags: [Oracle RAC, oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/Q3GYg4SvAAX
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/8HAkr867Bm
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5941150260284923904&type=U&a=G4A4
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _wpas_skip_tumblr: '1'
  _wpas_skip_path: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/"
---

As you may know, with a RAC database, **by default all objects populated into**  
**In-memory will be distributed across all of the IM column stores in the cluster**. It is also possible to have all the data appear in the IM column store on every node (Engineered Systems only). See this [white paper](http://www.oracle.com/technetwork/database/in-memory/overview/twp-oracle-database-in-memory-2245633.html) for more details.

<span style="text-decoration:underline;">**There is 2 very interesting blog post around this subject:**</span>

1.  Kerry Osborne show us **how we can distribute the data** (using the *distribute* INMEMORY attribute) across the nodes into this [blog post](http://kerryosborne.oracle-guy.com/2014/09/12c-in-memory-on-rac/).
2.  Christian Antognini show us that having the data **not fully populated** on each instance **could lead to bad performance** into this [blog post](http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/).

**But wait:** If my RAC service is an **active/passive** one (I mean the service is started on only one instance) then I would like to have **all the data fully populated into** **the In-Memory column store of the "active" instance ** (and not distributed across the instances), right?

<span style="text-decoration:underline;">Let's try to achieve this (all data fully populated into **the In-Memory column store of the active instance**):</span>

So, by default, the IMCS distribution is the following:

    SQL> alter table bdt inmemory ;

    Table altered.

    SQL> select count(*) from bdt;

      COUNT(*)
    ----------
      38220000

    SQL> SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
    FROM gv$im_user_segments
    WHERE segment_name = 'BDT';

       INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- --------- ------------- ---------- -------------------
             1 COMPLETED     313720832 1610612736           732684288
             2 COMPLETED     274530304 1610612736           826236928

As you can see **no instance contains all data** as the BYTES\_NOT\_POPULATED column is  greater than 0.

Now let's set the hidden parameter **"*\_inmemory\_auto\_distribute*" to false** and re-trigger the population on instance 1:

    SQL> alter system set "_inmemory_auto_distribute"=false;

    System altered.

    SQL> select count(*) from bdt;

      COUNT(*)
    ----------
      38220000

    SQL> SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
    FROM gv$im_user_segments
    WHERE segment_name = 'BDT';

       INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- --------- ------------- ---------- -------------------
             1 COMPLETED     587137024 1610612736                   0
             2 COMPLETED     274530304 1610612736           826236928

**BINGO!** Look at the Instance 1, it now contains all the data in its IMCS (as BYTES\_NOT\_POPULATED=0).

If I connect to the Instance 2 and launch the query to trigger the population I'll get:

    SQL> SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
    FROM gv$im_user_segments
    WHERE segment_name = 'BDT';
      2    3  
       INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
    ---------- --------- ------------- ---------- -------------------
             1 COMPLETED     587137024 1610612736                   0
             2 COMPLETED     589234176 1610612736                   0

So, **both instances now contain all the data (like the duplicate attribute would have done).**

<span style="text-decoration:underline;">**Remarks:**</span>

1.  In case of active/passive service (means preferred/available service) then after a service failover the second instance will also contain all the data (once populated).
2.  In case of active/active service then each database instance will contain all the data but not necessary at the same time (so the behaviour is a little bit different of the *duplicate* attribute).
3.  I used an hidden parameter, so you should get oracle support approval to use it.
4.  I'll check the behaviour with PDB into [another post](http://bdrouvot.wordpress.com/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/ "In-Memory Instance distribution with RAC and Multitenant Environment").

<span style="text-decoration:underline;">**Conclusion:**</span>

-   Thanks to the hidden parameter "*\_inmemory\_auto\_distribute*" we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.
-   By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
-   With "***\_inmemory\_auto\_distribute***" set to **false**, then we are able to populate one instance (or all the instances) **with all the data**.

**Update 2015/03/27:**

-   This “workaround” proposed to get rid of the default behaviour on non Engineered Systems (“Every row of the test table is stored in the IMCS of either one instance or the other”) **works for any type of service.**
-   In case of preferred/available service, there is no need to set the hidden parameter to get the IM fully populated on one node. The secret sauce is linked to the ***parallel\_instance\_group*** parameter. See [this blog post](https://bdrouvot.wordpress.com/2015/03/26/in-memory-instance-distribution-with-rac-databases-impact-of-the-parallel_instance_group-parameter/) for more details.

---
layout: post
title: 'In-Memory Instance distribution with RAC databases: Impact of the parallel_instance_group
  parameter'
date: 2015-03-26 17:36:59.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- In-Memory
tags: []
meta:
  _wpas_skip_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_7950430: '1'
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/JCEWehi99sH
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/YZLHkTPPKG
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5986898459993604096&type=U&a=quY8
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/03/26/in-memory-instance-distribution-with-rac-databases-impact-of-the-parallel_instance_group-parameter/"
---

INTRODUCTION
============

As you may know, with a RAC database, **by default all objects populated into**  
**In-memory will be distributed across all of the IM column stores in the cluster**. It is also possible to have all the data appear in the IM column store on every node (Engineered Systems only). See this [white paper](http://www.oracle.com/technetwork/database/in-memory/overview/twp-oracle-database-in-memory-2245633.html) for more details.

Into this [blog post](https://blogs.oracle.com/In-Memory/entry/oracle_database_in_memory_on1) we have been told that:

-   RAC services and the *parallel\_instance\_group* parameter can be used to control where data is populated, and the subsequent access of that data.
-   In a RAC environment services enable the use of a subset of nodes and still insure that all In-Memory queries will be able to access all IM column stores.

My customer uses a RAC service defined as **preferred/available** on a 2 nodes RAC. In this configuration, I would like to **see how the data is populated into** **the In-Memory column store (IMCS) on both Instances when the *parallel\_instance\_group* parameter is used .**

TESTS
=====

I'll do several tests and stop/start the 12.1.0.2 database between each tests (to avoid confusion).

First let's create a service BDT\_PREF as preferred/available:

    srvctl add service -db BDT12c1 -s BDT_PREF -preferred BDT12c1_1 -available BDT12c1_2

so that the Instance 1 (BDT12c1\_1) is the preferred one.

Test 1:
-------

-   *parallel\_instance\_group* **not set.**

Connect to the database through the BDT\_PREF service and check the data distribution.

    SQL> alter table bdt inmemory priority none;

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
             1 COMPLETED     308477952 1610612736           732684288
             2 COMPLETED     274530304 1610612736           826236928

As you can see **no instance contains all data** as the BYTES\_NOT\_POPULATED column is  greater than 0. The data is **distributed across all of the IM column stores in the cluster**.

Test 2:
-------

-   *parallel\_instance\_group* set on Instance 1.
-   The service is running on Instance 1.

Set the *parallel\_instance\_group* parameter on the Instance BDT12c1\_1 (INST\_ID=1) to the service:

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_1';

    System altered.

Check that the BDT\_PREF service is running on the Instance 1 (where *parallel\_instance\_group* has been set) :

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_1

Connect to the database through the BDT\_PREF service and check the data distribution:

    SQL> alter table bdt inmemory priority none;

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
             1 COMPLETED     575602688 1610612736                   0

As you can see the **Instance 1 contains all the data** in its IMCS (as BYTES\_NOT\_POPULATED=0).

Test 3:
-------

-   *parallel\_instance\_group* set on Instance 1.
-   The service is running on Instance 2.

The fact that the service is not running on its preferred Instance could happen after:

-   A failover.
-   The database startup: When you use automatic services in an administrator-managed database, during planned database startup, services may start on the first instances that become available rather than their preferred instances (see the [oracle documentation](https://docs.oracle.com/database/121/TDPRC/configwlm.htm#TDPRC293)).
-   A manual relocation.

Set the *parallel\_instance\_group* parameter on the Instance BDT12c1\_1 (INST\_ID=1) to the service:

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_1';

    System altered.

Relocate the BDT\_PREF service on the Instance 2 (if needed):

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_1

    SQL> host srvctl relocate service -d BDT12c1 -s BDT_PREF -newinst BDT12c1_2 -oldinst BDT12c1_1

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_2

Connect to the database through the BDT\_PREF service and check the data distribution:

    SQL> alter table bdt inmemory priority none;

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
             2 COMPLETED     277676032 1610612736           826236928
             1 COMPLETED     312672256 1610612736           732684288

As you can see **no instance contains all data** as the BYTES\_NOT\_POPULATED column is  greater than 0. The data is **distributed across all of the IM column stores in the cluster**.

**Remarks:**

1.  This distribution [could lead to performance issue.](http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/)
2.  If a failover occurred and the Instance 1 is still down, then the data is fully populated on the Instance 2.

Test 4:
-------

-   *parallel\_instance\_group* set on Instance 1 and on Instance 2.
-   The service is running on Instance 1.

Set the *parallel\_instance\_group* parameter on both Instances to the service:

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_1';

    System altered.

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_2';

    System altered.

Check that the BDT\_PREF service is running on the Instance 1:

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_1

Connect to the database through the BDT\_PREF service and check the data distribution:

    SQL> alter table bdt inmemory priority none;

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
             1 COMPLETED     575602688 1610612736                   0

As you can see the **Instance 1 contains all the data** in its IMCS (as BYTES\_NOT\_POPULATED=0).

Test 5:
-------

-   *parallel\_instance\_group* set on Instance 1 and on Instance 2.
-   The service is running on Instance 2.

Set the *parallel\_instance\_group* parameter on both Instances to the service:

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_1';

    System altered.

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_2';

    System altered.

Relocate the BDT\_PREF service on the Instance 2 (if needed):

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_1

    SQL> host srvctl relocate service -d BDT12c1 -s BDT_PREF -newinst BDT12c1_2 -oldinst BDT12c1_1

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_2

Connect to the database through the BDT\_PREF service and check the data distribution:

    SQL> alter table bdt inmemory priority none;

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
             2 COMPLETED     575602688 1610612736                   0

As you can see the **Instance 2 contains all the data** in its IMCS (as BYTES\_NOT\_POPULATED=0).

Out of curiosity, what about a service defined as preferred on both Instances and the *parallel\_instance\_group* set on both Instances?

Test 6:
-------

-   *parallel\_instance\_group* set on Instance 1 and on Instance 2.
-   The service is running on Instance 1 and 2.

First let's create a service BDT\_PREF as preferred on both Instances:

    srvctl add service -db BDT12c1 -s BDT_PREF -preferred BDT12c1_1,BDT12c1_2

Set the *parallel\_instance\_group* parameter on both Instances to the service:

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_1';

    System altered.

    SQL> alter system set parallel_instance_group=BDT_PREF scope=both sid='BDT12c1_2';

    System altered.

Check that the BDT\_PREF service is running on both Instances:

    SQL> host srvctl status service -d BDT12c1 -s BDT_PREF
    Service BDT_PREF is running on instance(s) BDT12c1_1,BDT12c1_2

Connect to the database through the BDT\_PREF service and check the data distribution:

    SQL> alter table bdt inmemory priority none;

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
             2 COMPLETED     277676032 1610612736           826236928
             1 COMPLETED     312672256 1610612736           732684288

As you can see **no instance contains all data** as the BYTES\_NOT\_POPULATED column is  greater than 0. The data is **distributed across all of the IM column stores in the cluster**.

REMARKS
=======

-   I am not discussing the impact of the distribution on the performance (with or without [Auto DOP](https://blogs.oracle.com/In-Memory/entry/oracle_database_in_memory_on)).
-   I just focus **on how** the data has been distributed.
-   In this previous [blog post](https://bdrouvot.wordpress.com/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/) the "workaround" proposed to get rid of the default behaviour on non Engineered Systems (“Every row of the test table is stored in the IMCS of either one instance or the other”) **works for any type of service**.
-   In the current post I focus **only** on service defined as preferred/available on a 2 nodes RAC (The last test is out of curiosity).

CONCLUSION
==========

With a 12.1.0.2 database and a service defined as preferred/available on a 2 nodes RAC:

-   With the *parallel\_instance\_group* parameter set to the service on **both** Instances:

The data **is fully populated** on the Instance the service is running on.

-   With the *parallel\_instance\_group* parameter set to the service on the preferred Instance **only**:

If the service is running on the **preferred** Instance, the data **is fully populated** **on the Instance.**

If the service is running on the "**available**" Instance, the data **is** **distributed across all of the IM column stores in the cluster,** .

With a 12.1.0.2 database, a service defined as preferred on both Instances on a 2 nodes RAC and *parallel\_instance\_group* parameter set to the service on both Instances:

The data **is** **distributed across all of the IM column stores in the cluster.**

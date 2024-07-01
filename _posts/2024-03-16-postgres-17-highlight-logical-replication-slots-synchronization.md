---
layout: post
title: 'Postgres 17 highlight: Logical replication slots synchronization'
date: 2024-03-16 06:26:32.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Postgresql
tags: [PostgreSQL]
author:
  login: bdrouvot
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2024/03/16/postgres-17-highlight-logical-replication-slots-synchronization/"
---

### Introduction

PostgreSQL 17 will normally (as there is always a risk of seeing something reverted in the beta phase) include this commit:
[Add a new slot sync worker to synchronize logical slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=93db6cbda037f1be9544932bd9a785dabf3ff712).

```
commit 93db6cbda037f1be9544932bd9a785dabf3ff712
Author: Amit Kapila <akapila@postgresql.org>
Date:   Thu Feb 22 15:25:15 2024 +0530

Add a new slot sync worker to synchronize logical slots.

By enabling slot synchronization, all the failover logical replication
slots on the primary (assuming configurations are appropriate) are
automatically created on the physical standbys and are synced
periodically. The slot sync worker on the standby server pings the primary
server at regular intervals to get the necessary failover logical slots
information and create/update the slots locally. The slots that no longer
require synchronization are automatically dropped by the worker.

The nap time of the worker is tuned according to the activity on the
primary. The slot sync worker waits for some time before the next
synchronization, with the duration varying based on whether any slots were
updated during the last cycle.

A new parameter sync_replication_slots enables or disables this new
process.

On promotion, the slot sync worker is shut down by the startup process to
drop any temporary slots acquired by the slot sync worker and to prevent
the worker from trying to fetch the failover slots.

A functionality to allow logical walsenders to wait for the physical will
be done in a subsequent commit.
```

It means that logical replication slots synchronization from primary to standby
is now part of core PostgreSQL. 

### Let's look at an example

On the primary server you can now create a logical replication slot with an extra *failover* flag:

```
postgres@primary=# SELECT 'init' FROM pg_create_logical_replication_slot('logical_slot', 'test_decoding', false, false, true);
 ?column?
----------
 init
```

The *failover* flag is the third boolean and it has been set to true.
This information appears in the *pg_replication_slots* view:

```
postgres@primary=# select slot_name, failover from pg_replication_slots where slot_name = 'logical_slot';
  slot_name   | failover
--------------+----------
 logical_slot | t
```

It indicates that the slot will be synced to the standby servers (if they are configured for).

Now, let's setup a standby to synchronize this slot from the primary by:

Adding a dbname to *primary_conninfo*:

```
postgres@standby=# alter system set primary_conninfo = 'user=repliuser host=localhost port=55449 dbname=postgres';
ALTER SYSTEM

postgres@standby=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
```
  
Setting the new GUC parameter *sync_replication_slots* to true:

```
postgres@standby=# alter system set sync_replication_slots = true;
ALTER SYSTEM

postgres@standby=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
```

Please note that it is also mandatory to have the GUC parameters *hot_standby_feedback*
and *primary_slot_name* being respectively set to "on" and to a physical replication
slot that has been created on the primary:

```
postgres@standby=# show hot_standby_feedback;
 hot_standby_feedback
----------------------
 on

postgres@standby=# show primary_slot_name;
 primary_slot_name
-------------------
 master_slot
```

With the above in place, you'd see in the standby log file something like:

```
2024-03-16 08:00:24.144 UTC [127030] LOG:  newly created slot "logical_slot" is sync-ready now
```

and you'd see the logical replication slot on the standby:

```
postgres@standby=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots ;
  slot_name   | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
--------------+-----------+--------------+-------------+---------------------+----------+--------
 logical_slot | logical   |          757 | 0/C05C88    | 0/C05CC0            | t        | t
```

You can see that it is a **failover** slot and that it has been **synced**.

Now, let's consume the slot on the primary:

```
postgres@primary=# create table bdt(a int);
CREATE TABLE
postgres@primary=# insert into bdt values (1);
INSERT 0 1
postgres@primary=#  SELECT * FROM pg_logical_slot_get_changes('logical_slot', NULL, NULL, 'include-xids', '0');
   lsn    | xid |                  data
----------+-----+----------------------------------------
 0/C15038 | 757 | BEGIN
 0/C2B510 | 757 | COMMIT
 0/C2B510 | 758 | BEGIN
 0/C2B510 | 758 | table public.bdt: INSERT: a[integer]:1
 0/C2B580 | 758 | COMMIT
```

And verify that some slot's metadata have been synced on the standby by comparing
their values on the primary:

```
postgres@primary=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots where slot_name = 'logical_slot';
  slot_name   | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
--------------+-----------+--------------+-------------+---------------------+----------+--------
 logical_slot | logical   |          759 | 0/C0B8E8    | 0/C2B688            | t        | f
```

with their values on the standby:

```
postgres@standby=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots ;
  slot_name   | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
--------------+-----------+--------------+-------------+---------------------+----------+--------
 logical_slot | logical   |          759 | 0/C0B8E8    | 0/C2B688            | t        | t
```

As you can see the *catalog_xmin*, *restart_lsn* and *confirmed_flush_lsn* are the same.

It means that if the standby is promoted, the logical decoding **could resume** on the
promoted standby from the same position as it was on the primary.

### Let's look at another example making use of publication and subscription

The setup on the standby has to be the same as it was in the previous example regarding:

- *primary_conninfo*
- *sync_replication_slots*
- *hot_standby_feedback*
- *primary_slot_name*

Now on the primary, let's create a table and a publication:

```
postgres@primary=# create table tab_rep(a int primary key, b int);
CREATE TABLE

postgres@primary=# CREATE PUBLICATION mypub FOR ALL TABLES;
CREATE PUBLICATION
```

Let's connect to the subscriber node and create a table and a subscription with *failover* set to true:

```
postgres@subscriber=# create table tab_rep(a int primary key, b int);
CREATE TABLE

postgres@subscriber=# CREATE SUBSCRIPTION mysub CONNECTION 'host=localhost port=55449 dbname=postgres' PUBLICATION mypub WITH (failover = true);
NOTICE:  created replication slot "mysub" on publisher
CREATE SUBSCRIPTION
```

It automatically creates a logical replication slot on the primary with *failover* set to true:

```
postgres@primary=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots where slot_name = 'mysub';
 slot_name | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
-----------+-----------+--------------+-------------+---------------------+----------+--------
 mysub     | logical   |          761 | 0/C60380    | 0/C603B8            | t        | f
```

which has been synchronized on the standby:

```
postgres@standby=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots where slot_name = 'mysub';
 slot_name | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
-----------+-----------+--------------+-------------+---------------------+----------+--------
 mysub     | logical   |          761 | 0/C60380    | 0/C603B8            | t        | t
```

Now, let's insert some data in the primary:

```
postgres@primary=# insert into tab_rep values (1,2);
INSERT 0 1
```

Display some slot's metadata on the primary:

```
postgres@primary=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots where slot_name = 'mysub';
 slot_name | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
-----------+-----------+--------------+-------------+---------------------+----------+--------
 mysub     | logical   |          762 | 0/C612E8    | 0/C61320            | t        | f
```

And verify that they have been synced on the standby:

```
postgres@standby=# select slot_name, slot_type, catalog_xmin, restart_lsn, confirmed_flush_lsn, failover, synced from pg_replication_slots where slot_name = 'mysub';
 slot_name | slot_type | catalog_xmin | restart_lsn | confirmed_flush_lsn | failover | synced
-----------+-----------+--------------+-------------+---------------------+----------+--------
 mysub     | logical   |          762 | 0/C612E8    | 0/C61320            | t        | t
```

Out of curiosity, check that the data is in the subscriber node:

```
postgres@subscriber=# select * from tab_rep;
 a | b
---+---
 1 | 2
```

Now let's promote the standby:

```
postgres@standby=# select pg_promote();
 pg_promote
------------
 t
```

And update the subscription so that it now uses the promoted standby:

```
postgres@subscriber=# ALTER SUBSCRIPTION mysub CONNECTION 'host=localhost port=65449 dbname=postgres';
ALTER SUBSCRIPTION
```

So that if some data is inserted into the promoted standby:

```
postgres@standby=# insert into tab_rep values (2,2);
INSERT 0 1
```

It gets replicated to the subscriber:

```
postgres@subscriber=# select * from tab_rep;
   a   | b
-------+---
     1 | 2
     2 | 2
```

As you can see, we've been **able to resume** the subscription from the synced slot
on the promoted standby.

### Remarks

A new GUC ~~*standby_slot_names*~~  *synchronized_standby_slots* has also been added: it provides a way to **ensure** that physical standbys that are
potential failover candidates have received and flushed changes before the primary server makes them visible to subscribers.

In the example above, it was set to:

```
postgres=# show synchronized_standby_slots;
 synchronized_standby_slots
----------------------------
 master_slot
```

A new function *pg_sync_replication_slots()* has also been added to synchronize the slots manually. 

It's not possible to consume synced logical replication slots on the standby (until it is promoted).

Logical replication slot synchronization is currently not supported on a cascading standby.

In addition to the one mentioned in the Introduction, the following commits have also been implemented:

- [Rename standby_slot_names to synchronized_standby_slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2357c9223b710c91b0f05cbc56bd435baeac961f)
- [Allow to enable failover property for replication slots via SQL API](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c393308b69)
- [Allow setting failover property in the replication command](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7329240437)
- [Add a failover option to subscriptions](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=776621a5e4)
- [Enhance libpqrcv APIs to support slot synchronization](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dafbfed9ef)
- [Add a slot synchronization function](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ddd5f4f54a)
- [Introduce a new GUC 'standby_slot_names'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf279ddd1c)

### Conclusion

Thanks to those new commits it will be possible to:

- create logical replication slots on primary that are synchronized automatically (or manually) on standbys
- resume the logical decoding from the synced slots once the standby is promoted

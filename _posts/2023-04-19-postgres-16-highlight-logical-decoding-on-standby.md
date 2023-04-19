---
layout: post
title: 'Postgres 16 highlight: Logical decoding on standby'
date: 2023-04-19 17:26:32.000000000 +01:00
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
permalink: "/2023/04/19/postgres-16-highlight-logical-decoding-on-standby/"
---

### Introduction

PostgreSQL 16 will normally (as there is always a risk of seeing something reverted in the beta phase) include this commit:
[Allow logical decoding on standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0fdab27ad68a059a1663fa5ce48d76333f1bd74c).

```
commit: 0fdab27ad68a059a1663fa5ce48d76333f1bd74c
committer: Andres Freund <andres@anarazel.de>	
date: Sat, 8 Apr 2023 09:20:05 +0000 (02:20 -0700)

Allow logical decoding on standbys

Unsurprisingly, this requires wal_level = logical to be set on the primary and
standby. The infrastructure added in 26669757b6a ensures that slots are
invalidated if the primary's wal_level is lowered.

Creating a slot on a standby waits for a xl_running_xact record to be
processed. If the primary is idle (and thus not emitting xl_running_xact
records), that can take a while.  To make that faster, this commit also
introduces the pg_log_standby_snapshot() function. By executing it on the
primary, completion of slot creation on the standby can be accelerated.

Note that logical decoding on a standby does not itself enforce that required
catalog rows are not removed. The user has to use physical replication slots +
hot_standby_feedback or other measures to prevent that. If catalog rows
required for a slot are removed, the slot is invalidated.

See 6af1793954e for an overall design of logical decoding on a standby.

Bumps catversion, for the addition of the pg_log_standby_snapshot() function.
```

It means that logical replication slot creation and logical decoding is possible on a standby.

### Before this commit

Trying to create a logical replication slot on a standby would produce:

````
postgres@standby=# SELECT * FROM pg_create_logical_replication_slot('active_slot', 'test_decoding', false, true);
ERROR:  logical decoding cannot be used while in recovery
````  

It is not possible to create a logical replication slot on a standby.

### With this commit

It's now possible:

```
postgres@standby=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

postgres@standby=# SELECT * FROM pg_create_logical_replication_slot('active_slot', 'test_decoding', false, true);
  slot_name  |    lsn
-------------+------------
 active_slot | 0/C0053A78
(1 row)
```

Please note that (as explained in the commit message), if the primary is idle, the slot
creation can take a while.  
To make that faster, the following function can be launched on the primary:

```
postgres@primary=# select pg_log_standby_snapshot();
 pg_log_standby_snapshot
-------------------------
 0/C0053A78
(1 row)
```

so that the primary emits the WAL record the logical replication slot creation on the standby 
was waiting for.

#### Let's look at an example

Once the replication slot has been created you can start decoding from it.  
Let's create a table and insert one row on the primary:

```
postgres@primary=# create table bdt(a int);
CREATE TABLE
postgres@primary=# insert into bdt values (1);
INSERT 0 1
```

And decode from the standby:

```
postgres@standby=# SELECT * FROM pg_logical_slot_peek_changes('active_slot',NULL,NULL);
    lsn     | xid |                  data
------------+-----+----------------------------------------
 0/C0053BC8 | 749 | BEGIN 749
 0/C00695D8 | 749 | COMMIT 749
 0/C0069610 | 750 | BEGIN 750
 0/C0069610 | 750 | table public.bdt: INSERT: a[integer]:1
 0/C0069680 | 750 | COMMIT 750
(5 rows)
```

As you can see, we have been able to decode the changes done on the primary 
from the standby.

#### What if the standby get promoted?

Say, there is logical decoding in progress on the standby:

```
$ pg_recvlogical -d postgres -S active_slot -f - --no-loop --start
BEGIN 749
COMMIT 749
BEGIN 750
table public.bdt: INSERT: a[integer]:1
COMMIT 750
```

Promote the standby:

```
postgres@standby=# select pg_promote();
 pg_promote
------------
 t
(1 row)
```

First, you could observe that the pg_recvlogical process has not been canceled/interrupted
and is still running.

Secondly, what if we insert one row into the table but this time on the **promoted** standby?

```
postgres@standby=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 f
(1 row)

postgres@standby=# insert into bdt values (1);
INSERT 0 1
```

We can see that pg_recvlogical has been able to decode this new insert:

```
$ pg_recvlogical -d postgres -S active_slot -f - --no-loop --start
BEGIN 749
COMMIT 749
BEGIN 750
table public.bdt: INSERT: a[integer]:1
COMMIT 750
BEGIN 751
table public.bdt: INSERT: a[integer]:1
COMMIT 751
```

It means that without any interruption pg_recvlogical is still doing the decoding
**after** the standby got promoted.


#### Thanks to this new feature, can we also subscribe to a standby?

Let's create a table and a publication on the primary:

```
postgres@primary=# CREATE TABLE tab_rep (a int primary key);
CREATE TABLE
postgres@primary=# CREATE PUBLICATION tap_pub;
CREATE PUBLICATION
postgres@primary=# ALTER PUBLICATION tap_pub ADD TABLE tab_rep;
ALTER PUBLICATION
```

As expected, the table and the publication are replicated to the standby:

```
postgres@standby=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

postgres@standby=# select * from pg_publication;
  oid  | pubname | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+---------+----------+--------------+-----------+-----------+-----------+-------------+------------
 17055 | tap_pub |       10 | f            | t         | t         | t         | t           | f
(1 row)

postgres@standby=# select * from tab_rep;
 a
---
(0 rows)
```

Let's now subscribe from another engine to the **publication on the standby**.  
Get the standby port:

```
postgres@standby=# show port;
 port
-------
 65448
(1 row)
```

Create the table on the subscriber:

```
postgres@subscriber=# CREATE TABLE tab_rep (a int primary key);
CREATE TABLE
```

**Without** the new feature, creating a subscription to a standby would produce:

```
postgres@subscriber=# CREATE SUBSCRIPTION mysub CONNECTION 'host=localhost port=65448 dbname=postgres' publication tap_pub;
ERROR:  could not create replication slot "mysub": ERROR:  logical decoding cannot be used while in recovery
```

**Thanks** to the new feature, it's now possible:

```
postgres@subscriber=# CREATE SUBSCRIPTION mysub CONNECTION 'host=localhost port=65448 dbname=postgres' publication tap_pub;
NOTICE:  created replication slot "mysub" on publisher
CREATE SUBSCRIPTION
```

Please note that, if the primary is idle, the subscribtion creation can take a while.  
To make that faster, the following function can be launched on the primary:

```
postgres@primary=# select pg_log_standby_snapshot();
 pg_log_standby_snapshot
-------------------------
 0/C0075EB0
(1 row)
```

Now that the subscription has been created, let's insert one row on the **primary**:

```
postgres@primary=# insert into tab_rep values (1);
INSERT 0 1
```

Verify that it has been propagated to the **standby**:

```
postgres@standby=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)

postgres@standby=# select * from tab_rep;
 a
---
 1
(1 row)
```

And that the **subscriber** can see it:

```
postgres@subscriber=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 f
(1 row)

postgres@subscriber=# select * from tab_rep;
 a
---
 1
(1 row)
```

As we can see, the subscriber has been able to get the row inserted in the primary while
subscribing to the standby.

### Remark

To allow logical decoding on standbys, prior to the one mentioned in the Introduction, the following commits have also been implemented:

- [Pass down table relation into more index relation functions](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61b313e47eb987682441c675724c22bf4363c9c4)
- [Add info in WAL records in preparation for logical slot conflict handling](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6af1793954e8c5e753af83c3edb37ed3267dd179) (this commit message also explains the overall design)
- [Replace replication slot's invalidated_at LSN with an enum](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=15f8203a5975d6b9b78e2c64e213ed964b50c044)
- [Prevent use of invalidated logical slot in CreateDecodingContext()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4397abd0a2af955326c0608d63f3716ce5901004)
- [Support invalidating replication slots due to horizon and wal_level](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=be87200efd9308ccfe217ce8828f316e93e370da)
- [Handle logical slot conflicts on standby](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26669757b6a7665c1069e77e6472bd8550193ca6)
- [For cascading replication, wake physical and logical walsenders separately](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e101dfac3a53c20bfbf1ca85d30a368c2954facf)

### Conclusion

Thanks to this new [commit](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0fdab27ad68a059a1663fa5ce48d76333f1bd74c)
it will be possible to:

- create logical replication slot on a standby
- launch logical decoding from a standby
- subscribe to a standby

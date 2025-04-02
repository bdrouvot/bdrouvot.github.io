---
layout: post
title: 'Postgres backend statistics (Part 2): WAL statistics'
date: 2025-04-02 06:26:32.000000000 +01:00
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
permalink: "/2025/04/02/postgres-backend-statistics-part-2/"
---

### Introduction

PostgreSQL 18 will normally (as there is always a risk of seeing something reverted until its GA release) include those commits:
[Add data for WAL in pg_stat_io and backend statistics](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a051e71e28a12342a4fb39a3c149a197159f9c46):

```
commit a051e71e28a12342a4fb39a3c149a197159f9c46
Author: Michael Paquier <michael@paquier.xyz>
Date:   Tue Feb 4 16:50:00 2025 +0900

Add data for WAL in pg_stat_io and backend statistics

This commit adds WAL IO stats to both pg_stat_io view and per-backend IO
statistics (pg_stat_get_backend_io()).
.
.
```
and [Add WAL data to backend statistics](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=76def4cdd7c2b32d19e950a160f834392ea51744):

```
commit 76def4cdd7c2b32d19e950a160f834392ea51744
Author: Michael Paquier <michael@paquier.xyz>
Date:   Tue Mar 11 09:04:11 2025 +0900

Add WAL data to backend statistics

This commit adds per-backend WAL statistics, providing the same
information as pg_stat_wal, except that it is now possible to know how
much WAL activity is happening in each backend rather than an overall
aggregate of all the activity.  Like pg_stat_wal, the implementation
relies on pgWalUsage, tracking the difference of activity between two
reports to pgstats.

This data can be retrieved with a new system function called
pg_stat_get_backend_wal(), that returns one tuple based on the PID
provided in input.  Like pg_stat_get_backend_io(), this is useful when
joined with pg_stat_activity to get a live picture of the WAL generated
for each running backend, showing how the activity is [un]balanced.
.
.
```

It means that:

- WAL IO statistics are available per backend through the *pg_stat_get_backend_io()* function (already introduced in [Postgres backend statistics (Part 1)](https://bdrouvot.github.io/2025/01/07/postgres-backend-statistics-part-1/))
- WAL statistics are available per backend through the *pg_stat_get_backend_wal()* function

So that we can see the WAL activity in each backend.

### Let's look at some examples

Thanks to the *pg_stat_get_backend_io()* function, we can:
  
#### Retrieve the WAL IO statistics for my backend

```
db1=# SELECT backend_type, object, context, reads, read_bytes, read_time, writes, write_bytes, write_time, fsyncs, fsync_time  FROM pg_stat_get_backend_io(pg_backend_pid()) where object = 'wal';
  backend_type  | object | context | reads | read_bytes | read_time | writes | write_bytes | write_time | fsyncs | fsync_time
----------------+--------+---------+-------+------------+-----------+--------+-------------+------------+--------+------------
 client backend | wal    | init    |       |            |           |      0 |           0 |          0 |      0 |          0
 client backend | wal    | normal  |     0 |          0 |         0 |   4533 |    41320448 |          0 |      2 |          0
(2 rows)

```

Please note that [track_wal_io_timing](https://www.postgresql.org/docs/current/runtime-config-statistics.html#GUC-TRACK-WAL-IO-TIMING)
needs to be enabled to see the IO timings for the WAL object:

```
db1=# SET track_wal_io_timing=true;
SET
db1=# insert into bdt select generate_series (1, 1000);
INSERT 0 1000
db1=# SELECT backend_type, object, context, reads, read_bytes, read_time, writes, write_bytes, write_time, fsyncs, fsync_time  FROM pg_stat_get_backend_io(pg_backend_pid()) where object = 'wal';
  backend_type  | object | context | reads | read_bytes | read_time | writes | write_bytes |      write_time      | fsyncs | fsync_time
----------------+--------+---------+-------+------------+-----------+--------+-------------+----------------------+--------+------------
 client backend | wal    | init    |       |            |           |      0 |           0 |                    0 |      0 |          0
 client backend | wal    | normal  |     0 |          0 |         0 |   4535 |    41443328 | 0.026000000000000002 |      4 |      0.513
(2 rows)
```
and that [track_io_timing](https://www.postgresql.org/docs/current/runtime-config-statistics.html#GUC-TRACK-IO-TIMING)
has no effects on the WAL object timings:

```
db1=# SET track_wal_io_timing=false;
SET
db1=# SET track_io_timing=true;
SET
db1=# SELECT pg_stat_reset_backend_stats(pg_backend_pid());
 pg_stat_reset_backend_stats
-----------------------------

(1 row)

db1=# SELECT backend_type, object, context, reads, read_bytes, read_time, writes, write_bytes, write_time, fsyncs, fsync_time  FROM pg_stat_get_backend_io(pg_backend_pid()) where object = 'wal';
  backend_type  | object | context | reads | read_bytes | read_time | writes | write_bytes | write_time | fsyncs | fsync_time
----------------+--------+---------+-------+------------+-----------+--------+-------------+------------+--------+------------
 client backend | wal    | init    |       |            |           |      0 |           0 |          0 |      0 |          0
 client backend | wal    | normal  |     0 |          0 |         0 |      0 |           0 |          0 |      0 |          0
(2 rows)

db1=# insert into bdt select generate_series (1, 1000);
INSERT 0 1000
db1=# SELECT backend_type, object, context, reads, read_bytes, read_time, writes, write_bytes, write_time, fsyncs, fsync_time  FROM pg_stat_get_backend_io(pg_backend_pid()) where object = 'wal';
  backend_type  | object | context | reads | read_bytes | read_time | writes | write_bytes | write_time | fsyncs | fsync_time
----------------+--------+---------+-------+------------+-----------+--------+-------------+------------+--------+------------
 client backend | wal    | init    |       |            |           |      0 |           0 |          0 |      0 |          0
 client backend | wal    | normal  |     0 |          0 |         0 |      1 |       65536 |          0 |      1 |          0
(2 rows)

```

Using *pg_backend_pid()* as input of *pg_stat_get_backend_io()*, I can see the
WAL IO statistics for my backend.

#### Find out the top 3 backends that are generating the most WAL bytes

```
db2=# SELECT a.backend_type,datname, application_name, pid, sum(write_bytes)
FROM pg_stat_activity a, pg_stat_get_backend_io(pid)
WHERE write_bytes != 0 AND object = 'wal'
GROUP BY a.backend_type,datname, application_name, pid
ORDER BY 5 desc
LIMIT 3;
  backend_type  | datname | application_name |   pid   |   sum
----------------+---------+------------------+---------+----------
 client backend | db2     | app2             | 3960495 | 41844736
 walwriter      |         |                  | 3960489 | 22429696
 client backend | db1     | app1             | 3960493 |    98304
(3 rows)
```

Using *pg_stat_get_backend_io()* in conjonction with *pg_stat_activity*, I can
figure out which backends are generating the most WAL bytes.

#### Get the WAL writes generated by application (and the ratio cluster wide)

```
db2=# SELECT application_name, writes, round(100 * writes/sum(writes) over(),2) AS "%"
FROM  (SELECT application_name, sum(writes) AS writes
       FROM pg_stat_activity, pg_stat_get_backend_io(pid)
       WHERE writes != 0 and object = 'wal' GROUP BY application_name);
 application_name | writes |   %
------------------+--------+-------
 app1             |      1 |  0.02
 app2             |   4598 | 99.48
                  |     23 |  0.50
(3 rows)
```

Using *pg_stat_get_backend_io()* in conjonction with *pg_stat_activity* and windows
function, I can get the WAL writes generated by application (and the ratio cluster wide).

Also, thanks to the *pg_stat_get_backend_wal()* function, we can:

#### Get the number of WAL records generated by application (and the ratio cluster wide)

```
db2=# SELECT application_name, wal_records, round(100 * wal_records/sum(wal_records) over(),2) AS "%"
FROM  (SELECT application_name, sum(wal_records) AS wal_records
       FROM pg_stat_activity, pg_stat_get_backend_wal(pid)
       WHERE wal_records != 0 GROUP BY application_name);
 application_name | wal_records |   %
------------------+-------------+-------
 app1             |        1004 |  0.10
 app2             |     1000004 | 99.90
(2 rows)
```
Using *pg_stat_get_backend_wal()* in conjonction with *pg_stat_activity* and windows
function, I can get the number of WAL records generated by application (and the ratio cluster wide).

There is much more we can do, the examples above are far from exhaustive.

### Remarks

Please note that *pg_stat_get_backend_wal()* also provides the "WAL bytes" being
generated. The granularity as compared with the metric provided through *pg_stat_get_backend_io()*
is not the same though: *pg_stat_get_backend_wal()* focus on the WAL records size
while *pg_stat_get_backend_io()* focus on the [wal_block_size](https://www.postgresql.org/docs/current/runtime-config-preset.html#GUC-WAL-BLOCK-SIZE)
size.

The output of the new *pg_stat_get_backend_wal()* function has the same meaning
as the one from the *pg_stat_wal* view (please refer to the [documentation](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-WAL-VIEW)
). 

The per backend WAL statistics do not persist after a server restart (it would not
make sense to report statistics for backends that are gone). The *pg_stat_wal*
data persists though.

Once a backend disconnect its WAL related stats are not available anymore.

### Conclusion

It's now possible to see the WAL IO activity in each backend thanks to the
*pg_stat_get_backend_io()* function and the WAL statistics (such as the number
of WAL records generated) thanks to the *pg_stat_get_backend_wal()* function.

One can build insightful queries on top of those new functions.

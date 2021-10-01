---
layout: post
title: Report long recovery conflict wait times with PostgreSQL 14
date: 2021-09-30 12:26:32.000000000 +01:00
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
permalink: "/2021/09/30-report-long-recovery-conflict-wait-times-with-PostgreSQL-14/"
---

### Introduction

PostgreSQL 14 is introducing a new parameter [log_recovery_conflict_waits](https://www.postgresql.org/docs/14/runtime-config-logging.html#GUC-LOG-RECOVERY-CONFLICT-WAITS).
It is useful in determining if recovery conflicts prevent the recovery from applying WAL.

### Example

Say, you have a standby in place with [max_standby_streaming_delay](https://www.postgresql.org/docs/14/runtime-config-replication.html#GUC-MAX-STANDBY-STREAMING-DELAY) set to 30s (default value).
Then you would notice recovery conflicts (if any) when the recovery is waiting for more than 30s.

You would get, messages like:

	ERROR:  canceling statement due to conflict with recovery
	DETAIL:  User query might have needed to see row versions that must be removed.

And that would be also visible in the [pg_stat_database_conflicts](https://www.postgresql.org/docs/14/monitoring-stats.html#MONITORING-PG-STAT-DATABASE-CONFLICTS-VIEW) view:

	postgres=# select * from pg_stat_database_conflicts where datname='postgres';
	 datid | datname  | confl_tablespace | confl_lock | confl_snapshot | confl_bufferpin | confl_deadlock
	-------+----------+------------------+------------+----------------+-----------------+----------------
	 13842 | postgres |                0 |          0 |              1 |               1 |              0
	(1 row)

but you won't see any of those if the recovery was blocked for less than 30s (while generating lag).

It means this could be difficult to diagnose where the recovery lag is coming from (if any).

And that's where the new parameter comes into play, by setting it to `on` you would get things like:

	LOG:  recovery still waiting after 1024.777 ms: recovery conflict on snapshot
	DETAIL:  Conflicting process: 10156.
	CONTEXT:  WAL redo at 0/3044D08 for Heap2/PRUNE: latestRemovedXid 744 nredirected 0 ndead 1; blkref #0: rel 1663/13842/16385, blk 0
	LOG:  recovery finished waiting after 18042.573 ms: recovery conflict on snapshot
	CONTEXT:  WAL redo at 0/3044D08 for Heap2/PRUNE: latestRemovedXid 744 nredirected 0 ndead 1; blkref #0: rel 1663/13842/16385, blk 0

We can see the time the recovery had to wait (18s) before the conflict has been resolved (without any cancellations).

### Remarks

-   You can also see useful information about the WAL record the recovery was blocked on.
-   This WAL record information is printed in the same format as pg_waldump.

### Conclusion

Thanks to the new log_recovery_conflict_waits parameter we now have more information when a recovery conflict is occuring longer than deadlock_timeout.

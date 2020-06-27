---
layout: post
title: Sampling pg_stat_statements based on the active sessions and their associated
  queryid
date: 2019-07-06 13:08:13.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
- Postgresql
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _wpas_skip_7950430: '1'
  _publicize_job_id: '32549258046'
  timeline_notification: '1562414896'
  publicize_linkedin_url: www.linkedin.com/updates?topic=6553243055997145088
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1147477369931759617";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2019/07/06/sampling-pg_stat_statements-based-on-the-active-sessions-and-their-associated-queryid/"
---

### Introduction

Now that we have the ability to sample and record the active sessions and their associated queryid with the *pg\_active\_session\_history* view (see [this blog post](https://bdrouvot.wordpress.com/2018/07/07/postgresql-active-session-history-ash-welcome-to-the-pg_active_session_history-view-part-of-the-pgsentinel-extension/)), it would be interesting to have insights about the queries statistics at the time the sessions were active.

PostgreSQL provides the queries statistics with the [pg\_stat\_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) view. We could query the *pg\_active\_session\_history* and the *pg\_stat\_statements* views and join them on the queryid field, but the queries statistics would be:

-   cumulative
-   the ones at the time we would launch this query

So it would not provide the queries statistics at the time the sessions were active.

### What's new?

To get more granular queries statistics, the [pgsentinel](https://github.com/pgsentinel/pgsentinel) extension has evolved so that it now samples the *pg\_stat\_statements*:

-   at the same time it is sampling the active sessions
-   only for the queryid that were associated to an active session (if any) during the sampling

The samples are recorded into a new ***pg\_stat\_statements\_history*** view.

This view looks like:

                        View "public.pg_stat_statements_history"
           Column        |           Type           | Collation | Nullable | Default 
    ---------------------+--------------------------+-----------+----------+---------
     ash_time            | timestamp with time zone |           |          | 
     userid              | oid                      |           |          | 
     dbid                | oid                      |           |          | 
     queryid             | bigint                   |           |          | 
     calls               | bigint                   |           |          | 
     total_time          | double precision         |           |          | 
     rows                | bigint                   |           |          | 
     shared_blks_hit     | bigint                   |           |          | 
     shared_blks_read    | bigint                   |           |          | 
     shared_blks_dirtied | bigint                   |           |          | 
     shared_blks_written | bigint                   |           |          | 
     local_blks_hit      | bigint                   |           |          | 
     local_blks_read     | bigint                   |           |          | 
     local_blks_dirtied  | bigint                   |           |          | 
     local_blks_written  | bigint                   |           |          | 
     temp_blks_read      | bigint                   |           |          | 
     temp_blks_written   | bigint                   |           |          | 
     blk_read_time       | double precision         |           |          | 
     blk_write_time      | double precision         |           |          | 

### Remarks:

-   The fields description are the same as for [*pg\_stat\_statements*](https://www.postgresql.org/docs/11/pgstatstatements.html) (except for the *ash\_time* one, which is the time of the active session history sampling)
-   As for *pg\_active\_sessions\_history*, the *pg\_stat\_statements\_history* view is implemented as in-memory ring buffer where the number of samples to be kept is configurable (thanks to the *pgsentinel\_pgssh.max\_entries* parameter)
-   The data collected are still cumulative metrics but you can make use of the [window functions](https://www.postgresql.org/docs/11/tutorial-window.html) in PostgreSQL to compute the delta between samples (and then get accurate statistics for the queries between two samples)

For example, we could get per queryid and ash\_time:  the rows per second, calls per second and rows per call that way:

    select ash_time,queryid,delta_rows/seconds "rows_per_second",delta_calls/seconds "calls_per_second",delta_rows/delta_calls "rows_per_call"
    from(
    SELECT ash_time,queryid,
    EXTRACT(EPOCH FROM ash_time::timestamp) - lag (EXTRACT(EPOCH FROM ash_time::timestamp))
    OVER (
            PARTITION BY pgssh.queryid
            ORDER BY ash_time
            ASC) as "seconds",
    rows-lag(rows)
            OVER (
            PARTITION BY pgssh.queryid
            ORDER BY ash_time
            ASC) as "delta_rows",
    calls-lag(calls)
            OVER (
            PARTITION BY pgssh.queryid
            ORDER BY ash_time
            ASC) as "delta_calls"
        FROM pg_stat_statements_history pgssh) as delta
    where delta_calls > 0 and seconds > 0
    order by ash_time desc;

               ash_time            |      queryid       | rows_per_second  | calls_per_second | rows_per_call
    -------------------------------+--------------------+------------------+------------------+----------------
     2019-07-06 11:09:48.629218+00 | 250416904599144140 | 10322.0031121842 | 10322.0031121842 |              1
     2019-07-06 11:09:47.627184+00 | 250416904599144140 | 10331.3930170891 | 10331.3930170891 |              1
     2019-07-06 11:09:46.625383+00 | 250416904599144140 | 10257.7574710211 | 10257.7574710211 |              1
     2019-07-06 11:09:42.620219+00 | 250416904599144140 |  10296.311364551 |  10296.311364551 |              1
     2019-07-06 11:09:41.618404+00 | 250416904599144140 | 10271.6737455877 | 10271.6737455877 |              1
     2019-07-06 11:09:36.612209+00 | 250416904599144140 | 10291.1563299622 | 10291.1563299622 |              1
     2019-07-06 11:09:35.610378+00 | 250416904599144140 | 10308.9798914136 | 10308.9798914136 |              1
     2019-07-06 11:09:33.607367+00 | 250416904599144140 |  10251.230955397 |  10251.230955397 |              1
     2019-07-06 11:09:31.604193+00 | 250416904599144140 | 10284.3551339058 | 10284.3551339058 |              1
     2019-07-06 11:09:30.60238+00  | 250416904599144140 | 10277.4631222064 | 10277.4631222064 |              1
     2019-07-06 11:09:24.595353+00 | 250416904599144140 | 10283.3919912856 | 10283.3919912856 |              1
     2019-07-06 11:09:22.59222+00  | 250416904599144140 | 10271.3534800552 | 10271.3534800552 |              1
     2019-07-06 11:09:21.59021+00  | 250416904599144140 | 10300.1104655978 | 10300.1104655978 |              1
     2019-07-06 11:09:20.588376+00 | 250416904599144140 | 10343.9790974522 | 10343.9790974522 |              1
     2019-07-06 11:09:16.583341+00 | 250416904599144140 | 10276.5525289304 | 10276.5525289304 |              1

-   it's a good practice to normalize the statistics per second (as the sampling interval might change)
-   With that level of information we can understand the database activity in the past (thanks to the active sessions sampling) and get statistics per query at the time they were active
-   pgsentinel is available in [this github repository](https://github.com/pgsentinel/pgsentinel)

### Conclusion

The [pgsentinel](https://github.com/pgsentinel/pgsentinel) extension now provides:

-   Active session history (through the *pg\_active\_session\_history* view)
-   Queries statistics history (through the *pg\_stat\_statements\_history* view), recorded at the exact same time as their associated active sessions

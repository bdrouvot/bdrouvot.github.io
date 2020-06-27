---
layout: post
title: 'Active Session History in PostgreSQL: blocker and wait chain'
date: 2018-09-01 11:26:42.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Postgresql
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _wpas_skip_7950430: '1'
  _thumbnail_id: '3492'
  publicize_linkedin_url: www.linkedin.com/updates?topic=6441602047291727872
  timeline_notification: '1535797605'
  _publicize_job_id: '21682617338'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1035836362996498432";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_skip_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/09/01/active-session-history-in-postgresql-blocker-and-wait-chain/"
---

While the [active session history extension for PostgreSQL](https://github.com/pgsentinel/pgsentinel) is still in beta, some information is added to it.

The pg\_active\_session\_history view is currently made of:

                       View "public.pg_active_session_history"
          Column      |           Type           | Collation | Nullable | Default
    ------------------+--------------------------+-----------+----------+---------
     ash_time         | timestamp with time zone |           |          |
     datid            | oid                      |           |          |
     datname          | text                     |           |          |
     pid              | integer                  |           |          |
     usesysid         | oid                      |           |          |
     usename          | text                     |           |          |
     application_name | text                     |           |          |
     client_addr      | text                     |           |          |
     client_hostname  | text                     |           |          |
     client_port      | integer                  |           |          |
     backend_start    | timestamp with time zone |           |          |
     xact_start       | timestamp with time zone |           |          |
     query_start      | timestamp with time zone |           |          |
     state_change     | timestamp with time zone |           |          |
     wait_event_type  | text                     |           |          |
     wait_event       | text                     |           |          |
     state            | text                     |           |          |
     backend_xid      | xid                      |           |          |
     backend_xmin     | xid                      |           |          |
     top_level_query  | text                     |           |          |
     query            | text                     |           |          |
     cmdtype          | text                     |           |          |
     queryid          | bigint                   |           |          |
     backend_type     | text                     |           |          |
     blockers         | integer                  |           |          |
     blockerpid       | integer                  |           |          |
     blocker_state    | text                     |           |          |

You could see it as samplings of `pg_stat_activity` providing more information:

-   `ash_time`: the sampling time
-   `top_level_query`: the top level statement (in case PL/pgSQL is used)
-   `query`: the statement being executed (not normalised, as it is in `pg_stat_statements`, means you see the values)
-   `cmdtype`: the statement type (SELECT,UPDATE,INSERT,DELETE,UTILITY,UNKNOWN,NOTHING)
-   `queryid`: the queryid of the statement which links to pg\_stat\_statements
-   `blockers`: the number of blockers
-   `blockerpid`: the pid of the blocker (if blockers = 1), the pid of one blocker (if blockers &gt; 1)
-   `blocker_state`: state of the blocker (state of the blockerpid)

Thanks to the queryid field you are able to link the session activity with the sql activity.

The information related to the blocking activity (if any) has been added recently. Why? To easily drill down in case of session being blocked.

Let's see how we could display some interesting information in case of blocked session(s), for examples:

-   The wait chain
-   The seconds in wait in this chain
-   The percentage of the total wait time that this chain represents

As PostgreSQL provides [recursive query](https://www.postgresql.org/docs/current/static/queries-with.html) and [window functions](https://www.postgresql.org/docs/current/static/tutorial-window.html), let's make use of them to write this query:

\[code language="sql"\]  
postgres@pgu:~$ cat pg\_ash\_wait\_chain.sql  
WITH RECURSIVE search\_wait\_chain(ash\_time,pid, blockerpid, wait\_event\_type,wait\_event,level, path)  
AS (  
SELECT ash\_time,pid, blockerpid, wait\_event\_type,wait\_event, 1 AS level,  
'pid:'||pid||' ('||wait\_event\_type||' : '||wait\_event||') -&gt;'||'pid:'||blockerpid AS path  
from pg\_active\_session\_history WHERE blockers &gt; 0  
union ALL  
SELECT p.ash\_time,p.pid, p.blockerpid, p.wait\_event\_type,p.wait\_event, swc.level + 1 AS level,  
'pid:'||p.pid||' ('||p.wait\_event\_type||' : '||p.wait\_event||') -&gt;'||swc.path AS path  
FROM pg\_active\_session\_history p, search\_wait\_chain swc  
WHERE p.blockerpid = swc.pid and p.ash\_time = swc.ash\_time and p.blockers &gt; 0  
)  
select round(100 \* count(\*) / cnt)||'%' as "% of total wait",count(\*) as seconds,path as wait\_chain from (  
SELECT pid,wait\_event,path,sum(count) over() as cnt from (  
select ash\_time,level,pid,wait\_event,path,count(\*) as count, max(level) over(partition by ash\_time,pid) as max\_level  
FROM search\_wait\_chain where level &gt; 0 group by ash\_time,level,pid,wait\_event,path  
) as all\_wait\_chain  
where level=max\_level  
) as wait\_chain  
group by path,cnt  
order by count(\*) desc;  
\[/code\]

Let's launch this query while only one session is being blocked by another one:

    postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
     % of total wait | seconds |                 wait_chain
    -----------------+---------+--------------------------------------------
     100%            |      23 | pid:1890 (Lock : transactionid) ->pid:1888
    (1 row)

It means that the pid 1890 is waiting since 23 seconds on the transactionid wait event, while being blocked by pid 1888. This wait chain represents 100% of the blocking activity time.

Now another session comes into the game, query the active session history view one more time:

    postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
     % of total wait | seconds |                                  wait_chain
    -----------------+---------+------------------------------------------------------------------------------
     88%             |     208 | pid:1890 (Lock : transactionid) ->pid:1888
     12%             |      29 | pid:1913 (Lock : transactionid) ->pid:1890 (Lock : transactionid) ->pid:1888
    (2 rows)

So we still see our first blocking chain. It is now not the only one (so represents 88% of the blocking activity time).

We can see a new chain that represents 12% of the blocking activity time:

-   pid 1913 (waiting on transactionid) is blocked since 29 seconds by pid 1890 (waiting on transactionid) that is also blocked by pid 1888.

Let's commit the transaction hold by pid 1888 and launch the query again 2 times:

    postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
     % of total wait | seconds |                                  wait_chain
    -----------------+---------+------------------------------------------------------------------------------
     57%             |     582 | pid:1890 (Lock : transactionid) ->pid:1888
     40%             |     403 | pid:1913 (Lock : transactionid) ->pid:1890 (Lock : transactionid) ->pid:1888
     3%              |      32 | pid:1913 (Lock : transactionid) ->pid:1890
    (3 rows)

    postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
     % of total wait | seconds |                                  wait_chain
    -----------------+---------+------------------------------------------------------------------------------
     57%             |     582 | pid:1890 (Lock : transactionid) ->pid:1888
     40%             |     403 | pid:1913 (Lock : transactionid) ->pid:1890 (Lock : transactionid) ->pid:1888
     3%              |      33 | pid:1913 (Lock : transactionid) ->pid:1890
    (3 rows)

As you can see the first two chains are still displayed (as the **query does not filter on ash\_time**) but are not waiting anymore (seconds does not increase) while the last one (new one) is still waiting (seconds increase).

### Remarks

-   This query and active session history usage is 100% inspired by [Tanel Poder](https://blog.tanelpoder.com/2013/11/06/diagnosing-buffer-busy-waits-with-the-ash_wait_chains-sql-script-v0-2/) [oracle script](https://github.com/tanelpoder/tpt-oracle/blob/master/ash/ash_wait_chains.sql)
-   The query in this blog post does not filter on ash\_time, but it would be easy to adapt to do so (so that you could drill down in a particular time window interval)
-   Franck Pachot described other usages of the pg\_active\_session\_history in those posts:
    -   [pgSentinel: the sampling approach for PostgreSQL](https://blog.dbi-services.com/pgsentinel-the-sampling-approach-for-postgresql/)
    -   [Drilling down the pgSentinel Active Session History](https://blog.dbi-services.com/drilling-down-the-pgsentinel-active-session-history/)
-   You can find the script used in this post (as well as others and usage examples) in [this github repository](https://github.com/pgsentinel/pg_ash_scripts)

### Conclusion

We have seen how the blocking information part of the pg\_active\_session\_history view could help to drill down in case of blocking activity.

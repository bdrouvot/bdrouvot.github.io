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
<p>While the <a href="https://github.com/pgsentinel/pgsentinel" target="_blank" rel="noopener">active session history extension for PostgreSQL</a> is still in beta, some information is added to it.</p>
<p>The pg_active_session_history view is currently made of:</p>
<pre style="padding-left:30px;">                   View "public.pg_active_session_history"
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
</pre>
<p>You could see it as samplings of <code>pg_stat_activity</code> providing more information:</p>
<ul>
<li><code>ash_time</code>: the sampling time</li>
<li><code>top_level_query</code>: the top level statement (in case PL/pgSQL is used)</li>
<li><code>query</code>: the statement being executed (not normalised, as it is in <code>pg_stat_statements</code>, means you see the values)</li>
<li><code>cmdtype</code>: the statement type (SELECT,UPDATE,INSERT,DELETE,UTILITY,UNKNOWN,NOTHING)</li>
<li><code>queryid</code>: the queryid of the statement which links to pg_stat_statements</li>
<li><code>blockers</code>: the number of blockers</li>
<li><code>blockerpid</code>: the pid of the blocker (if blockers = 1), the pid of one blocker (if blockers &gt; 1)</li>
<li><code>blocker_state</code>: state of the blocker (state of the blockerpid)</li>
</ul>
<p>Thanks to the queryid field you are able to link the session activity with the sql activity.</p>
<p>The information related to the blocking activity (if any) has been added recently. Why? To easily drill down in case of session being blocked.</p>
<p>Let's see how we could display some interesting information in case of blocked session(s), for examples:</p>
<ul>
<li>The wait chain</li>
<li>The seconds in wait in this chain</li>
<li>The percentage of the total wait time that this chain represents</li>
</ul>
<p>As PostgreSQL provides <a href="https://www.postgresql.org/docs/current/static/queries-with.html" target="_blank" rel="noopener">recursive query</a> and <a href="https://www.postgresql.org/docs/current/static/tutorial-window.html" target="_blank" rel="noopener">window functions</a>, let's make use of them to write this query:</p>
<p>[code language="sql"]<br />
postgres@pgu:~$ cat pg_ash_wait_chain.sql<br />
WITH RECURSIVE search_wait_chain(ash_time,pid, blockerpid, wait_event_type,wait_event,level, path)<br />
AS (<br />
          SELECT ash_time,pid, blockerpid, wait_event_type,wait_event, 1 AS level,<br />
          'pid:'||pid||' ('||wait_event_type||' : '||wait_event||') -&gt;'||'pid:'||blockerpid AS path<br />
          from pg_active_session_history WHERE blockers &gt; 0<br />
        union ALL<br />
          SELECT p.ash_time,p.pid, p.blockerpid, p.wait_event_type,p.wait_event, swc.level + 1 AS level,<br />
          'pid:'||p.pid||' ('||p.wait_event_type||' : '||p.wait_event||') -&gt;'||swc.path AS path<br />
          FROM pg_active_session_history p, search_wait_chain swc<br />
          WHERE p.blockerpid = swc.pid and p.ash_time = swc.ash_time and p.blockers &gt; 0<br />
)<br />
select round(100 * count(*) / cnt)||'%' as &quot;% of total wait&quot;,count(*) as seconds,path as wait_chain  from (<br />
        SELECT  pid,wait_event,path,sum(count) over() as cnt from (<br />
                select ash_time,level,pid,wait_event,path,count(*) as count, max(level) over(partition by ash_time,pid) as max_level<br />
                FROM search_wait_chain where level &gt; 0 group by ash_time,level,pid,wait_event,path<br />
        ) as all_wait_chain<br />
        where level=max_level<br />
) as wait_chain<br />
group by path,cnt<br />
order by count(*) desc;<br />
[/code]</p>
<p>Let's launch this query while only one session is being blocked by another one:</p>
<pre style="padding-left:30px;">postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
 % of total wait | seconds |                 wait_chain
-----------------+---------+--------------------------------------------
 100%            |      23 | pid:1890 (Lock : transactionid) -&gt;pid:1888
(1 row)</pre>
<p>It means that the pid 1890 is waiting since 23 seconds on the transactionid wait event, while being blocked by pid 1888. This wait chain represents 100% of the blocking activity time.</p>
<p>Now another session comes into the game, query the active session history view one more time:</p>
<pre style="padding-left:30px;">postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
 % of total wait | seconds |                                  wait_chain
-----------------+---------+------------------------------------------------------------------------------
 88%             |     208 | pid:1890 (Lock : transactionid) -&gt;pid:1888
 12%             |      29 | pid:1913 (Lock : transactionid) -&gt;pid:1890 (Lock : transactionid) -&gt;pid:1888
(2 rows)</pre>
<p>So we still see our first blocking chain. It is now not the only one (so represents 88% of the blocking activity time).</p>
<p>We can see a new chain that represents 12% of the blocking activity time:</p>
<ul>
<li>pid 1913 (waiting on transactionid) is blocked since 29 seconds by pid 1890 (waiting on transactionid) that is also blocked by pid 1888.</li>
</ul>
<p>Let's commit the transaction hold by pid 1888 and launch the query again 2 times:</p>
<pre style="padding-left:30px;">postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
 % of total wait | seconds |                                  wait_chain
-----------------+---------+------------------------------------------------------------------------------
 57%             |     582 | pid:1890 (Lock : transactionid) -&gt;pid:1888
 40%             |     403 | pid:1913 (Lock : transactionid) -&gt;pid:1890 (Lock : transactionid) -&gt;pid:1888
 3%              |      32 | pid:1913 (Lock : transactionid) -&gt;pid:1890
(3 rows)

postgres@pgu:~$ psql -f pg_ash_wait_chain.sql
 % of total wait | seconds |                                  wait_chain
-----------------+---------+------------------------------------------------------------------------------
57% | 582 | pid:1890 (Lock : transactionid) -\>pid:1888 40% | 403 | pid:1913 (Lock : transactionid) -\>pid:1890 (Lock : transactionid) -\>pid:1888 3% | 33 | pid:1913 (Lock : transactionid) -\>pid:1890 (3 rows)

As you can see the first two chains are still displayed (as the **query does not filter on ash\_time** ) but are not waiting anymore (seconds does not increase) while the last one (new one) is still waiting (seconds increase).

### Remarks

- This query and active session history usage is 100% inspired by [Tanel Poder](https://blog.tanelpoder.com/2013/11/06/diagnosing-buffer-busy-waits-with-the-ash_wait_chains-sql-script-v0-2/)&nbsp;[oracle script](https://github.com/tanelpoder/tpt-oracle/blob/master/ash/ash_wait_chains.sql)
- The query in this blog post does not filter on ash\_time, but it would be easy to adapt to do so (so that you could drill down in a particular time window interval)
- Franck Pachot described other usages of the pg\_active\_session\_history in those posts:
  - [pgSentinel: the sampling approach for PostgreSQL](https://blog.dbi-services.com/pgsentinel-the-sampling-approach-for-postgresql/)
  - [Drilling down the pgSentinel Active Session History](https://blog.dbi-services.com/drilling-down-the-pgsentinel-active-session-history/)
- You can find the script used in this post (as well as others and usage examples) in [this github repository](https://github.com/pgsentinel/pg_ash_scripts)

### Conclusion

We have seen how the blocking information part of the pg\_active\_session\_history view could help to drill down in case of blocking activity.


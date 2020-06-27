---
layout: post
title: 'PostgreSQL Active Session History (ash): welcome to the pg_active_session_history
  view (part of the pgsentinel extension)'
date: 2018-07-07 08:56:44.000000000 +02:00
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
  _oembed_d2729cf7a01e8a82a0c521de5fc92846: "{{unknown}}"
  _oembed_65fa179a42b838d10152b3bbf59c6439: "{{unknown}}"
  _oembed_9a99daff83d68a51f9c58b471c00a948: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="907" height="567" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_9a99daff83d68a51f9c58b471c00a948: '1530948970'
  _oembed_c20fdc8d8babb0fe9e166b30f7c321a0: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="1100" height="688" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_c20fdc8d8babb0fe9e166b30f7c321a0: '1532066495'
  _oembed_c036740387cc6a20b71203d8b149c15c: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="911" height="569" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_c036740387cc6a20b71203d8b149c15c: '1530948985'
  _thumbnail_id: '3459'
  _oembed_5d02a1b2a6c53646bfac44d494b8ab22: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="999" height="624" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_1c5743b216d96654e3844804381df43e: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="840" height="525" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_1c5743b216d96654e3844804381df43e: '1530950206'
  publicize_linkedin_url: www.linkedin.com/updates?topic=6421270589184577536
  timeline_notification: '1530950207'
  _publicize_job_id: '19755269023'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _oembed_b3a66e11c2724c0b61fe33576978e557: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="500" height="313" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_b3a66e11c2724c0b61fe33576978e557: '1530950210'
  _oembed_571c280078daae7788e617ab1dbd6d3b: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="685" height="428" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_571c280078daae7788e617ab1dbd6d3b: '1530950211'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1015504908097937408";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _oembed_time_5d02a1b2a6c53646bfac44d494b8ab22: '1531573961'
  _oembed_67e069a60e0612875b95cb94e1deb45b: <div class="embed-vimeo"><iframe src="https://player.vimeo.com/video/278781365?app_id=122963"
    width="584" height="365" frameborder="0" title="pgsentinel" webkitallowfullscreen
    mozallowfullscreen allowfullscreen></iframe></div>
  _oembed_time_67e069a60e0612875b95cb94e1deb45b: '1531575605'
  _oembed_87417f0dba89a85e917f160cf80bc1e7: <div class="embed-vimeo"><iframe title="pgsentinel"
    src="https://player.vimeo.com/video/278781365?dnt=1&app_id=122963" width="700"
    height="438" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe></div>
  _oembed_time_87417f0dba89a85e917f160cf80bc1e7: '1562217303'
  _oembed_6045d5ad665e6edc5a5575384ba0ea1a: <div class="embed-vimeo"><iframe title="pgsentinel"
    src="https://player.vimeo.com/video/278781365?dnt=1&amp;app_id=122963" width="580"
    height="363" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe></div>
  _oembed_time_6045d5ad665e6edc5a5575384ba0ea1a: '1578137197'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/07/07/postgresql-active-session-history-ash-welcome-to-the-pg_active_session_history-view-part-of-the-pgsentinel-extension/"
---
<h2>Why active session history?</h2>
<p>What if you could record and query an history of the active sessions? Would not it be useful for performance tuning activities?</p>
<p>With active session history in place you could have a look to the "near" past database activity. You could answer questions like:</p>
<ul>
<li>What wait events type were taking most time?</li>
<li>What wait events were taking most time?</li>
<li>Which application name was taking most time?</li>
<li>What was a session doing?</li>
<li>What does a SQL statement wait for?</li>
<li>How many sessions were running in CPU?</li>
<li>Which database was taking most time?</li>
<li>Which backend type was taking most time?</li>
<li>On which wait event was the session waiting for?</li>
<li>And so on....</li>
</ul>
<h2>How does it look like?</h2>
<p>Let's have a look to the pg_active_session_history view (more details on how to create it later on):</p>
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
</pre>
You could see it as samplings of&nbsp;[pg\_stat\_activity](https://www.postgresql.org/docs/current/static/monitoring-stats.html#PG-STAT-ACTIVITY-VIEW)&nbsp;(one second interval) providing more information:

- **ash\_time** : the sampling time
- **top\_level\_query** : the top level statement (in case&nbsp;PL/pgSQL is used)
- **query** : the statement being executed (not normalised, as it is in [pg\_stat\_statements](https://www.postgresql.org/docs/current/static/pgstatstatements.html), means you see the values)
- **queryid** : the queryid of the statement (the one coming from [pg\_stat\_statements](https://www.postgresql.org/docs/current/static/pgstatstatements.html))

Thanks to the queryid field you will be able to link the session activity with the sql activity (then addressing the point reported by Franck Pachot into this blog post: [PGIO, PG\_STAT\_ACTIVITY and PG\_STAT\_STATEMENTS](https://blog.dbi-services.com/pgio-pg_stat_activity-and-pg_stat_statements/))

## Installation

pgsentinel uses the&nbsp;[pg\_stat\_statements](http://www.postgresql.org/docs/current/static/pgstatstatements.html)&nbsp;extension (officially bundled with PostgreSQL) for tracking which queries get executed in your database.

So, add the following entries to your postgres.conf:

```
shared_preload_libraries = 'pg_stat_statements,pgsentinel'

# Increase the max size of the query strings Postgres records
track_activity_query_size = 2048

# Track statements generated by stored procedures as well
pg_stat_statements.track = all
```

restart the postgresql daemon and create the extension:

```
postgres@pgu:~$ /usr/local/pgsql/bin/psql -c "create extension pgsentinel;"
CREATE EXTENSION
```

Now, the pg\_active\_session\_history view has been created and the activity is being recorded in it.

## Let's see the extension in action during a 2 minutes&nbsp;[pgio](https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/) run:

[Video](https://vimeo.com/278781365)

## Remarks

- No objects are created into the PostgreSQL instance (except the view). The data being queried through the pg\_active\_session\_history view are fully in memory
- You are able to choose the maximum number of records, once reached, then the data rotate (oldest records are aged out)

## Next Steps

- The library and the source code will be available soon on github in the [pgsentinel github repository](https://github.com/pgsentinel/pgsentinel) (under the GNU Affero General Public License v3.0): please do contribute!
- Once the source code is published, more information on how to create the pgsentinel library will be available
- Once the source code is published, more information on how to define the maximum of records will be available
- The extension is named "pgsentinel" because more cool stuff will be added in it in the near future

## Conclusion

At [pgsentinel](https://www.pgsentinel.com/) we really love performance tuning activity. We will do our best to add valuable performance tuning stuff to the postgresql community: stay tuned

## Update 07/14/2018:

[The extension is now publicly available](https://bdrouvot.wordpress.com/2018/07/14/postgresql-active-session-history-extension-is-now-publicly-available/).


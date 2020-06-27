---
layout: post
title: PostgreSQL Active Session History extension testing with Docker
date: 2018-08-12 16:19:15.000000000 +02:00
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
  _thumbnail_id: '3487'
  publicize_linkedin_url: www.linkedin.com/updates?topic=6434427915127001088
  timeline_notification: '1534087159'
  _publicize_job_id: '21006093518'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1028662235462361090";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/08/12/postgresql-active-session-history-extension-testing-with-docker/"
---
<h3>Introduction</h3>
<p>You may have noticed that a beta version of the pgsentinel extension (providing active session history for postgresql) <a href="https://bdrouvot.wordpress.com/2018/07/14/postgresql-active-session-history-extension-is-now-publicly-available/" target="_blank" rel="noopener">is publicly available</a> in <a href="https://github.com/pgsentinel/pgsentinel" target="_blank" rel="noopener">this github repository</a>.</p>
<p>We can rely on docker to facilitate and automate the testing of the extension on a particular postgreSQL version (has to be &gt;= 10).</p>
<p>Let's share a <a href="https://docs.docker.com/engine/reference/builder/" target="_blank" rel="noopener">Dockerfile</a> so that we can easly build a docker image for testing pgsentinel (on a postgreSQL version of our choice).</p>
<h3>Description</h3>
<p>The dockerfile used to provision a pgsentinel testing docker image:</p>
<ul>
<li>downloads the postgresql source code (version of your choice)</li>
<li>compiles and installs it</li>
<li>downloads the pgsentinel extension</li>
<li>compiles and installs it</li>
<li>compiles and installs the pg_stat_statements extension</li>
</ul>
<h3>Dockerfile github repository</h3>
<p>The dockerfile is available in <a href="https://github.com/pgsentinel/docker_for_testing" target="_blank" rel="noopener">this github repository</a>.</p>
<h3>Usage</h3>
<p>3 arguments can be used:</p>
<ul>
<li>PG_VERSION (default: 10.5)</li>
<li>PG_ASH_SAMPLING (default: 1)</li>
<li>PG_ASH_MAX_ENTRIES (default: 10000)</li>
</ul>
<h4>Example to build an image</h4>
<p>Let's build a pgsentinel testing docker image for postgreSQL version 11beta3:</p>
<pre style="padding-left:30px;">[root@bdtdocker dockerfile]# docker build -t pgsentinel-testing -f Dockerfile-pgsentinel-testing --build-arg PG_VERSION=11beta3 --force-rm=true --no-cache=true .</pre>
<h4>Once done, let's run a container</h4>
<pre style="padding-left:30px;">[root@bdtdocker dockerfile]# docker run -d -p 5432:5432 --name pgsentinel pgsentinel-testing
</pre>
<h4>and verify that the pg_active_session_history view is available</h4>
<pre style="padding-left:30px;">[root@bdtdocker dockerfile]# docker exec -it pgsentinel psql -c "\d pg_active_session_history"
                   View "public.pg_active_session_history"
      Column      |           Type           | Collation | Nullable | Default
------------------+--------------------------+-----------+----------+---------
ash\_time | timestamp with time zone | | | datid | oid | | | datname | text | | | pid | integer | | | usesysid | oid | | | usename | text | | | application\_name | text | | | client\_addr | text | | | client\_hostname | text | | | client\_port | integer | | | backend\_start | timestamp with time zone | | | xact\_start | timestamp with time zone | | | query\_start | timestamp with time zone | | | state\_change | timestamp with time zone | | | wait\_event\_type | text | | | wait\_event | text | | | state | text | | | backend\_xid | xid | | | backend\_xmin | xid | | | top\_level\_query | text | | | query | text | | | cmdtype | text | | | queryid | bigint | | | backend\_type | text | | | blockers | integer | | | blockerpid | integer | | | blocker\_state | text | | |

So that we can now test the extension behavior on the postgreSQL version of our choice (11beta3 in this example).


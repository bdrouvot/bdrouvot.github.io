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
tags: [PostgreSQL]
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

### Introduction

You may have noticed that a beta version of the pgsentinel extension (providing active session history for postgresql) [is publicly available](https://bdrouvot.wordpress.com/2018/07/14/postgresql-active-session-history-extension-is-now-publicly-available/) in [this github repository](https://github.com/pgsentinel/pgsentinel).

We can rely on docker to facilitate and automate the testing of the extension on a particular postgreSQL version (has to be &gt;= 10).

Let's share a [Dockerfile](https://docs.docker.com/engine/reference/builder/) so that we can easly build a docker image for testing pgsentinel (on a postgreSQL version of our choice).

### Description

The dockerfile used to provision a pgsentinel testing docker image:

-   downloads the postgresql source code (version of your choice)
-   compiles and installs it
-   downloads the pgsentinel extension
-   compiles and installs it
-   compiles and installs the pg\_stat\_statements extension

### Dockerfile github repository

The dockerfile is available in [this github repository](https://github.com/pgsentinel/docker_for_testing).

### Usage

3 arguments can be used:

-   PG\_VERSION (default: 10.5)
-   PG\_ASH\_SAMPLING (default: 1)
-   PG\_ASH\_MAX\_ENTRIES (default: 10000)

#### Example to build an image

Let's build a pgsentinel testing docker image for postgreSQL version 11beta3:

    [root@bdtdocker dockerfile]# docker build -t pgsentinel-testing -f Dockerfile-pgsentinel-testing --build-arg PG_VERSION=11beta3 --force-rm=true --no-cache=true .

#### Once done, let's run a container

    [root@bdtdocker dockerfile]# docker run -d -p 5432:5432 --name pgsentinel pgsentinel-testing

#### and verify that the pg\_active\_session\_history view is available

    [root@bdtdocker dockerfile]# docker exec -it pgsentinel psql -c "\d pg_active_session_history"
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

So that we can now test the extension behavior on the postgreSQL version of our choice (11beta3 in this example).

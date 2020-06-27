---
layout: post
title: PostgreSQL Active Session History extension is now publicly available
date: 2018-07-14 14:39:57.000000000 +02:00
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
  _thumbnail_id: '3459'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1018127994571894789";}}
  timeline_notification: '1531575600'
  _publicize_job_id: '20001677006'
  publicize_linkedin_url: www.linkedin.com/updates?topic=6423893675008499712
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/07/14/postgresql-active-session-history-extension-is-now-publicly-available/"
---

### Publicly available

A quick one to let you know that the pgsentinel extension providing active session history sampling is now publicly available.

You can find more details on it into [this previous blog post](https://bdrouvot.wordpress.com/2018/07/07/postgresql-active-session-history-ash-welcome-to-the-pg_active_session_history-view-part-of-the-pgsentinel-extension/) and also from some early beta testers:

-   [Active session history in PostgreSQL: Say hello to pgSentinel](https://blog.dbi-services.com/active-session-history-in-postgresql-say-hello-to-pgsentinel/)
-   [pgSentinel: the sampling approach for PostgreSQL](https://blog.dbi-services.com/pgsentinel-the-sampling-approach-for-postgresql/)

Thank you [Franck Pachot](https://twitter.com/FranckPachot) and [Daniel Westermann](https://twitter.com/westermanndanie) for beta testing and sharing.

### Where to find it?

The extension is available from the [pgsentinel github repository](https://github.com/pgsentinel/pgsentinel). Please keep in mind that at the time of this writing the extension is still in beta so may contain some bugs: don't hesitate to raise [issues at github ](https://github.com/pgsentinel/pgsentinel/issues)with your bug report.

### Where to follow pgsentinel stuff?

On the [website](https://www.pgsentinel.com/), [twitter](https://twitter.com/Pg_Sentinel) or [github](https://github.com/pgsentinel). More stuff to come, stay tuned.

### Please do contribute

If you're lacking of some functionality, then you're welcome to make pull requests.

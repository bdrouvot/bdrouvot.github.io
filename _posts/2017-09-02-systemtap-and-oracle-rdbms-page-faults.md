---
layout: post
title: 'SystemTap and Oracle RDBMS: Page Faults'
date: 2017-09-02 12:38:51.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SystemTap
tags: [oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/903945328633794560";}}
  _publicize_job_id: '8880148292'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6309711017018425344&type=U&a=N5PJ
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2017/09/02/systemtap-and-oracle-rdbms-page-faults/"
---

Introduction
------------

Now that I am able to [aggregate SytemTap probes by Oracle database](https://bdrouvot.wordpress.com/2017/06/05/systemtap-aggregate-by-database/), let's focus on page faults.

For this purpose a new SystemTap script (page\_faults\_per\_db.stp) has been created and has been added into this [github repository](https://github.com/bdrouvot/SystemTap).

page\_faults\_per\_db
---------------------

This script tracks the page faults per database. It reports the total number of page faults and splits them into Major or Minor faults as well as Read or Write access.

### Usage:

    $> stap -g ./page_faults_per_db.stp <oracle uid> <refresh time ms>

### Output example:

    $> stap -g ./page_faults_per_db.stp 54321 10000

    ---------------------------------------------------------------------------------------
    DBNAME      : READS_PFLT   WRITES_PFLT  TOTAL_PFLT   MAJOR_PFLT   MINOR_PFLT
    ---------------------------------------------------------------------------------------
    BDT         : 30418        22526        52944        22           52922
    NOT_A_DB    : 773          1088         1861         4            1858

Remarks
-------

-   In this post the word database stands for “all the foreground and background processes linked to an oracle database”.
-   In a consolidated environment, having a view per database can be very useful.
-   Should you be interested by this subject, then do read Hatem Mahmoud [post](https://mahmoudhatem.wordpress.com/2016/01/25/assessing-impact-of-major-page-fault-on-oracle-database-systemtap-in-action/).

Conclusion
----------

We are able to track and group the page faults at the database level.

---
layout: post
title: Real-Time database utilities grouped into a single script
date: 2013-01-30 17:21:45.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
- ToolKit
tags:
- oracle
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:48;}s:2:"wp";a:1:{i:0;i:15;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/01/30/real-time-database-utilities-grouped-into-a-single-script/"
---

In most of my previous posts I provided some perl scripts used to collect real-time information from the database based on cumulative views.

Those cumulative views provide a lot of useful information but are useless when real-time information is needed.

So, the idea of those utilities is more or less to take a snapshot each second (default interval) of the cumulative views and compute the differences with the previous snapshot to get real-time information.

<span style="color:#008000;">**I aggregated those perl scripts into a single one**</span> ([real\_time.pl](http://bdrouvot.wordpress.com/real_time/ "real_time")) (Click on the link, and then on the view source button and then copy/paste the source code or download it from my shared directory [here](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)) :

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   This script is oracle RAC aware: you can work on all the instances, a subset or the local one.
-   You have to set oraenv on one instance of the database you want to diagnose first.
-   The script has been tested on Linux, Unix and Windows.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's have a look to the main help:</span></span>

     ./real_time.pl -help

    Usage: ./real_time.pl [-interval] [-count] [-type] [-help]
     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter      Value                        Comment
      ---------      -------                      -------
      -type          sysstat                      Real-Time snaps extracted from gv$sysstat
                     system_event                 Real-Time snaps extracted from gv$system_event
                     event_histogram              Real-Time snaps extracted from gv$event_histogram
                     sgastat                      Real-Time snaps extracted from gv$sgastat
                     enqueue_statistics           Real-Time snaps extracted from gv$enqueue_statistics
                     librarycache                 Real-Time snaps extracted from gv$librarycache
                     segments_stats               Real-Time snaps extracted from gv$segstat
                     sess_event                   Real-Time snaps extracted from gv$session_event and gv$session
                     sess_stat                    Real-Time snaps extracted from gv$sesstat and gv$session

      -help          Print the main help or related to a type (if not empty)                                                                                

      Description:
      -----------
      Utility used to display real time informations based on cumulative views
      It basically takes a snapshot each second (default interval) of the cumulative view and computes the differences with the previous snapshot
      It is oracle RAC aware: you can work on all the instances, a subset or the local one
      You have to set oraenv on one instance of the database you want to diagnose first
      You can choose the number of snapshots to display and the time to wait between snapshots

    Example: ./real_time.pl -type=sysstat -help
    Example: ./real_time.pl -type=system_event -help

So, as you can see it allows to deal with some type of snapshots, that is to say:

    sysstat                  Real-Time snaps extracted from gv$sysstat
    system_event             Real-Time snaps extracted from gv$system_event
    event_histogram          Real-Time snaps extracted from gv$event_histogram
    sgastat                  Real-Time snaps extracted from gv$sgastat
    enqueue_statistics       Real-Time snaps extracted from gv$enqueue_statistics
    librarycache             Real-Time snaps extracted from gv$librarycache
    segments_stats           Real-Time snaps extracted from gv$segstat
    sess_event               Real-Time snaps extracted from gv$session_event and gv$session
    sess_stat                Real-Time snaps extracted from gv$sesstat and gv$session

Each type has its own help associated.

<span style="text-decoration:underline;color:#0000ff;">To get more help about a particular type, just call the help with the associated type:</span>

    ./real_time.pl -type=sysstat -help

    Usage: ./real_time.pl -type=sysstat [-interval] [-count] [-inst] [-top] [-statname] [-help]
     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter         Comment                                                      Default
      ---------         -------                                                      -------
      -INST=            ALL - Show all Instance(s)                                   ALL
                        CURRENT - Show Current Instance
                        INSTANCE_NAME,... - choose Instance(s) to display

      -TOP=             Number of rows to display                                    10
      -STATNAME=        ALL - Show all STATS (wildcard allowed)                      ALL

    Example: ./real_time.pl -type=sysstat
    Example: ./real_time.pl -type=sysstat -inst=BDTO_1,BDTO_2
    Example: ./real_time.pl -type=sysstat -statname='bytes sent via SQL*Net to client'
    Example: ./real_time.pl -type=sysstat -statname='%bytes%'

<span style="text-decoration:underline;">**Conclusion:**</span>

There is no need anymore to use the "individuals" perl scripts provided so far, as they have been grouped into this main utility.

Two perl scripts still remain outside this main utility as they are not related to database cumulative views, that is to say:

-   [exadata\_metrics.pl](http://bdrouvot.wordpress.com/exadata_metrics/ "exadata_metrics") : This perl script is used to collect exadata cell’s real-time metrics based on cumulative metrics. You can find a practical case in the [Exadata real-time metrics extracted from cumulative metrics](http://bdrouvot.wordpress.com/2012/11/27/exadata-real-time-metrics-extracted-from-cumulative-metrics/ "Exadata real-time metrics extracted from cumulative metrics") post.
-   [os\_cpu\_per\_db.pl](http://bdrouvot.wordpress.com/os_cpu_per_dp/ "os_cpu_per_db") : This perl script is used to collect real-time cpu utilization per database . You can find a practical case in the [Real-Time CPU utilization per oracle database](http://bdrouvot.wordpress.com/2012/12/04/real-time-cpu-utilization-per-oracle-database/ "Real-Time CPU utilization per oracle database") post.

Please do not hesitate to give your feedback and report any issues you may found.

<span style="color:#000000;">**Update:** real\_time.pl now contains asmiostat type (see this [post](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility"))</span>

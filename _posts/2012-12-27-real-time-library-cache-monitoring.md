---
layout: post
title: Real-Time library cache monitoring
date: 2012-12-27 16:09:45.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
- ToolKit
tags:
- perl scripts
- real-time
meta:
  _publicize_pending: '1'
  _edit_last: '40807211'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/27/real-time-library-cache-monitoring/"
---

Again the same story : Oracle provides a useful view to check librarycache statistics (v$librarycache), but this view is a cumulative one. So how to check what’s is going on my database right now with cumulative values ?

Right : You have to substract the values between 2 measures.

So to get real-time librarycache statistics, you can use the [librarycache.pl](http://bdrouvot.wordpress.com/librarycache/ "librarycache") script (click on the link and then on the view source button to copy/paste the source code) that basically takes a snapshot based on the gv$librarycache view each second (default interval) and computes the differences with the previous snapshot.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example:</span></span>

    ./librarycache.pl

    15:20:30   INST_NAME    NAMESPACE   RELOADS INVALIDATIONS   GETS    GETRATIO    PINS    PINRATIO
    15:20:30   BDT          TRIGGER         0   0           1   100.0           10  100.0
    15:20:30   BDT          TABLE/PROCEDURE 0   0           2   100.0           46  100.0
    15:20:30   BDT          BODY            0   0           5   100.0           20  100.0
    15:20:30   BDT          SQL AREA    0   0           16  88.9            213 100.0
    --------------------------------------> NEW
    15:20:31   INST_NAME    NAMESPACE   RELOADS INVALIDATIONS   GETS    GETRATIO    PINS    PINRATIO
    15:20:31   BDT          TABLE/PROCEDURE 0   0           1   100.0           24  100.0
    15:20:31   BDT          TRIGGER         0   0           1   100.0           10  100.0
    15:20:31   BDT          PIPE            0   0           1   100.0           1   100.0
    15:20:31   BDT          SQL AREA    0   0           12  92.3            162 100.0

So, as you can see 12 gets occured on the SQL AREA during the last second without invalidations or reloads. The output is sorted on the GETS column but you could choose to sort on another one.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see the help:</span></span>

    ./librarycache.pl help

    Usage: ./librarycache.pl [Interval [Count]] [inst=] [top=] [namespace=] [sort_field=]
            Default Interval : 1 second.
            Default Count    : Unlimited

      Parameter      Comment                                                           Default
      ---------      -------                                                           -------
      INST=          ALL - Show all Instance(s)                                        ALL
                     CURRENT - Show Current Instance
                     INSTANCE_NAME,... - choose Instance(s) to display

                        << Instances are only displayed in a RAC DB >>

      TOP=           Number of rows to display                                         10
      NAMESPACE=     ALL - Show all NAMESPACE (wildcard allowed)                       ALL
      SORT_FIELD=    RELOADS|INVALIDATIONS|GETS|PINS                                   GETS

    Example : ./librarycache.pl
    Example : ./librarycache.pl sort_field='PINS'
    Example : ./librarycache.pl namespace='%TRI%'

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">As usual:</span></span>

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on namespace (by default no filter is applied).
-   You can choose the column to sort the output on.
-   This script is oracle RAC aware : you can work on all the instances, a subset or the local one.
-   You have to set oraenv on one instance of the database you want to diagnose first.
-   The script has been tested on Linux, Unix and Windows.

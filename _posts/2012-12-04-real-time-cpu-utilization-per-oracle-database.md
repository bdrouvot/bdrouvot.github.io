---
layout: post
title: Real-Time CPU utilization per oracle database
date: 2012-12-04 21:22:07.000000000 +01:00
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
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:2;}s:2:"wp";a:1:{i:0;i:3;}}
  _publicize_pending: '1'
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/04/real-time-cpu-utilization-per-oracle-database/"
---

Suppose you have a lot of oracle databases running on the same host, the server is almost 100% cpu used and you would like to know which database is the top cpu consumer right now.

The well known "top" utility could help ? This is not sure in case the culprit database has a lot of active sessions as "top" works at the process level.

So, to answer this need, I wrote a perl script that collects real-time processes cpu consumption at the os level (the script does not connect to any database) and aggregate the result per database.

The script takes a snapshot of os processes each second (default interval), computes the differences with the previous snapshot and aggregate te result per database.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example</span></span> of the [os\_cpu\_per\_db.pl](http://bdrouvot.wordpress.com/os_cpu_per_dp/ "os_cpu_per_db") perl script (click on the link and then on the view source button to copy/paste the source code):

    ./os_cpu_per_db.pl 

    Collecting during 1 seconds......

                                   SUMMARY PER DB

    20:31:56       DB_NAME      CPU_SEC     NB_CPU    
    20:31:56       MYDB1        0               0.0
    20:31:56       MYDB3        2       2.0
    20:31:56       MYDB2        23      23.0

So we find out our top oracle database cpu consumer: the MYDB2 oracle database spent 23 seconds into the cpus during the last second (default collection interval) that is to say used 23 cpus.

Once you have the top database cpu consumer you can connect to it and diagnose more in depth (Example of cpu spike [on this post](http://srivenukadiyala.wordpress.com/2012/01/30/sched_noage-and-latch-contention/)).

The script does a little bit more in case the cpu issue is not related to any oracle database:

-   <span style="line-height:13px;">It can display the cpu usage per os user.</span>
-   It can display the top os PID and commands.

<span style="text-decoration:underline;color:#0000ff;">Let's see the help:</span>

    ./os_cpu_per_db.pl help                       

    Usage: ./os_cpu_per_db.pl [Interval [Count]] [top=] [displayuser=[Y|N]] [displaycmd=[Y|N]] 

            Default Interval : 1 second.
            Default Count    : Unlimited

    Parameter               Comment                 Default    
    ---------               -------                             -------         
    TOP=                    Number of rows to display               10         
    DISPLAYUSER=        REPORT ON USER TOO                      N          
    DISPLAYCMD=             REPORT ON COMMAND TOO                   N

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You have to set .oraenv on one instance first.
-   The script has been tested on Linux and Solaris.

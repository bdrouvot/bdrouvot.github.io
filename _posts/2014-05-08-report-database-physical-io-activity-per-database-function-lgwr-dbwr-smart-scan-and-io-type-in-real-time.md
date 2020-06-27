---
layout: post
title: Report database physical IO activity per database function (LGWR, DBWR, Smart
  Scan...) and IO type in real time
date: 2014-05-08 10:37:47.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1430905230496328
  _wpas_done_5536151: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/i26iAXwcZ5b
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/m4mboIxxD4
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5870104077425213440&type=U&a=oiHP
  _wpas_done_2077996: '1'
  _wpas_skip_5536151: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/05/08/report-database-physical-io-activity-per-database-function-lgwr-dbwr-smart-scan-and-io-type-in-real-time/"
---

Welcome to db\_io\_function\_<span class="skimlinks-unlinked">metrics, an utility used to display database **physical IO per database functions (DBWR, LGWR, Direct Reads, Smart Scan and so on)** real-time metrics. It basically takes a snapshot each second (default interval) from the gv$iostat\_function cumulative view and computes the delta with the previous snapshot. The utility is RAC aware (No need to be multitenant aware as con\_id=0 into gv$iostat\_function for 12.1 in any cases even if pdbs are created).</span>

<span style="text-decoration:underline;">**This utility:**</span>

-   provides useful metrics.
-   is RAC aware.
-   is fully customizable: you can aggregate the results depending on your needs.
-   does not install anything into the database.

<span style="text-decoration:underline;">**It displays the following metrics per IO Type (small, large, reads and writes) and database functions:**</span>

-   MB/s: Megabytes per second.
-   RQ/s: Requests per second.
-   Avg MB/RQ: Average Megabytes per request.
-   Avg ms/RQ: Average ms per request.

<span style="text-decoration:underline;">**At the following levels:**</span>

-   Database Instance.
-   Function

<span style="text-decoration:underline;">**Let’s see the help:**</span>

    ./db_io_function_metrics.pl -help

    Usage: ./db_io_function_metrics.pl [-interval] [-count] [-inst] [-function] [-io_type] [-show] [-display] [-sort_field] [-help]

     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter         Comment                                                                     Default
      ---------         -------                                                                     -------
      -INST=            ALL - Show all Instance(s)                                                  ALL
                        CURRENT - Show Current Instance
      -FUNCTION=        IO Function to collect (wildcard allowed)                                   ALL
      -IO_TYPE=         IO Type to collect: reads,writes,small,large                                READS
      -SHOW=            What to show: inst,function (comma separated list)                          INST
      -DISPLAY=         What to display: snap,avg (comma separated list)                            SNAP
      -SORT_FIELD=      small_reads,small_writes,large_reads,large_writes                           NONE

    Example: ./db_io_function_metrics.pl
    Example: ./db_io_function_metrics.pl  -inst=CBDT1
    Example: ./db_io_function_metrics.pl  -show=inst,function
    Example: ./db_io_function_metrics.pl  -show=inst,function -function=%Direct%
    Example: ./db_io_function_metrics.pl  -show=inst,function -function=%Direct% -io_type=small -sort_field=small_reads

<span style="text-decoration:underline;">**The main options/features are:**</span>

1.  You can choose the number of snapshots to display and the time to wait between snapshots.
2.  You can choose on which **database instance** to collect the metrics thanks to the -**INST=** parameter.
3.  You can choose on which **Database Function** to collect the metric thanks to the **-FUNCTION**=parameter (wilcard allowed).
4.  You can choose on which **IO Type** to collect the metrics thanks to the **-IO\_TYPE=** parameter.
5.  You can **aggregate the results** on the database instances and database functions level thanks to the -**SHOW=** parameter.
6.  You can display the metrics **per snapshot, the average metrics** value since the collection began (that is to say since the script has been launched) or both thanks to the -**DISPLAY=** parameter.
7.  You can sort based on the number of **small\_reads**, number of **small\_writes,** number of **large\_reads** or number of **large\_writes** thanks to the -**SORT\_FIELD=** parameter.

<span style="text-decoration:underline;">**Examples:**</span>

<span style="color:#555555;">Report the IO metrics for “small” IO per database instances and functions during a SLOB run:  
</span>

    ./db_io_function_metrics.pl -show=inst,function -io_type=small
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    09:49:55                                       SMALL R   SMALL R   SMALL W   SMALL W   Avg MB/   Avg MB/   R+W       Avg ms/
    09:49:55   INST         FUNCTION               MB/s      RQ/s      MB/s      RQ/s      SMALL R   SMALL W   IO/s      R+W IO
    09:49:55   ----------   --------------------   -------   -------   -------   -------   -------   -------   -------   -------
    09:49:55   BDT_1                               147       18817     0         2         0.008     0.000     18847     0.04
    09:49:55   BDT_1        ARCH                   0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Archive Manager        0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Buffer Cache Reads     147       18811     0         0         0.008     0.000     18843     0.04
    09:49:55   BDT_1        DBWR                   0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Data Pump              0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Direct Reads           0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Direct Writes          0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        LGWR                   0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Others                 0         6         0         2         0.000     0.000     4         0.25
    09:49:55   BDT_1        RMAN                   0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Recovery               0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Smart Scan             0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        Streams AQ             0         0         0         0         0.000     0.000     0         0.00
    09:49:55   BDT_1        XDB                    0         0         0         0         0.000     0.000     0         0.00

 

Report <span style="color:#555555;">the IO metrics for “reads” IO per database instances and functions for the RMAN function during RMAN backup:</span>

    ./db_io_function_metrics.pl -show=inst,function -io_type=reads -function=RMAN
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    10:04:58                                       SMALL R   SMALL R   LARGE R   LARGE R   Avg MB/   Avg MB/   R+W       Avg ms/
    10:04:58   INST         FUNCTION               MB/s      RQ/s      MB/s      RQ/s      SMALL R   LARGE R   IO/s      R+W IO
    10:04:58   ----------   --------------------   -------   -------   -------   -------   -------   -------   -------   ---------
    10:04:58   BDT_1                               3         159       248       62        0.019     4.0       176       1.10
    10:04:58   BDT_1        RMAN                   3         159       248       62        0.019     4.0       176       1.10

You can also report the metrics for Smart Scan operations, Data Pump operations and so on...

You can **download** it from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](http://bdrouvot.wordpress.com/db_io_function_metrics_source/ "db_io_function_metrics_source").

<span style="text-decoration:underline;">**Conclusion:**</span>

Four utilities are now available to report IO metrics in real time depending on our needs:

1.  [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm_metrics") for reads and writes metrics related to ASM.
2.  [db\_io\_metrics](http://bdrouvot.wordpress.com/db_io_metrics_script/ "db_io_metrics") for reads and writes metrics related to data files and temp files.
3.  [db\_io\_type\_metrics](http://bdrouvot.wordpress.com/db_io_type_metrics_script/ "db_io_type_metrics") for reads, writes, small, large and synchronous metrics related to data files, temp files,control files, log files, archive logs, and so on.
4.  [db\_io\_function\_metrics](http://bdrouvot.wordpress.com/db_io_function_metrics_script/ "db_io_function_metrics") for reads, writes, small and large metrics related to database functions (LGWR, DBWR, Smart Scan and so on).

---
layout: post
title: Retrieve and visualize system statistics in real time with R
date: 2013-06-05 10:31:13.000000000 +02:00
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
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:152;}s:2:"wp";a:1:{i:0;i:32;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/06/05/retrieve-and-visualize-system-statistics-in-real-time-with-r/"
---

In one of my [previous post](http://bdrouvot.wordpress.com/2013/03/27/retrieve-and-visualize-system-statistics-metrics-from-awr-with-r/ "Retrieve and visualize system statistics metrics from AWR with R") I provided a R script to retrieve and visualize system statistics from **AWR**. Now with this post I will provide a R script to retrieve and visualize the same metrics in **real time**.

For that purpose I created a R script “**graph\_real\_time\_sysstat.r**” (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory"))  that provides:

1.  A **graph** for the VALUE metric refreshed in real time.
2.  A **graph** for the VALUE PER SECOND metric refreshed in real time.
3.  A **graph** for the Histogram of VALUE refreshed in real time.
4.  A **graph** for the Histogram of VALUE PER SECOND refreshed in real time.
5.  A **pdf file** that contains those graphs.
6.  A **text file** that contains the output of the query used to build the graphs.

Basically the script **takes a snapshot based on the v$sysstat view** then computes and graphs the differences with the previous snapshot.

The graph is generated to both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**. In that particular case the pdf file contains a page per snapshot.

<span style="text-decoration:underline;">Let’s see the script in action:</span>

    ./graph_real_time_sysstat.r
    Building the thin jdbc connection string....

    host ?: bdt_host
    port ?: 1521
    service_name ?: BDT
    system password ?: donotreadthis
    Display which sysstat (no quotation marks) ?: physical reads
    Number of snapshots: 60
    Refresh interval (seconds): 20
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    Please enter any key to exit:

<span style="text-decoration:underline;">As you can see you are prompted for:</span>

-   jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database).
-   oracle system user password.
-   The statistic we want to focus on.
-   Number of snapshots.
-   Refresh Interval.

So you can **choose the number of snapshots and the graphs refresh interval**.

<span style="text-decoration:underline;">**When the number of snapshots is reached the output is like:**</span>

[<img src="{{ site.baseurl }}/assets/images/real_time_physical_reads.png" class="aligncenter size-full wp-image-1067" width="620" height="412" alt="real_time_physical_reads" />](http://bdrouvot.files.wordpress.com/2013/06/real_time_physical_reads.png)

The graphs sequence that leads to this final graph is the following (see the [real\_time\_physical\_reads](http://bdrouvot.files.wordpress.com/2013/06/real_time_physical_reads.pdf) pdf file).

<span style="text-decoration:underline;">**Remarks:**</span>

-   All the points will be graphed (No points will be moved outside the graph), even if:

-   For lisibility the X axis could contains not all the ticks.

-   The script does not create any objects into the database.

-   If you want to install R, a good staring point is into the “Getting Staring” section from this [link](http://www.r-project.org/).

-   The graphs "value" and "value\_per\_sec" should looks like the same: It may not be the case if the snaps are not sampled at regular interval (unexpected delay during the collection).

<span style="text-decoration:underline;">**Conclusion:**</span>

We now have 4 scripts at our disposal to retrieve and display with R:

-   <span style="text-decoration:underline;">for a particular **system statistic**:</span>

1.  real time metrics (script described into this post)
2.  AWR based metrics (script described [this post](http://bdrouvot.wordpress.com/2013/03/27/retrieve-and-visualize-system-statistics-metrics-from-awr-with-r/ "Retrieve and visualize system statistics metrics from AWR with R"))

-   <span style="text-decoration:underline;">and also for a particular **wait event**:</span>

1.  real time metrics (script described [this post](http://bdrouvot.wordpress.com/2013/06/04/retrieve-and-visualize-in-real-time-wait-events-metrics-with-r/ "Retrieve and visualize in real time wait events metrics with R"))
2.  AWR based metrics (script described [this post](http://bdrouvot.wordpress.com/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/ "Retrieve and visualize wait events metrics from AWR with R"))

---
layout: post
title: Retrieve and visualize in real time wait events metrics with R
date: 2013-06-04 16:26:38.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- R scripts
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:152;}s:2:"wp";a:1:{i:0;i:32;}}
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/06/04/retrieve-and-visualize-in-real-time-wait-events-metrics-with-r/"
---

In one of my [previous post](http://bdrouvot.wordpress.com/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/ "Retrieve and visualize wait events metrics from AWR with R")  I provided a R script to retrieve and visualize wait events metrics from **AWR. **Now with this post I will provide a R script to retrieve and visualize the same metrics in **real time**.

For that purpose I created a R script "**graph\_real\_time\_event.r**"  (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) that provides:

1.  A **graph** for the TIME\_WAITED\_MS metric refreshed in real time.
2.  A **graph** for the NB\_WAITS metric refreshed in real time.
3.  A **graph** for the MS\_PER\_WAIT metric refreshed in real time.
4.  A **graph** for the Histogram of MS\_PER\_WAIT refreshed in real time.
5.  A **pdf file** that contains those graphs.
6.  A **text file** that contains the output of the query used to build the graphs.

Basically the script **takes a snapshot based on the v$system\_event view** then computes and graphs the differences with the previous snapshot.

The graph is generated to both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**. In that particular case the pdf file contains a page per snapshot.

<span style="text-decoration:underline;">Let's see the script in action:</span>

    ./graph_real_time_event.r
    Building the thin jdbc connection string....

    host ?: bdt_host
    port ?: 1521
    service_name ?: BDT
    system password ?: donotreadthis
    Display which system event (no quotation marks) ?: db file sequential read
    Number of snapshots: 60
    Refresh interval (seconds): 2
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    Please enter any key to exit:

<span style="text-decoration:underline;">As you can see you are prompted for:</span>

-   jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database).
-   oracle system user password.
-   The wait event we want to focus on.
-   Number of snapshots.
-   Refresh Interval.

So you can **choose the number of snapshots and the graphs refresh interval**.

<span style="text-decoration:underline;">**When the number of snapshots is reached the output is like:**</span>

<img src="{{ site.baseurl }}/assets/images/real_time_db_file_sequential_read.png" class="aligncenter size-full wp-image-1043" width="620" height="409" alt="real_time_db_file_sequential_read" />

The graphs sequence that leads to this final graph is the following (see the [real\_time\_db\_file\_sequential\_read](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_file_sequential_read.pdf) pdf file).

<span style="text-decoration:underline;">**Remarks:**</span>

-   All the points will be graphed (No points will be moved outside the graph), even if:
-   For lisibility the X axis could contains not all the ticks.
-   The script does not create any objects into the database.
-   If you want to install R, a good staring point is into the "Getting Staring" section from this [link](http://www.r-project.org/).
-   If you just want a text output in real time then see this [blog post](http://bdrouvot.wordpress.com/2012/11/20/measure-oracle-real-time-io-performance/).

<span style="text-decoration:underline;">**Conclusion:**</span>

We are now able to retrieve and display wait events metrics with R from AWR (see [the previous post](http://bdrouvot.wordpress.com/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/ "Retrieve and visualize wait events metrics from AWR with R")) and in real time.

Now that I am able to graph in real time with R, my next work is to graph in real time the metrics coming from my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility"). I'll keep you posted.

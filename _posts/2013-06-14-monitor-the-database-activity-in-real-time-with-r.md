---
layout: post
title: Monitor the database activity in real time with R
date: 2013-06-14 14:31:56.000000000 +02:00
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
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:161;}s:2:"wp";a:1:{i:0;i:32;}}
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/06/14/monitor-the-database-activity-in-real-time-with-r/"
---

A quick post to let you know that I just finished a [R](http://www.r-project.org/) script to monitor the database activity in real time.

The "**graph\_real\_time\_db\_activity.r**" script (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) basically **takes a snapshot based on the v$system\_event **view then computes and graphs the differences with the previous snapshot.

<span style="text-decoration:underline;">One graph refreshed in real time is provided. It contains:</span>

-   A sub-graph for the time waited (in ms) per wait class.

[<img src="{{ site.baseurl }}/assets/images/real_time_db_activity_time_values.png" class="aligncenter size-full wp-image-1085" width="620" height="184" alt="real_time_db_activity_time_values" />](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_time_values.png)

-   A sub-graph for the wait events distribution of the wait class having the max time waited during the last snap.

[<img src="{{ site.baseurl }}/assets/images/real_time_db_activity_events_distribution.png" class="aligncenter size-full wp-image-1086" width="526" height="364" alt="real_time_db_activity_events_distribution" />](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_events_distribution.png)

-   A sub-graph for the wait class distribution since the script has been launched.

[<img src="{{ site.baseurl }}/assets/images/real_time_db_activity_wait_class_distribution.png" class="aligncenter size-full wp-image-1087" width="472" height="365" alt="real_time_db_activity_wait_class_distribution" />](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_wait_class_distribution.png)

<span style="text-decoration:underline;">The script also provides:</span>

-   a text file that contains the snaps computations.
-   a pdf file that contains the final graph.

As you can see, for a better understanding of the database behavior, I also included a fake "CPU"  wait class (coming from the **v$sys\_time\_model** view) as suggested by Guy Harrison into this [blog post](http://guyharrison.typepad.com/oracleguy/2006/09/10g_time_model_.html).

The graph is generated to both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**.

<span style="text-decoration:underline;">Let’s see the script in action:</span>

    ./graph_real_time_db_activity.r
    Building the thin jdbc connection string....

    host ?:BDT_HOST
    port ?:1521
    service_name ?: BDT
    system password ?:donotreadthis
    Number of snapshots:50
    Refresh interval (seconds):2
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    Please enter any key to exit:

<span style="text-decoration:underline;">As you can see you are prompted for:</span>

-   jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database).
-   oracle system user password.
-   Number of snapshots.
-   Refresh Interval.

So you can **choose the number of snapshots and the graph refresh interval**.

<span style="text-decoration:underline;">**The output is like:**</span>

[<img src="{{ site.baseurl }}/assets/images/real_time_db_activity_wait_class_all.png" class="aligncenter size-full wp-image-1090" width="620" height="395" alt="real_time_db_activity_wait_class_all" />](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_wait_class_all.png)

<span style="text-decoration:underline;">**Remarks:**</span>

-   The script does not create any objects into the database.
-   If you want to install R, a good staring point is into the “Getting Staring” section of this [link](http://www.r-project.org/).
-   Now that I am able to graph in real time with R, my next work is to graph in real time the metrics coming from my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility"). I’ll keep you posted.

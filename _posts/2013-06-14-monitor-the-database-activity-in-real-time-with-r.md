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
A quick post to let you know that I just finished a&nbsp;[R](http://www.r-project.org/)&nbsp;script to monitor the database activity in real time.

The " **graph\_real\_time\_db\_activity.r**" script (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) basically **takes a snapshot based on the v$system\_event&nbsp;** view then computes and graphs the differences with the previous snapshot.

One graph refreshed in real time is provided. It contains:

- A sub-graph for the time waited (in ms) per wait class.

[![real_time_db_activity_time_values]({{ site.baseurl }}/assets/images/real_time_db_activity_time_values.png)](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_time_values.png)

- A sub-graph for the wait events distribution of the wait class having the max time waited during the last snap.

[![real_time_db_activity_events_distribution]({{ site.baseurl }}/assets/images/real_time_db_activity_events_distribution.png)](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_events_distribution.png)

- A sub-graph for the&nbsp;wait class distribution since the script has been launched.

[![real_time_db_activity_wait_class_distribution]({{ site.baseurl }}/assets/images/real_time_db_activity_wait_class_distribution.png)](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_wait_class_distribution.png)

The script also provides:

- a text file that contains the snaps computations.
- a pdf file that contains the final graph.

As you can see, for a better understanding of the database&nbsp;behavior, I also included a fake "CPU" &nbsp;wait class (coming from the **v$sys\_time\_model** view)&nbsp;as suggested by&nbsp;Guy Harrison into this [blog post](http://guyharrison.typepad.com/oracleguy/2006/09/10g_time_model_.html).

The graph is generated to both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**.

Let’s see the script in action:

```
./graph\_real\_time\_db\_activity.r Building the thin jdbc connection string.... host ?:BDT\_HOST port ?:1521 service\_name ?: BDT system password ?:donotreadthis Number of snapshots:50 Refresh interval (seconds):2 Loading required package: methods Loading required package: DBI Loading required package: rJava Please enter any key to exit:
```

As you can see you are prompted for:

- jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database).
- oracle system user password.
- Number of snapshots.
- Refresh Interval.

So you can **choose the number of snapshots and the graph refresh interval**.

**The output is like:**

[![real_time_db_activity_wait_class_all]({{ site.baseurl }}/assets/images/real_time_db_activity_wait_class_all.png)](http://bdrouvot.files.wordpress.com/2013/06/real_time_db_activity_wait_class_all.png)

**Remarks:**

- The script does not create any objects into the database.
- If you want to install R, a good staring point is into the “Getting Staring” section of this [link](http://www.r-project.org/).
- Now that I am able to graph in real time with R, my next work is to graph in real time the metrics coming from my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility"). I’ll keep you posted.

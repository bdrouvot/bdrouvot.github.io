---
layout: post
title: Retrieve and visualize wait events metrics from AWR with R
date: 2013-03-26 18:00:32.000000000 +01:00
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
permalink: "/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/"
---
<p>In this post I will provide a way to retrieve wait events metrics from AWR and to display graphically those metrics thanks to <a href="http://www.r-project.org/" target="_blank">R</a> over a period of time.</p>
<p><span style="color:#0000ff;"><strong><span style="text-decoration:underline;">Why R ?:</span></strong></span>  Because R is a <strong><span style="color:#0000ff;">powerful</span> </strong>tool for statistical analysis with graphing and plotting packages built in. Furthermore, <strong><span style="color:#0000ff;">R can connect to Oracle via a JDBC package</span></strong> which makes importing data very easy.</p>
<p>So, for a particular wait event, I'll retrieve from the dba_hist_system_event view:</p>
<ul>
<li>TIME_WAITED_MS: Time waited in ms between 2 snaps</li>
<li>TOTAL_WAITS: Number of waits between 2 snaps</li>
<li>MS_PER_WAIT: Avg wait time in ms between 2 snaps</li>
</ul>
<p><span style="text-decoration:underline;">As those metrics are cumulative ones, I need to compute the difference between 2 snaps that way:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat check_awr_event.sql<br />
set linesi 220;</p>
<p>alter session set nls_date_format='DD-MM-YYYY HH24:MI:SS';</p>
<p>col BEGIN_INTERVAL_TIME format a28<br />
col event_name format a40<br />
col WAIT_CLASS format a20<br />
set pagesi 999</p>
<p>select distinct(WAIT_CLASS) from v$system_event;</p>
<p>select e.WAIT_CLASS,e.event_name,s.begin_interval_time,e.TOTAL_WAITS,e.TIME_WAITED_MS,e.TIME_WAITED_MS / TOTAL_WAITS &quot;MS_PER_WAIT&quot;<br />
from<br />
(<br />
select instance_number,snap_id,WAIT_CLASS,event_name,<br />
total_waits - first_value(total_waits) over (partition by event_name order by snap_id rows 1 preceding) &quot;TOTAL_WAITS&quot;,<br />
(time_waited_micro - first_value(time_waited_micro) over (partition by event_name order by snap_id rows 1 preceding))/1000 &quot;TIME_WAITED_MS&quot;<br />
from<br />
dba_hist_system_event<br />
where<br />
WAIT_CLASS like nvl('&amp;WAIT_CLASS',WAIT_CLASS)<br />
and event_name like nvl('&amp;event_name',event_name)<br />
and instance_number = (select instance_number from v$instance)<br />
) e, dba_hist_snapshot s<br />
where e.TIME_WAITED_MS &gt; 0<br />
and e.instance_number=s.instance_number<br />
and e.snap_id=s.snap_id<br />
and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-&amp;sysdate_nb_day_begin_interval+1)<br />
and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-&amp;sysdate_nb_day_end_interval+1) order by 1,2,3;<br />
[/code]</p>
<p>I use the "<strong>partition by event_name order by snap_id rows 1 preceding</strong>" to compute the difference between snaps per event.</p>
<p><span style="text-decoration:underline;">The output is like:</span></p>
<pre style="padding-left:30px;">SQL&gt; @check_awr_event.sql

Session altered.

WAIT_CLASS
--------------------
Administrative
Application
Commit
Concurrency
Configuration
Idle
Network
Other
System I/O
User I/O

10 rows selected.

Enter value for wait_class:
old  10: WAIT_CLASS like nvl('&amp;WAIT_CLASS',WAIT_CLASS)
new  10: WAIT_CLASS like nvl('',WAIT_CLASS)
Enter value for event_name: db file sequential read
old  11: and event_name like nvl('&amp;event_name',event_name)
new  11: and event_name like nvl('db file sequential read',event_name)
Enter value for sysdate_nb_day_begin_interval: 7
old  17: and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-&amp;sysdate_nb_day_begin_interval+1)
new  17: and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-7+1)
Enter value for sysdate_nb_day_end_interval: 0
old  18: and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-&amp;sysdate_nb_day_end_interval+1) order by 1,2,3
new  18: and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-0+1) order by 1,2,3

WAIT_CLASS           EVENT_NAME                               BEGIN_INTERVAL_TIME          TOTAL_WAITS TIME_WAITED_MS MS_PER_WAIT
-------------------- ---------------------------------------- ---------------------------- ----------- -------------- -----------
User I/O db file sequential read 20-MAR-13 12.00.45.270 AM 286608 271639.345 .947773073 User I/O db file sequential read 20-MAR-13 12.20.49.821 AM 32759 125296.732 3.82480332 User I/O db file sequential read 20-MAR-13 12.40.54.540 AM 4404 8946.577 2.03146617 User I/O db file sequential read 20-MAR-13 01.00.58.981 AM 3617 4737.182 1.3096992 User I/O db file sequential read 20-MAR-13 01.20.03.039 AM 22624 94671.254 4.18454977 User I/O db file sequential read 20-MAR-13 01.40.07.163 AM 94323 181118.282 1.92019213 User I/O db file sequential read 20-MAR-13 02.00.11.636 AM 119458 317204.205 2.65536176 User I/O db file sequential read 20-MAR-13 02.20.16.040 AM 81720 212678.865 2.60253139 User I/O db file sequential read 20-MAR-13 02.40.20.827 AM 61664 120446.947 1.9532782 User I/O db file sequential read 20-MAR-13 03.00.25.531 AM 92493 110715.902 1.19701926 User I/O db file sequential read 20-MAR-13 03.20.29.923 AM 2692 6102.149 2.26677155

So if you use this sql, you'll be able to see for a particular event its historical behaviour. That's fine and I used it a lot of times.

But I like also to have a graphical view of what's going on, and that is exactly where R comes into play.

I created a R script named: &nbsp; **graph\_awr\_event.r** &nbsp;(You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) that provides:

      1. A **graph** for the TIME\_WAITED\_MS metric over the period of time
      2. A **graph** for the NB\_WAITS metric over the period of time
      3. A **graph** for the MS\_PER\_WAIT metric over the period of time
      4. A **graph** for the Histogram of MS\_PER\_WAIT over the period of time
      5. A **pdf file** that contains those graphs
      6. A **text file** that contains the metrics used to build the graphs

The graphs will come from both outputs (X11 and the pdf file). In case the X11 environment&nbsp;does not work, **the pdf file is generated anyway**.

As a graphical view is better to understand, let's have a look how it works and what the display is:

For example, let's focus on the "db file sequential read" wait event over the last 7 days that way:

```
./graph\_awr\_event.r Building the thin jdbc connection string.... host ?: bdt\_host port ?: 1521 service\_name ?: bdt system password ?: XXXXXXXX Display which event (no quotation marks) ?: db file sequential read Please enter nb\_day\_begin\_interval: 7 Please enter nb\_day\_end\_interval: 0 Loading required package: methods Loading required package: DBI Loading required package: rJava [1] 11 Please enter any key to exit:
```

The output will be like:

[![db_file_sequential_read]({{ site.baseurl }}/assets/images/db_file_sequential_read.png)](http://bdrouvot.files.wordpress.com/2013/03/db_file_sequential_read.png)

and the [db\_file\_sequential\_read](http://bdrouvot.files.wordpress.com/2013/03/db_file_sequential_read.pdf)&nbsp;pdf&nbsp;&nbsp;file will be generated as well.

As you can see, you are prompted for:

- jdbc thin "like" details to connect to the database (You can launch the R script outside the host hosting the database)
- oracle system user password
- The wait event we want to focus on
- Number of days to go back as the time frame starting point
- Number of days to go back &nbsp;as the time frame ending point

Remarks:

- You need to purchase the Diagnostic Pack in order to be allowed to query the AWR repository.

- If the script has been launched with X11 not working properly, you'll get:

```
Loading required package: methods Loading required package: DBI Loading required package: rJava [1] "Not able to display, so only pdf generation..." Warning message: In x11(width = 15, height = 10) : unable to open connection to X11 display '' [1] 11 Please enter any key to exit:
```

But the script takes care of it and the pdf file will be generated anyway.

**Conclusion:**

We are able to display graphically AWR historical metrics for a particular wait event over a period of time with the help of a single script&nbsp;&nbsp;named: &nbsp; **graph\_awr\_event.r** &nbsp;(You can download it from this&nbsp;[repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")).

If you don't have R installed:

- you can use the sql provided at the beginning of this post to get at least a "text" view of the historical metrics.
- Install it ;-) (See "Getting Starting" from [this link](http://www.r-project.org/))

**Updates:**

- You can do the same with system statistics (see&nbsp;[Retrieve and visualize system statistics from AWR with R](http://bdrouvot.wordpress.com/2013/03/27/retrieve-and-visualize-system-statistics-metrics-from-awr-with-r/ "Retrieve and visualize system statistics metrics from AWR with R"))
- You can do the same in real time (see [Retrieve and visualize in real time wait events metrics with R](http://bdrouvot.wordpress.com/2013/06/04/retrieve-and-visualize-in-real-time-wait-events-metrics-with-r/ "Retrieve and visualize in real time wait events metrics with R"))

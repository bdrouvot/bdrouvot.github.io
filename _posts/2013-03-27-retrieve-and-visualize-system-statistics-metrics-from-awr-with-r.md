---
layout: post
title: Retrieve and visualize system statistics metrics from AWR with R
date: 2013-03-27 12:30:28.000000000 +01:00
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
permalink: "/2013/03/27/retrieve-and-visualize-system-statistics-metrics-from-awr-with-r/"
---
<p>Into my last <a title="Retrieve and visualize wait events metrics from AWR with R" href="http://bdrouvot.wordpress.com/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/" target="_blank">post</a> I gave a way to Retrieve and visualize <strong>wait events</strong> metrics from AWR with <a href="http://www.r-project.org/" target="_blank">R</a>, now it's time for the <strong>system statistics</strong>.</p>
<p>So, for a particular system statistic, I’ll retrieve from the dba_hist_sysstat view:</p>
<ul>
<li>Its VALUE between 2 snaps</li>
<li>Its VALUE per second between 2 snaps</li>
</ul>
<p><span style="text-decoration:underline;">As the VALUE is cumulative, I need to compute the difference between 2 snaps that way:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat check_awr_stats.sql<br />
set linesi 200<br />
col BEGIN_INTERVAL_TIME format a28<br />
col stat_name format a40</p>
<p>alter session set nls_timestamp_format='YYYY/MM/DD HH24:MI:SS';<br />
alter session set nls_date_format='YYYY/MM/DD HH24:MI:SS';</p>
<p>select s.begin_interval_time,sta.stat_name,sta.VALUE,<br />
--round(((sta.VALUE)/(to_date(s.end_interval_time)-to_date(s.begin_interval_time)))/86400,2) VALUE_PER_SEC_NOT_ACCURATE,<br />
round(((sta.VALUE)/<br />
(<br />
(extract(day from s.END_INTERVAL_TIME)-extract(day from s.BEGIN_INTERVAL_TIME))*86400 +<br />
(extract(hour from s.END_INTERVAL_TIME)-extract(hour from s.BEGIN_INTERVAL_TIME))*3600 +<br />
(extract(minute from s.END_INTERVAL_TIME)-extract(minute from s.BEGIN_INTERVAL_TIME))*60 +<br />
(extract(second from s.END_INTERVAL_TIME)-extract(second from s.BEGIN_INTERVAL_TIME))<br />
)<br />
),2) VALUE_PER_SEC<br />
from<br />
(<br />
select instance_number,snap_id,stat_name,<br />
value - first_value(value) over (partition by stat_name order by snap_id rows 1 preceding) &quot;VALUE&quot;<br />
from<br />
dba_hist_sysstat<br />
where stat_name like nvl('&amp;stat_name',stat_name)<br />
and instance_number = (select instance_number from v$instance)<br />
) sta, dba_hist_snapshot s<br />
where sta.instance_number=s.instance_number<br />
and sta.snap_id=s.snap_id<br />
and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-&amp;sysdate_nb_day_begin_interval+1)<br />
and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-&amp;sysdate_nb_day_end_interval+1)<br />
order by s.begin_interval_time asc;<br />
[/code]</p>
<p>I use the "<strong>partition by stat_name order by snap_id rows 1 preceding</strong>" to compute the difference between snaps par stat_name.</p>
<p>I also use <strong>Extract</strong> to get an accurate value per second, you should read  this <a href="http://flashdba.com/2012/08/30/querying-dba_hist_snapshot-and-dba_hist_sysstat/" target="_blank">blog post</a> to understand why.</p>
<p><span style="text-decoration:underline;">The output is like:</span></p>
<pre style="padding-left:30px;">SQL&gt; @check_awr_stats.sql

Session altered.

Session altered.

Enter value for stat_name: physical reads
old  17: where stat_name like nvl('&amp;stat_name',stat_name)
new  17: where stat_name like nvl('physical reads',stat_name)
Enter value for sysdate_nb_day_begin_interval: 7
old  22: and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-&amp;sysdate_nb_day_begin_interval+1)
new  22: and s.BEGIN_INTERVAL_TIME &gt;= trunc(sysdate-7+1)
Enter value for sysdate_nb_day_end_interval: 0
old  23: and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-&amp;sysdate_nb_day_end_interval+1)
new  23: and s.BEGIN_INTERVAL_TIME &lt;= trunc(sysdate-0+1)

BEGIN_INTERVAL_TIME          STAT_NAME                                     VALUE VALUE_PER_SEC
---------------------------- ---------------------------------------- ---------- -------------
2013/03/21 00:00:12 physical reads 1363483 1132.11 2013/03/21 00:20:17 physical reads 260228 216.04 2013/03/21 00:40:21 physical reads 29573 24.56 2013/03/21 01:00:25 physical reads 231492 192.18 2013/03/21 01:20:30 physical reads 494749 410.7 2013/03/21 01:40:35 physical reads 232803 193.02 2013/03/21 02:00:41 physical reads 318803 264.66 2013/03/21 02:20:45 physical reads 1253398 1039.57 2013/03/21 02:40:51 physical reads 2064294 1711.98 2013/03/21 03:00:57 physical reads 503404 439.13 2013/03/21 03:20:03 physical reads 138052 114.59

So if you use this sql, you’ll be able to see for a particular system statistic its historical behaviour. That’s fine and I used it a lot of times.

But I like also to have a graphical view of what’s going on, and that is exactly where R comes into play.

I created a R script named: **graph\_awr\_sysstat.r** &nbsp;(You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) that provides:

1. A **graph** for the VALUE metric over the period of time
2. A **graph** for the VALUE per second metric over the period of time
3. A **graph** for the Histogram of VALUE over the period of time
4. A&nbsp; **graph** &nbsp;for the Histogram of VALUE per second over the period of time
5. A **pdf file** that contains those graphs
6. A **text file** that contains the metrics used to build the graphs

The graphs will come from both outputs (X11 and the pdf file). In case the X11 environment&nbsp;does not work, **the pdf file is generated anyway**.

As a graphical view is better to understand, let’s have a look how it works and what the display is:

For example, let’s focus on the “physical reads"&nbsp;system statistic over the last 7 days that way:

```
./graph\_awr\_sysstat.r Building the thin jdbc connection string.... host ?: bdt\_host port ?: 1521 service\_name ?: bdt system password ?: XXXXXXXX Display which sysstat (no quotation marks) ?: physical reads Please enter nb\_day\_begin\_interval: 7 Please enter nb\_day\_end\_interval: 0 Loading required package: methods Loading required package: DBI Loading required package: rJava [1] 11 Please enter any key to exit:
```

The output will be like:

[![physical_reads]({{ site.baseurl }}/assets/images/physical_reads.png)](http://bdrouvot.files.wordpress.com/2013/03/physical_reads.png)  
and the [physical\_reads](http://bdrouvot.files.wordpress.com/2013/03/physical_reads.pdf)&nbsp;pdf file will be generated as well.

As you can see, you are prompted for:

- jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database)
- oracle system user password
- The statistic we want to focus on
- Number of days to go back as the time frame starting point
- Number of days to go back &nbsp;as the time frame ending point

**Remarks:**

- You need to purchase the Diagnostic Pack in order to be allowed to query the AWR repository.

- If the script has been launched with X11 not working properly, you’ll get:

```
Loading required package: methods Loading required package: DBI Loading required package: rJava [1] "Not able to display, so only pdf generation..." Warning message: In x11(width = 15, height = 10) : unable to open connection to X11 display '' [1] 11 Please enter any key to exit:
```

But the script takes care of it and the pdf file will be generated anyway.

**Conclusion:**

We are able to display graphically AWR historical metrics for a particular statistic over a period of time with the help of a single script&nbsp;&nbsp;named: &nbsp; **graph\_awr\_sysstat.r&nbsp;** (You can download it from this&nbsp;[repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")).

If you don’t have R installed:

- you can use the sql provided at the beginning of this post to get at least a “text” view of the historical metrics.
- Install it :-)&nbsp;(See “Getting Starting” from [this link](http://www.r-project.org/))

**Update:** If you want to see the same metrics in real time then you could have a look to this [post](http://bdrouvot.wordpress.com/2013/06/05/retrieve-and-visualize-system-statistics-in-real-time-with-r/ "Retrieve and visualize system statistics in real time with R").


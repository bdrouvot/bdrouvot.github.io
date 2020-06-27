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

Into my last [post](http://bdrouvot.wordpress.com/2013/03/26/retrieve-and-visualize-wait-events-metrics-from-awr-with-r/ "Retrieve and visualize wait events metrics from AWR with R") I gave a way to Retrieve and visualize **wait events** metrics from AWR with [R](http://www.r-project.org/), now it's time for the **system statistics**.

So, for a particular system statistic, I’ll retrieve from the dba\_hist\_sysstat view:

-   Its VALUE between 2 snaps
-   Its VALUE per second between 2 snaps

<span style="text-decoration:underline;">As the VALUE is cumulative, I need to compute the difference between 2 snaps that way:</span>

```
SQL&gt; !cat check\_awr\_stats.sql  
set linesi 200  
col BEGIN\_INTERVAL\_TIME format a28  
col stat\_name format a40

alter session set nls\_timestamp\_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls\_date\_format='YYYY/MM/DD HH24:MI:SS';

select s.begin\_interval\_time,sta.stat\_name,sta.VALUE,  
--round(((sta.VALUE)/(to\_date(s.end\_interval\_time)-to\_date(s.begin\_interval\_time)))/86400,2) VALUE\_PER\_SEC\_NOT\_ACCURATE,  
round(((sta.VALUE)/  
(  
(extract(day from s.END\_INTERVAL\_TIME)-extract(day from s.BEGIN\_INTERVAL\_TIME))\*86400 +  
(extract(hour from s.END\_INTERVAL\_TIME)-extract(hour from s.BEGIN\_INTERVAL\_TIME))\*3600 +  
(extract(minute from s.END\_INTERVAL\_TIME)-extract(minute from s.BEGIN\_INTERVAL\_TIME))\*60 +  
(extract(second from s.END\_INTERVAL\_TIME)-extract(second from s.BEGIN\_INTERVAL\_TIME))  
)  
),2) VALUE\_PER\_SEC  
from  
(  
select instance\_number,snap\_id,stat\_name,  
value - first\_value(value) over (partition by stat\_name order by snap\_id rows 1 preceding) "VALUE"  
from  
dba\_hist\_sysstat  
where stat\_name like nvl('&stat\_name',stat\_name)  
and instance\_number = (select instance\_number from v$instance)  
) sta, dba\_hist\_snapshot s  
where sta.instance\_number=s.instance\_number  
and sta.snap\_id=s.snap\_id  
and s.BEGIN\_INTERVAL\_TIME &gt;= trunc(sysdate-&sysdate\_nb\_day\_begin\_interval+1)  
and s.BEGIN\_INTERVAL\_TIME &lt;= trunc(sysdate-&sysdate\_nb\_day\_end\_interval+1)  
order by s.begin\_interval\_time asc;  
```

I use the "**partition by stat\_name order by snap\_id rows 1 preceding**" to compute the difference between snaps par stat\_name.

I also use **Extract** to get an accurate value per second, you should read  this [blog post](http://flashdba.com/2012/08/30/querying-dba_hist_snapshot-and-dba_hist_sysstat/) to understand why.

<span style="text-decoration:underline;">The output is like:</span>

    SQL> @check_awr_stats.sql

    Session altered.

    Session altered.

    Enter value for stat_name: physical reads
    old  17: where stat_name like nvl('&stat_name',stat_name)
    new  17: where stat_name like nvl('physical reads',stat_name)
    Enter value for sysdate_nb_day_begin_interval: 7
    old  22: and s.BEGIN_INTERVAL_TIME >= trunc(sysdate-&sysdate_nb_day_begin_interval+1)
    new  22: and s.BEGIN_INTERVAL_TIME >= trunc(sysdate-7+1)
    Enter value for sysdate_nb_day_end_interval: 0
    old  23: and s.BEGIN_INTERVAL_TIME <= trunc(sysdate-&sysdate_nb_day_end_interval+1)
    new  23: and s.BEGIN_INTERVAL_TIME <= trunc(sysdate-0+1)

    BEGIN_INTERVAL_TIME          STAT_NAME                                     VALUE VALUE_PER_SEC
    ---------------------------- ---------------------------------------- ---------- -------------
    2013/03/21 00:00:12          physical reads                              1363483       1132.11
    2013/03/21 00:20:17          physical reads                               260228        216.04
    2013/03/21 00:40:21          physical reads                                29573         24.56
    2013/03/21 01:00:25          physical reads                               231492        192.18
    2013/03/21 01:20:30          physical reads                               494749         410.7
    2013/03/21 01:40:35          physical reads                               232803        193.02
    2013/03/21 02:00:41          physical reads                               318803        264.66
    2013/03/21 02:20:45          physical reads                              1253398       1039.57
    2013/03/21 02:40:51          physical reads                              2064294       1711.98
    2013/03/21 03:00:57          physical reads                               503404        439.13
    2013/03/21 03:20:03          physical reads                               138052        114.59

So if you use this sql, you’ll be able to see for a particular system statistic its historical behaviour. That’s fine and I used it a lot of times.

But I like also to have a graphical view of what’s going on, and that is exactly where R comes into play.

I created a R script named: **graph\_awr\_sysstat.r** (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) that provides:

1.  A **graph** for the VALUE metric over the period of time
2.  A **graph** for the VALUE per second metric over the period of time
3.  A **graph** for the Histogram of VALUE over the period of time
4.  A **graph** for the Histogram of VALUE per second over the period of time
5.  A **pdf file** that contains those graphs
6.  A **text file** that contains the metrics used to build the graphs

The graphs will come from both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**.

<span style="text-decoration:underline;">As a graphical view is better to understand, let’s have a look how it works and what the display is:</span>

For example, let’s focus on the “physical reads" system statistic over the last 7 days that way:

    ./graph_awr_sysstat.r
    Building the thin jdbc connection string....

    host ?: bdt_host
    port ?: 1521
    service_name ?: bdt
    system password ?: XXXXXXXX
    Display which sysstat (no quotation marks) ?: physical reads
    Please enter nb_day_begin_interval: 7
    Please enter nb_day_end_interval: 0
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    [1] 11
    Please enter any key to exit:

<span style="text-decoration:underline;">The output will be like:</span>

[<img src="{{ site.baseurl }}/assets/images/physical_reads.png" class="aligncenter size-full wp-image-888" width="620" height="407" alt="physical_reads" />](http://bdrouvot.files.wordpress.com/2013/03/physical_reads.png)  
and the [physical\_reads](http://bdrouvot.files.wordpress.com/2013/03/physical_reads.pdf) pdf file will be generated as well.

<span style="text-decoration:underline;">As you can see, you are prompted for:</span>

-   jdbc thin “like” details to connect to the database (You can launch the R script outside the host hosting the database)
-   oracle system user password
-   The statistic we want to focus on
-   Number of days to go back as the time frame starting point
-   Number of days to go back  as the time frame ending point

**<span style="text-decoration:underline;">Remarks:</span>**

\- You need to purchase the Diagnostic Pack in order to be allowed to query the AWR repository.

\- If the script has been launched with X11 not working properly, you’ll get:

    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    [1] "Not able to display, so only pdf generation..."
    Warning message:
    In x11(width = 15, height = 10) :
      unable to open connection to X11 display ''
    [1] 11
    Please enter any key to exit:

But the script takes care of it and the pdf file will be generated anyway.

<span style="text-decoration:underline;">**Conclusion:**</span>

We are able to display graphically AWR historical metrics for a particular statistic over a period of time with the help of a single script  named:  **graph\_awr\_sysstat.r **(You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")).

If you don’t have R installed:

-   you can use the sql provided at the beginning of this post to get at least a “text” view of the historical metrics.
-   Install it :-) (See “Getting Starting” from [this link](http://www.r-project.org/))

**Update:** If you want to see the same metrics in real time then you could have a look to this [post](http://bdrouvot.wordpress.com/2013/06/05/retrieve-and-visualize-system-statistics-in-real-time-with-r/ "Retrieve and visualize system statistics in real time with R").

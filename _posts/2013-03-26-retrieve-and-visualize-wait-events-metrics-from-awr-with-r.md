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

In this post I will provide a way to retrieve wait events metrics from AWR and to display graphically those metrics thanks to [R](http://www.r-project.org/) over a period of time.

<span style="color:#0000ff;">**<span style="text-decoration:underline;">Why R ?:</span>**</span>  Because R is a **<span style="color:#0000ff;">powerful</span>** tool for statistical analysis with graphing and plotting packages built in. Furthermore, **<span style="color:#0000ff;">R can connect to Oracle via a JDBC package</span>** which makes importing data very easy.

So, for a particular wait event, I'll retrieve from the dba\_hist\_system\_event view:

-   TIME\_WAITED\_MS: Time waited in ms between 2 snaps
-   TOTAL\_WAITS: Number of waits between 2 snaps
-   MS\_PER\_WAIT: Avg wait time in ms between 2 snaps

<span style="text-decoration:underline;">As those metrics are cumulative ones, I need to compute the difference between 2 snaps that way:</span>

```
SQL&gt; !cat check\_awr\_event.sql  
set linesi 220;

alter session set nls\_date\_format='DD-MM-YYYY HH24:MI:SS';

col BEGIN\_INTERVAL\_TIME format a28  
col event\_name format a40  
col WAIT\_CLASS format a20  
set pagesi 999

select distinct(WAIT\_CLASS) from v$system\_event;

select e.WAIT\_CLASS,e.event\_name,s.begin\_interval\_time,e.TOTAL\_WAITS,e.TIME\_WAITED\_MS,e.TIME\_WAITED\_MS / TOTAL\_WAITS "MS\_PER\_WAIT"  
from  
(  
select instance\_number,snap\_id,WAIT\_CLASS,event\_name,  
total\_waits - first\_value(total\_waits) over (partition by event\_name order by snap\_id rows 1 preceding) "TOTAL\_WAITS",  
(time\_waited\_micro - first\_value(time\_waited\_micro) over (partition by event\_name order by snap\_id rows 1 preceding))/1000 "TIME\_WAITED\_MS"  
from  
dba\_hist\_system\_event  
where  
WAIT\_CLASS like nvl('&WAIT\_CLASS',WAIT\_CLASS)  
and event\_name like nvl('&event\_name',event\_name)  
and instance\_number = (select instance\_number from v$instance)  
) e, dba\_hist\_snapshot s  
where e.TIME\_WAITED\_MS &gt; 0  
and e.instance\_number=s.instance\_number  
and e.snap\_id=s.snap\_id  
and s.BEGIN\_INTERVAL\_TIME &gt;= trunc(sysdate-&sysdate\_nb\_day\_begin\_interval+1)  
and s.BEGIN\_INTERVAL\_TIME &lt;= trunc(sysdate-&sysdate\_nb\_day\_end\_interval+1) order by 1,2,3;  
```

I use the "**partition by event\_name order by snap\_id rows 1 preceding**" to compute the difference between snaps per event.

<span style="text-decoration:underline;">The output is like:</span>

    SQL> @check_awr_event.sql

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
    old  10: WAIT_CLASS like nvl('&WAIT_CLASS',WAIT_CLASS)
    new  10: WAIT_CLASS like nvl('',WAIT_CLASS)
    Enter value for event_name: db file sequential read
    old  11: and event_name like nvl('&event_name',event_name)
    new  11: and event_name like nvl('db file sequential read',event_name)
    Enter value for sysdate_nb_day_begin_interval: 7
    old  17: and s.BEGIN_INTERVAL_TIME >= trunc(sysdate-&sysdate_nb_day_begin_interval+1)
    new  17: and s.BEGIN_INTERVAL_TIME >= trunc(sysdate-7+1)
    Enter value for sysdate_nb_day_end_interval: 0
    old  18: and s.BEGIN_INTERVAL_TIME <= trunc(sysdate-&sysdate_nb_day_end_interval+1) order by 1,2,3
    new  18: and s.BEGIN_INTERVAL_TIME <= trunc(sysdate-0+1) order by 1,2,3

    WAIT_CLASS           EVENT_NAME                               BEGIN_INTERVAL_TIME          TOTAL_WAITS TIME_WAITED_MS MS_PER_WAIT
    -------------------- ---------------------------------------- ---------------------------- ----------- -------------- -----------
    User I/O             db file sequential read                  20-MAR-13 12.00.45.270 AM         286608     271639.345  .947773073
    User I/O             db file sequential read                  20-MAR-13 12.20.49.821 AM          32759     125296.732  3.82480332
    User I/O             db file sequential read                  20-MAR-13 12.40.54.540 AM           4404       8946.577  2.03146617
    User I/O             db file sequential read                  20-MAR-13 01.00.58.981 AM           3617       4737.182   1.3096992
    User I/O             db file sequential read                  20-MAR-13 01.20.03.039 AM          22624      94671.254  4.18454977
    User I/O             db file sequential read                  20-MAR-13 01.40.07.163 AM          94323     181118.282  1.92019213
    User I/O             db file sequential read                  20-MAR-13 02.00.11.636 AM         119458     317204.205  2.65536176
    User I/O             db file sequential read                  20-MAR-13 02.20.16.040 AM          81720     212678.865  2.60253139
    User I/O             db file sequential read                  20-MAR-13 02.40.20.827 AM          61664     120446.947   1.9532782
    User I/O             db file sequential read                  20-MAR-13 03.00.25.531 AM          92493     110715.902  1.19701926
    User I/O             db file sequential read                  20-MAR-13 03.20.29.923 AM           2692       6102.149  2.26677155

So if you use this sql, you'll be able to see for a particular event its historical behaviour. That's fine and I used it a lot of times.

But I like also to have a graphical view of what's going on, and that is exactly where R comes into play.

I created a R script named:  <span style="color:#0000ff;">**graph\_awr\_event.r**</span> (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")) that provides:

1.  A **graph** for the TIME\_WAITED\_MS metric over the period of time
2.  A **graph** for the NB\_WAITS metric over the period of time
3.  A **graph** for the MS\_PER\_WAIT metric over the period of time
4.  A **graph** for the Histogram of MS\_PER\_WAIT over the period of time
5.  A **pdf file** that contains those graphs
6.  A **text file** that contains the metrics used to build the graphs

The graphs will come from both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**.

<span style="text-decoration:underline;color:#0000ff;">As a graphical view is better to understand, let's have a look how it works and what the display is:</span>

For example, let's focus on the "db file sequential read" wait event over the last 7 days that way:

    ./graph_awr_event.r   
    Building the thin jdbc connection string....

    host ?: bdt_host
    port ?: 1521
    service_name ?: bdt
    system password ?: XXXXXXXX
    Display which event (no quotation marks) ?: db file sequential read
    Please enter nb_day_begin_interval: 7
    Please enter nb_day_end_interval: 0
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    [1] 11
    Please enter any key to exit:

<span style="text-decoration:underline;color:#0000ff;">The output will be like:</span>

<img src="{{ site.baseurl }}/assets/images/db_file_sequential_read.png" class="aligncenter size-full wp-image-850" width="620" height="411" alt="db_file_sequential_read" />

and the [db\_file\_sequential\_read](http://bdrouvot.files.wordpress.com/2013/03/db_file_sequential_read.pdf) pdf  file will be generated as well.

<span style="text-decoration:underline;color:#0000ff;">As you can see, you are prompted for:</span>

-   jdbc thin "like" details to connect to the database (You can launch the R script outside the host hosting the database)
-   oracle system user password
-   The wait event we want to focus on
-   Number of days to go back as the time frame starting point
-   Number of days to go back  as the time frame ending point

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Remarks:</span></span>

\- You need to purchase the Diagnostic Pack in order to be allowed to query the AWR repository.

\- If the script has been launched with X11 not working properly, you'll get:

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

**<span style="text-decoration:underline;color:#0000ff;">Conclusion:</span>**

We are able to display graphically AWR historical metrics for a particular wait event over a period of time with the help of a single script  named:  **graph\_awr\_event.r** (You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory")).

If you don't have R installed:

-   you can use the sql provided at the beginning of this post to get at least a "text" view of the historical metrics.
-   Install it ;-) (See "Getting Starting" from [this link](http://www.r-project.org/))

**Updates:**

-   You can do the same with system statistics (see [Retrieve and visualize system statistics from AWR with R](http://bdrouvot.wordpress.com/2013/03/27/retrieve-and-visualize-system-statistics-metrics-from-awr-with-r/ "Retrieve and visualize system statistics metrics from AWR with R"))
-   You can do the same in real time (see [Retrieve and visualize in real time wait events metrics with R](http://bdrouvot.wordpress.com/2013/06/04/retrieve-and-visualize-in-real-time-wait-events-metrics-with-r/ "Retrieve and visualize in real time wait events metrics with R"))

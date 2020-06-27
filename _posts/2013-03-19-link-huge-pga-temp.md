---
layout: post
title: Link huge PGA or TEMP consumption to sql_id over a period of time
date: 2013-03-19 19:30:01.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
tags: []
meta:
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _publicize_pending: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
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
permalink: "/2013/03/19/link-huge-pga-temp/"
---

Imagine you discovered that during a particular period of time a huge amount of PGA or TEMP space has been consumed by your database.

Then you want to know, if you could link this behavior to one or more sql\_id.

Ok, I am a little bit late :-) but you should have noticed that since 11.2.0.1  two useful columns have been added to the v$active\_session\_history and dba\_hist\_active\_sess\_history views:

-   **PGA\_ALLOCATED**:  Amount of PGA memory (in bytes) consumed by this session at the time this sample was taken
-   **TEMP\_SPACE\_ALLOCATED**: Amount of TEMP memory (in bytes) consumed by this session at the time this sample was taken

<span style="text-decoration:underline;color:#0000ff;">Interesting, but is it helpful to answer:</span>

-   What are the top sql\_id linked to pga consumption during a particular period of time ?
-   What are the top sql\_id linked to temp space consumption during a particular period of time ?

Coskan Gundogar gave a nice example related to a temp space issue, for a particular session [into this blog post](http://coskan.wordpress.com/2011/01/24/analysing-temp-usage-on-11gr2-temp-space-is-not-released/).

With this blog post, I just want to provide a way to generalize the computation at the instance level instead of the session one.

So, to find the top sql\_id(s) responsible of  PGA or TEMP space consumption during a particular period of time, I use:

<span style="text-decoration:underline;color:#0000ff;">For the PGA consumption:</span>

```
SQL&gt; !cat ash\_sql\_id\_pga.sql  
col percent head '%' for 99990.99  
col star for A10 head ''

accept seconds prompt "Last Seconds \[60\] : " default 60;  
accept top prompt "Top Rows \[10\] : " default 10;

select SQL\_ID,round(PGA\_MB,1) PGA\_MB,percent,rpad('\*',percent\*10/100,'\*') star  
from  
(  
select SQL\_ID,sum(DELTA\_PGA\_MB) PGA\_MB ,(ratio\_to\_report(sum(DELTA\_PGA\_MB)) over ())\*100 percent,rank() over(order by sum(DELTA\_PGA\_MB) desc) rank  
from  
(  
select SESSION\_ID,SESSION\_SERIAL\#,sample\_id,SQL\_ID,SAMPLE\_TIME,IS\_SQLID\_CURRENT,SQL\_CHILD\_NUMBER,PGA\_ALLOCATED,  
greatest(PGA\_ALLOCATED - first\_value(PGA\_ALLOCATED) over (partition by SESSION\_ID,SESSION\_SERIAL\# order by sample\_time rows 1 preceding),0)/power(1024,2) "DELTA\_PGA\_MB"  
from  
v$active\_session\_history  
where  
IS\_SQLID\_CURRENT='Y'  
and sample\_time &gt; sysdate-&seconds/86400  
order by 1,2,3,4  
)  
group by sql\_id  
having sum(DELTA\_PGA\_MB) &gt; 0  
)  
where rank &lt; (&top+1)  
order by rank  
/```

<span style="text-decoration:underline;">The output is like:</span>

    SQL> @ash_sql_id_pga.sql
    Last Seconds [60] : 3600
    Top  Rows    [10] : 10
    old  13: and sample_time > sysdate-&seconds/86400
    new  13: and sample_time > sysdate-3600/86400
    old  20: where rank < (&top+1)
    new  20: where rank < (10+1)

    SQL_ID            PGA_MB         %
    ------------- ---------- --------- ----------
    4nyd6q26dzvb2     2211.6     55.75 *****
    3s5d6gj84kban      309.8      7.81
    8pfzqzrsvjj38        171      4.31
    2bjfxk6vqc0ft      134.1      3.38
    fxr8wdgq9bmsv      115.4      2.91
    4r23u15d4c9rh      108.4      2.73
                        94.3      2.38
    g02u5ztkuv2sz       43.5      1.10
    ddr8uck5s5kp3       23.8      0.60
    10pty85f37hrb       20.5      0.52

    10 rows selected.

<span style="text-decoration:underline;color:#0000ff;">For the TEMP consumption:</span>

```
SQL&gt; !cat ash\_sql\_id\_temp.sql  
col percent head '%' for 99990.99  
col star for A10 head ''

accept seconds prompt "Last Seconds \[60\] : " default 60;  
accept top prompt "Top Rows \[10\] : " default 10;

select SQL\_ID,TEMP\_MB,percent,rpad('\*',percent\*10/100,'\*') star  
from  
(  
select SQL\_ID,sum(DELTA\_TEMP\_MB) TEMP\_MB ,(ratio\_to\_report(sum(DELTA\_TEMP\_MB)) over ())\*100 percent,rank() over(order by sum(DELTA\_TEMP\_MB) desc) rank  
from  
(  
select SESSION\_ID,SESSION\_SERIAL\#,sample\_id,SQL\_ID,SAMPLE\_TIME,IS\_SQLID\_CURRENT,SQL\_CHILD\_NUMBER,temp\_space\_allocated,  
greatest(temp\_space\_allocated - first\_value(temp\_space\_allocated) over (partition by SESSION\_ID,SESSION\_SERIAL\# order by sample\_time rows 1 preceding),0)/power(1024,2) "DELTA\_TEMP\_MB"  
from  
v$active\_session\_history  
where  
IS\_SQLID\_CURRENT='Y'  
and sample\_time &gt; sysdate-&seconds/86400  
order by 1,2,3,4  
)  
group by sql\_id  
having sum(DELTA\_TEMP\_MB) &gt; 0  
)  
where rank &lt; (&top+1)  
order by rank  
/  
```

<span style="text-decoration:underline;">The output is like:</span>

    SQL> @ash_sql_id_temp.sql
    Last Seconds [60] : 3600
    Top  Rows    [10] : 
    old  13: and sample_time > sysdate-&seconds/86400
    new  13: and sample_time > sysdate-3600/86400
    old  19: where rank < (&top+1)
    new  19: where rank < (10+1)

    SQL_ID           TEMP_MB         %
    ------------- ---------- --------- ----------
    by714720ajxwk          2     50.00 *****
    c2jdkwzndq685          1     25.00 **
    5h7w8ykwtb2xt          1     25.00 **

<span style="text-decoration:underline;color:#0000ff;">Basically:</span>

-   The SQL computes, for each session, the PGA or TEMP space allocated between two active session history samples  thanks to "**over (partition by SESSION\_ID,SESSION\_SERIAL\# order by sample\_time rows 1 preceding)**".
-   Those computed values are linked to the "active" sql\_id observed during the sampling.
-   Then, it sums per sql\_id those computed values and display the top sql\_id(s).

**So, we are now able to know which sql\_id are responsible of huge PGA or TEMP consumption during a certain period of time.**

<span style="text-decoration:underline;color:#0000ff;">**Important remarks:**</span>

1.  You can also query the dba\_hist\_active\_sess\_history view but bear in mind that it is no so accurate as only a subset of the rows coming from v$active\_session\_history are flushed into the dba\_hist\_active\_sess\_history view.
2.  Those SQL work as of 11.2.0.1.
3.  You need to purchase the Diagnostic Pack in order to be allowed to query the "v$active\_session\_history" view.
4.  Those queries are useful to diagnose "huge" PGA or TEMP consumption, they are not so helpful to find out which sql\_id used exactly how much PGA or TEMP  (As it may used already pre-allocated PGA or TEMP space and did not need over allocation: See the columns definition in the beginning of the post)

<span style="text-decoration:underline;">**UPDATE:**</span> You can drill down in details per sql\_id execution into this [blog post](http://bdrouvot.wordpress.com/2013/04/19/drill-down-to-sql_id-execution-details-in-ash/ "Drill down to sql_id execution details in ASH")

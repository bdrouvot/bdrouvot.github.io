---
layout: post
title: Drill down to sql_id execution details in ASH
date: 2013-04-19 14:17:42.000000000 +02:00
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
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/04/19/drill-down-to-sql_id-execution-details-in-ash/"
---

Some times ago I explained how we can link a huge PGA or TEMP consumption to a sql\_id over a period of time into this [blog post](http://bdrouvot.wordpress.com/2013/03/19/link-huge-pga-temp/ "Link huge PGA or TEMP consumption to sql_id over a period of time").

Now I need to extract this information not only per sql\_id but also per execution. This is not so simple to extract from ash as the same session could execute many times the same sql\_id over a period of time.

Hopefully, since 11g the **sql\_exec\_id** column has been added to the v$active\_session\_history and dba\_hist\_active\_sess\_history views. You can find a very useful description of this column into this [blog post](http://blog.tanelpoder.com/2011/10/24/what-the-heck-is-the-sql-execution-id-sql_exec_id/) from Tanel Poder.

Basically, the **sql\_exec\_id** is a unique identifier of a sql\_id execution on the database. That way we are now able to know if an ash entry for this sql\_id is linked to a **new execution (means new sql\_exec\_id)** or is showing a **long running execution (means same sql\_exec\_id).**

We are also able to retrieve some useful metrics as avg, min, max execution time: You can see some good examples of sql\_exec\_id usage into [Kyle Hailey's blog post](http://dboptimizer.com/2011/05/04/sql-execution-times-from-ash/) or [Karl Arao's one](http://karlarao.tiddlyspot.com/#Elapsed-AvgMinMax).

<span style="text-decoration:underline;">**Back to my need:**</span>

Let's suppose that I found (Thanks to the sql provided into [this post](http://bdrouvot.wordpress.com/2013/03/19/link-huge-pga-temp/ "Link huge PGA or TEMP consumption to sql_id over a period of time")) that the sql\_id "btvk5dzpdmadh" is responsible of about 2Gb of pga over allocation during a period of time:

    SQL_ID            PGA_MB         %
    ------------- ---------- --------- ----------
    btvk5dzpdmadh       2394    100.00 **********

Now I can drill down to details to get the pga over allocation per execution that way:

\[code language="sql"\]  
alter session set nls\_date\_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls\_timestamp\_format='YYYY/MM/DD HH24:MI:SS';

select sql\_id,  
starting\_time,  
end\_time,  
(EXTRACT(HOUR FROM run\_time) \* 3600  
+ EXTRACT(MINUTE FROM run\_time) \* 60  
+ EXTRACT(SECOND FROM run\_time)) run\_time\_sec,  
READ\_IO\_BYTES,  
PGA\_ALLOCATED PGA\_ALLOCATED\_BYTES,  
TEMP\_ALLOCATED TEMP\_ALLOCATED\_BYTES  
from (  
select  
sql\_id,  
max(sample\_time - sql\_exec\_start) run\_time,  
max(sample\_time) end\_time,  
sql\_exec\_start starting\_time,  
sum(DELTA\_READ\_IO\_BYTES) READ\_IO\_BYTES,  
sum(DELTA\_PGA) PGA\_ALLOCATED,  
sum(DELTA\_TEMP) TEMP\_ALLOCATED  
from  
(  
select sql\_id,  
sample\_time,  
sql\_exec\_start,  
DELTA\_READ\_IO\_BYTES,  
sql\_exec\_id,  
greatest(PGA\_ALLOCATED - first\_value(PGA\_ALLOCATED) over (partition by sql\_id,sql\_exec\_id order by sample\_time rows 1 preceding),0) DELTA\_PGA,  
greatest(TEMP\_SPACE\_ALLOCATED - first\_value(TEMP\_SPACE\_ALLOCATED) over (partition by sql\_id,sql\_exec\_id order by sample\_time rows 1 preceding),0) DELTA\_TEMP  
from  
dba\_hist\_active\_sess\_history  
where  
sample\_time &gt;= to\_date ('2013/04/16 00:00:00','YYYY/MM/DD HH24:MI:SS')  
and sample\_time &lt; to\_date ('2013/04/16 03:10:00','YYYY/MM/DD HH24:MI:SS')  
and sql\_exec\_start is not null  
and IS\_SQLID\_CURRENT='Y'  
)  
group by sql\_id,SQL\_EXEC\_ID,sql\_exec\_start  
order by sql\_id  
)  
where sql\_id = 'btvk5dzpdmadh'  
order by sql\_id, run\_time\_sec desc;  
\[/code\]

It will produces this kind of output:

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC READ_IO_BYTES PGA_ALLOCATED_BYTES TEMP_ALLOCATED_BYTES
    ------------- ------------------- ------------------- ------------ ------------- ------------------- --------------------
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 03:09:56    11095.559    2.0417E+10           240123904           2642411520
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 03:09:56    11095.559    2.2207E+10           181993472           2233466880
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 03:09:56    11095.559    2.4342E+10           192610304           2453667840
    btvk5dzpdmadh 2013/04/16 00:13:43 2013/04/16 03:09:56    10573.559    1.3095E+10           212074496           1142947840
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 02:10:10     7509.398    1.9374E+10           200540160           2076180480
    btvk5dzpdmadh 2013/04/16 00:26:32 2013/04/16 02:05:19     5927.907    6603603968           226099200            251658240
    btvk5dzpdmadh 2013/04/16 01:37:19 2013/04/16 03:09:56     5557.559    7589232640           245497856            639631360
    btvk5dzpdmadh 2013/04/16 00:05:22 2013/04/16 01:36:57     5495.072    4285022208           237371392            125829120
    btvk5dzpdmadh 2013/04/16 02:12:54 2013/04/16 03:09:56     3422.559    7433412608           237371392            209715200
    btvk5dzpdmadh 2013/04/16 02:29:17 2013/04/16 03:09:56     2439.559    2776473600           288489472            146800640
    btvk5dzpdmadh 2013/04/16 02:05:43 2013/04/16 02:29:02     1399.436    1841930240           126484480             62914560
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 00:22:59     1078.508    1548386304            19333120
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 00:13:38      517.558     652337152            19202048
    btvk5dzpdmadh 2013/04/16 00:23:08 2013/04/16 00:26:29      201.867     439246848            29425664
    btvk5dzpdmadh 2013/04/16 02:10:57 2013/04/16 02:12:50      113.706      97910784            53673984
    btvk5dzpdmadh 2013/04/16 00:05:01 2013/04/16 00:05:17       16.703      22503424                   0

<span style="text-decoration:underline;">As you can see we retrieved:</span>

-   Start time of the execution
-   End time of the execution
-   The run time of the execution
-   The number of READ\_IO\_BYTES of the execution
-   The PGA and TEMP over allocation of the execution

<span style="text-decoration:underline;">Remarks:</span>

1.  The dba\_hist\_active\_sess\_history view is no so accurate as only a subset of the rows coming from v$active\_session\_history are flushed into the dba\_hist\_active\_sess\_history view.
2.  This SQL works as of 11.2.0.1.
3.  You need to purchase the Diagnostic Pack in order to be allowed to query the “v$active\_session\_history” view.
4.  This query is useful to diagnose “huge” PGA or TEMP consumption. It is not so helpful to find out which execution used **exactly** how much PGA or TEMP (As it may used already pre-allocated PGA or TEMP space and did not need over allocation: See the columns definition in the beginning of [this post](http://bdrouvot.wordpress.com/2013/03/19/link-huge-pga-temp/ "Link huge PGA or TEMP consumption to sql_id over a period of time"))

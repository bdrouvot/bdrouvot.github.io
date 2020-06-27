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
<p>Imagine you discovered that during a particular period of time a huge amount of PGA or TEMP space has been consumed by your database.</p>
<p>Then you want to know, if you could link this behavior to one or more sql_id.</p>
<p>Ok, I am a little bit late :-) but you should have noticed that since 11.2.0.1  two useful columns have been added to the v$active_session_history and dba_hist_active_sess_history views:</p>
<ul>
<li><strong>PGA_ALLOCATED</strong>:  Amount of PGA memory (in bytes) consumed by this session at the time this sample was taken</li>
<li><strong>TEMP_SPACE_ALLOCATED</strong>: Amount of TEMP memory (in bytes) consumed by this session at the time this sample was taken</li>
</ul>
<p><span style="text-decoration:underline;color:#0000ff;">Interesting, but is it helpful to answer:</span></p>
<ul>
<li>What are the top sql_id linked to pga consumption during a particular period of time ?</li>
<li>What are the top sql_id linked to temp space consumption during a particular period of time ?</li>
</ul>
<p>Coskan Gundogar gave a nice example related to a temp space issue, for a particular session <a href="http://coskan.wordpress.com/2011/01/24/analysing-temp-usage-on-11gr2-temp-space-is-not-released/" target="_blank">into this blog post</a>.</p>
<p>With this blog post, I just want to provide a way to generalize the computation at the instance level instead of the session one.</p>
<p>So, to find the top sql_id(s) responsible of  PGA or TEMP space consumption during a particular period of time, I use:</p>
<p><span style="text-decoration:underline;color:#0000ff;">For the PGA consumption:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat ash_sql_id_pga.sql<br />
col percent head '%' for 99990.99<br />
col star for A10 head ''</p>
<p>accept seconds prompt &quot;Last Seconds [60] : &quot; default 60;<br />
accept top prompt &quot;Top  Rows    [10] : &quot; default 10;</p>
<p>select SQL_ID,round(PGA_MB,1) PGA_MB,percent,rpad('*',percent*10/100,'*') star<br />
from<br />
(<br />
select SQL_ID,sum(DELTA_PGA_MB) PGA_MB ,(ratio_to_report(sum(DELTA_PGA_MB)) over ())*100 percent,rank() over(order by sum(DELTA_PGA_MB) desc) rank<br />
from<br />
(<br />
select SESSION_ID,SESSION_SERIAL#,sample_id,SQL_ID,SAMPLE_TIME,IS_SQLID_CURRENT,SQL_CHILD_NUMBER,PGA_ALLOCATED,<br />
greatest(PGA_ALLOCATED - first_value(PGA_ALLOCATED) over (partition by SESSION_ID,SESSION_SERIAL# order by sample_time rows 1 preceding),0)/power(1024,2) &quot;DELTA_PGA_MB&quot;<br />
from<br />
v$active_session_history<br />
where<br />
IS_SQLID_CURRENT='Y'<br />
and sample_time &gt; sysdate-&amp;seconds/86400<br />
order by 1,2,3,4<br />
)<br />
group by sql_id<br />
having sum(DELTA_PGA_MB) &gt; 0<br />
)<br />
where rank &lt; (&amp;top+1)<br />
order by rank<br />
/[/code]</p>
<p><span style="text-decoration:underline;">The output is like:</span></p>
<pre style="padding-left:30px;">SQL&gt; @ash_sql_id_pga.sql
Last Seconds [60] : 3600
Top  Rows    [10] : 10
old  13: and sample_time &gt; sysdate-&amp;seconds/86400
new  13: and sample_time &gt; sysdate-3600/86400
old  20: where rank &lt; (&amp;top+1)
new  20: where rank &lt; (10+1)

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

10 rows selected.</pre>
<p><span style="text-decoration:underline;color:#0000ff;">For the TEMP consumption:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat ash_sql_id_temp.sql<br />
col percent head '%' for 99990.99<br />
col star for A10 head ''</p>
<p>accept seconds prompt &quot;Last Seconds [60] : &quot; default 60;<br />
accept top prompt &quot;Top  Rows    [10] : &quot; default 10;</p>
<p>select SQL_ID,TEMP_MB,percent,rpad('*',percent*10/100,'*') star<br />
from<br />
(<br />
select SQL_ID,sum(DELTA_TEMP_MB) TEMP_MB ,(ratio_to_report(sum(DELTA_TEMP_MB)) over ())*100 percent,rank() over(order by sum(DELTA_TEMP_MB) desc) rank<br />
from<br />
(<br />
select SESSION_ID,SESSION_SERIAL#,sample_id,SQL_ID,SAMPLE_TIME,IS_SQLID_CURRENT,SQL_CHILD_NUMBER,temp_space_allocated,<br />
greatest(temp_space_allocated - first_value(temp_space_allocated) over (partition by SESSION_ID,SESSION_SERIAL# order by sample_time rows 1 preceding),0)/power(1024,2) &quot;DELTA_TEMP_MB&quot;<br />
from<br />
v$active_session_history<br />
where<br />
IS_SQLID_CURRENT='Y'<br />
and sample_time &gt; sysdate-&amp;seconds/86400<br />
order by 1,2,3,4<br />
)<br />
group by sql_id<br />
having sum(DELTA_TEMP_MB) &gt; 0<br />
)<br />
where rank &lt; (&amp;top+1)<br />
order by rank<br />
/<br />
[/code]</p>
<p><span style="text-decoration:underline;">The output is like:</span></p>
<pre style="padding-left:30px;">SQL&gt; @ash_sql_id_temp.sql
Last Seconds [60] : 3600
Top  Rows    [10] : 
old  13: and sample_time &gt; sysdate-&amp;seconds/86400
new  13: and sample_time &gt; sysdate-3600/86400
old  19: where rank &lt; (&amp;top+1)
new  19: where rank &lt; (10+1)

SQL_ID           TEMP_MB         %
------------- ---------- --------- ----------
by714720ajxwk 2 50.00 \*\*\*\*\* c2jdkwzndq685 1 25.00 \*\* 5h7w8ykwtb2xt 1 25.00 \*\*

Basically:

- The SQL computes,&nbsp;for each session, the PGA or TEMP space allocated between two active session history samples &nbsp;thanks to "**over (partition by SESSION\_ID,SESSION\_SERIAL# order by sample\_time rows 1 preceding)**".
- Those computed values are linked to the "active" sql\_id observed during the sampling.
- Then, it sums per sql\_id those computed values and display the top sql\_id(s).

**So, we are now able to know which sql\_id are responsible of huge PGA or TEMP consumption during a certain period of time.**

**Important remarks:**

1. You can also query the&nbsp;dba\_hist\_active\_sess\_history view but bear in mind that it is no so accurate as only a subset of the rows coming from&nbsp;v$active\_session\_history are flushed into the&nbsp;dba\_hist\_active\_sess\_history view.
2. Those SQL work as of 11.2.0.1.
3. You need to purchase the Diagnostic Pack in order to be allowed to query the "v$active\_session\_history" view.
4. Those queries are useful to diagnose "huge" PGA or TEMP consumption, they are not so helpful to find out which sql\_id used exactly how much PGA or TEMP &nbsp;(As it may used already pre-allocated PGA or TEMP space and did not need over allocation: See the columns definition in the&nbsp;beginning&nbsp;of the post)

**UPDATE:** You can drill down in details per sql\_id execution into this [blog post](http://bdrouvot.wordpress.com/2013/04/19/drill-down-to-sql_id-execution-details-in-ash/ "Drill down to sql\_id execution details in ASH")


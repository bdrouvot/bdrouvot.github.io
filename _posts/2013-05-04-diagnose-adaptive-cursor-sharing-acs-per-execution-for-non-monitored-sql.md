---
layout: post
title: Diagnose Adaptive Cursor Sharing (ACS) per execution for non monitored sql
date: 2013-05-04 05:59:45.000000000 +02:00
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
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:125;}s:2:"wp";a:1:{i:0;i:30;}}
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
permalink: "/2013/05/04/diagnose-adaptive-cursor-sharing-acs-per-execution-for-non-monitored-sql/"
---
<p>Into this blog post: <a title="Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2" href="http://bdrouvot.wordpress.com/2013/05/01/diagnose-adaptive-cursor-sharing-acs-per-execution-in-11-2/" target="_blank">Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2</a>  I provided a way to check if ACS came into play per execution of a given sql.</p>
<p>You should read the previous post to understand this one.</p>
<p>As you can see I retrieved for the bind variables the "<strong>peeked</strong>" and the "<strong>passed</strong>" values.</p>
<p>The "<strong>passed</strong>" values come from the <strong>v$sql_monitor.binds_xml</strong> column:  This information could be useful but is not mandatory to check if ACS came into play (<strong>as the check rely on the</strong> "<strong>peeked</strong>" values).</p>
<p>So <strong>we can get rid of the "passed" </strong>values (and then of the v$sql_monitor view) to check where ACS came into play per execution for non monitored sql.</p>
<p>For this purpose, let's modify the sql introduced into the previous post that way:</p>
<p>[code language="sql"]<br />
SQL&gt; !cat binds_peeked_acs.sql<br />
set linesi 200 pages 999 feed off verify off<br />
col bind_name format a20<br />
col end_time format a19<br />
col start_time format a19<br />
col peeked format a20</p>
<p>alter session set nls_date_format='YYYY/MM/DD HH24:MI:SS';<br />
alter session set nls_timestamp_format='YYYY/MM/DD HH24:MI:SS';</p>
<p>select<br />
pee.sql_id,<br />
ash.starting_time,<br />
ash.end_time,<br />
(EXTRACT(HOUR FROM ash.run_time) * 3600<br />
                    + EXTRACT(MINUTE FROM ash.run_time) * 60<br />
                    + EXTRACT(SECOND FROM ash.run_time)) run_time_sec,<br />
pee.plan_hash_value,<br />
pee.bind_name,<br />
pee.bind_pos,<br />
pee.bind_data peeked,<br />
--first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) previous_peeked,<br />
case when pee.bind_data =  first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) then 'NO' else 'YES' end &quot;ACS&quot;<br />
from<br />
(<br />
select<br />
p.sql_id,<br />
p.child_number,<br />
p.child_address,<br />
c.bind_name,<br />
c.bind_pos,<br />
p.plan_hash_value,<br />
case<br />
     when c.bind_type = 1 then utl_raw.cast_to_varchar2(c.bind_data)<br />
     when c.bind_type = 2 then to_char(utl_raw.cast_to_number(c.bind_data))<br />
     when c.bind_type = 96 then to_char(utl_raw.cast_to_varchar2(c.bind_data))<br />
     else 'Sorry: Not printable try with DBMS_XPLAN.DISPLAY_CURSOR'<br />
end bind_data<br />
from<br />
v$sql_plan p,<br />
xmltable<br />
(<br />
'/*/peeked_binds/bind' passing xmltype(p.other_xml)<br />
columns bind_name varchar2(30) path '/bind/@nam',<br />
bind_pos number path '/bind/@pos',<br />
bind_type number path '/bind/@dty',<br />
bind_data  raw(2000) path '/bind'<br />
) c<br />
where<br />
p.other_xml is not null<br />
) pee,<br />
(<br />
select<br />
sql_id,<br />
sql_child_number,<br />
sql_exec_id,<br />
max(sample_time - sql_exec_start) run_time,<br />
max(sample_time) end_time,<br />
sql_exec_start starting_time<br />
from<br />
v$active_session_history<br />
group by sql_id,sql_child_number,sql_exec_id,sql_exec_start<br />
) ash<br />
where<br />
pee.sql_id=ash.sql_id and<br />
pee.child_number=ash.sql_child_number and<br />
pee.sql_id like nvl('&amp;sql_id',pee.sql_id)<br />
order by 1,2,3,7 ;<br />
[/code]</p>
<p>Let's see the result with the same test as <a title="Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2" href="http://bdrouvot.wordpress.com/2013/05/01/diagnose-adaptive-cursor-sharing-acs-per-execution-in-11-2/" target="_blank">Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2:</a></p>
<pre style="padding-left:30px;">SQL&gt; @binds_peeked_acs.sql
Enter value for sql_id: bu9367qrhq28t

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               ACS
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- ---
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_OWNER 1 BDT NO bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_OBJECT\_ID 3 1 NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_OWNER 1 BDT NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_OBJECT\_ID 3 1 NO bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_OWNER 1 ME YES bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_OBJECT\_ID 3 1 NO bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_OWNER 1 BDT YES bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_OBJECT\_ID 3 1 NO bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_OWNER 1 ME YES bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_OBJECT\_ID 3 1 NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_OWNER 1 ME NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_DATE 2 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_OBJECT\_ID 3 1 NO

So, same result as in the previous post except that the bind variable "passed" values have been lost.

**Conclusion:**

We are able to check for which execution ACS came into play for non monitored sql (as we get rid of the bind variable "passed" values and as a consequence we don't query the v$sql\_monitor view anymore).

**Remark:**

You need to purchase the Diagnostic Pack in order to be allowed to query the “v$active\_session\_history” view


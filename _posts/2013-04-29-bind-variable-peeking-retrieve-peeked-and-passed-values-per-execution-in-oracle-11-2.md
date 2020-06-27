---
layout: post
title: 'Bind variable peeking: Retrieve peeked and passed values per execution in
  oracle 11.2'
date: 2013-04-29 10:37:45.000000000 +02:00
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
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _oembed_8c7a261b5f90d70c4404901f4dbdb553: "{{unknown}}"
  _oembed_35f7ee0d952dbf5a2df974bcfa527f51: "{{unknown}}"
  _oembed_130fd421dedf595f337b5c80f3acc748: "{{unknown}}"
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/04/29/bind-variable-peeking-retrieve-peeked-and-passed-values-per-execution-in-oracle-11-2/"
---
<p>To understand this blog post you have to know what bind variable peeking is. You can found a very good explanation into this <a href="http://kerryosborne.oracle-guy.com/2009/03/bind-variable-peeking-drives-me-nuts/" target="_blank">Kerry Osborne's blog post</a>.</p>
<p>So when dealing with performance issues linked to bind variable peeking you have to know:</p>
<ul>
<li>The peeked values (The ones that generate the execution plan)</li>
<li>The passed values (The ones that have been passed to the sql statement)</li>
</ul>
<p>Kerry Osborne helped us to retrieve the <strong>peeked</strong> values from <strong>v$sql_plan</strong> view into this <a href="http://kerryosborne.oracle-guy.com/2009/07/creating-test-scripts-with-bind-variables/" target="_blank">blog post</a>, but what about the passed values ?  For those ones, Tanel Poder helped us to retrieve the <strong>passed</strong> values from <strong>v$sql_monitor</strong> into this <a href="http://blog.tanelpoder.com/2010/10/18/read-currently-running-sql-statements-bind-variable-values/" target="_blank">blog post</a> (This is reliable compare to <strong>v$sql_bind_capture</strong>)</p>
<p>Great ! So we know how to extract the peeked and the passed values. Another interesting point is that v$sql_monitor contains also the <strong>sql_exec_id</strong> field (see this <a title="Drill down to sql_id execution details in ASH" href="http://bdrouvot.wordpress.com/2013/04/19/drill-down-to-sql_id-execution-details-in-ash/" target="_blank">blog post</a> for more details about this field).</p>
<p>Here we are:  <strong>It looks like that as of 11.2 we are able to retrieve the passed and peeked values per execution</strong> (If the statement is "monitored" which  means CPU + I/O wait time &gt;= 5 seconds per default (can be changed thanks to the _sqlmon_threshold hidden parameter).</p>
<p>But as your are dealing with performance issues related to bind variable peeking it is likely that the sql is monitored ;-)</p>
<p>So let's write the sql to do so, and let's test it.</p>
<p><span style="text-decoration:underline;">The sql script is the following:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat binds_peeked_passed.sql<br />
set linesi 200 pages 999 feed off verify off<br />
col bind_name format a20<br />
col end_time format a19<br />
col start_time format a19<br />
col peeked format a20<br />
col passed format a20</p>
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
run_t.bind_data passed<br />
from<br />
(<br />
select<br />
p.sql_id,<br />
p.sql_child_address,<br />
p.sql_exec_id,<br />
c.bind_name,<br />
c.bind_pos,<br />
c.bind_data<br />
from<br />
v$sql_monitor p,<br />
xmltable<br />
(<br />
'/binds/bind' passing xmltype(p.binds_xml)<br />
columns bind_name varchar2(30) path '/bind/@name',<br />
bind_pos number path '/bind/@pos',<br />
bind_data varchar2(30) path '/bind'<br />
) c<br />
where<br />
p.binds_xml is not null<br />
) run_t<br />
,<br />
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
sql_exec_id,<br />
max(sample_time - sql_exec_start) run_time,<br />
max(sample_time) end_time,<br />
sql_exec_start starting_time<br />
from<br />
v$active_session_history<br />
group by sql_id,sql_exec_id,sql_exec_start<br />
) ash<br />
where<br />
pee.sql_id=run_t.sql_id and<br />
pee.sql_id=ash.sql_id and<br />
run_t.sql_exec_id=ash.sql_exec_id and<br />
pee.child_address=run_t.sql_child_address and<br />
pee.bind_name=run_t.bind_name and<br />
pee.bind_pos=run_t.bind_pos and<br />
pee.sql_id like nvl('&amp;sql_id',pee.sql_id)<br />
order by 1,2,3,7 ;<br />
[/code]</p>
<p><span style="text-decoration:underline;">Now let's test it:</span></p>
<pre style="padding-left:30px;">SQL&gt; var my_owner varchar2(50)
SQL&gt; var my_date varchar2(30)
SQL&gt; var my_object_id number
SQL&gt; exec :my_owner :='BDT'

PL/SQL procedure successfully completed.

SQL&gt; exec :my_date := '01-jan-2001'

PL/SQL procedure successfully completed.

SQL&gt; exec :my_object_id :=1

PL/SQL procedure successfully completed.

SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
   6974365              2          18233

SQL&gt; @binds_peeked_passed.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- --------------------
bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OWNER                     1 BDT                  BDT
bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001
bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OBJECT_ID                 3 1                    1</pre>
<p>As this is the first execution then peeked values = passed values. Note that I used the "MONITOR" hint to force the sql to be monitored and then get an entry into v$sql_monitor.</p>
<p><span style="text-decoration:underline;">Let's put new passed values:</span></p>
<pre style="padding-left:30px;">SQL&gt; exec :my_date := '01-jan-2002'

PL/SQL procedure successfully completed.

SQL&gt; exec :my_object_id :=100

PL/SQL procedure successfully completed.

SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
   6923776            101          18233

SQL&gt; @binds_peeked_passed.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- --------------------
bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07 5.678 1047781245 :MY\_OWNER 1 BDT BDT bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07 5.678 1047781245 :MY\_DATE 2 01-jan-2001 01-jan-2001 bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07 5.678 1047781245 :MY\_OBJECT\_ID 3 1 1 bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25 4.139 1047781245 :MY\_OWNER 1 BDT BDT bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25 4.139 1047781245 :MY\_DATE 2 01-jan-2001 01-jan-2002 bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25 4.139 1047781245 :MY\_OBJECT\_ID 3 1 100

So as you can see, peeked values are the same and passed are not: bind variable peeking in action ;-)

**Conclusion:**

We are able to retrieve peeked and passed values per execution.

**Remarks:**

1. You need Diagnostic and tuning licenses pack to query v$active\_session\_history and v$sql\_monitor.
2. The query is not able to retrieve the DATE values (if any) from the v$sql\_plan (check the code): This is because I don't want to create a new function into the database. If you want to extract the DATE datatype then you could create the display\_raw function (see Kerry Osborne's [blog post](http://kerryosborne.oracle-guy.com/2009/07/creating-test-scripts-with-bind-variables/) for this) and modify the sql.
3. If you know how to extract DATE values from RAW type without creating new function please tell me so that i can update the code ;-)

---
layout: post
title: Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2
date: 2013-05-01 17:00:54.000000000 +02:00
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
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _oembed_020de7c7a1416e9e437103f418e20edc: "{{unknown}}"
  _oembed_fece1af1eff9c54f6da635929cef63ef: "{{unknown}}"
  _oembed_3f5e4706e63100b9f99b7eae932c51b0: "{{unknown}}"
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/05/01/diagnose-adaptive-cursor-sharing-acs-per-execution-in-11-2/"
---
<p>As you know oracle introduced a new feature "Adaptive cursor sharing (ACS)"  in 11g. You can find a very good explanation of what it is into this <a href="http://optimizermagic.blogspot.be/2007/12/why-are-there-more-cursors-in-11g-for.html" target="_blank">Maria Colgan's blog post</a>.</p>
<p>So, as Maria said: "<strong>A bind aware cursor may use different plans for different bind values, depending on how selective the predicates containing the bind variable are.</strong>"</p>
<p>That's fine, but <strong>I would like to see per execution</strong> of a given sql_id, if the Adaptive Cursor Sharing feature came into play.</p>
<p><span style="text-decoration:underline;"><strong>Let's define "When ACS comes into play" means: ACS comes into play for a particular execution:</strong> </span></p>
<ol>
<li>if the <strong>peeked</strong> values (The ones that generate the execution plan) <strong>changed</strong> compare to the previous execution.</li>
<li>if this execution is not the first one that has been executed after the initial hard parse.</li>
</ol>
<p>For this, I adapted the query that I use to retrieve "peeked" and "passed" bind values per execution into this <a title="Bind variable peeking: Retrieve peeked and passed values per execution in oracle 11.2" href="http://bdrouvot.wordpress.com/2013/04/29/bind-variable-peeking-retrieve-peeked-and-passed-values-per-execution-in-oracle-11-2/" target="_blank">blog post</a> that way:</p>
<p>[code language="sql"]<br />
SQL&gt;!cat binds_peeked_passed_acs.sql<br />
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
--first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) previous_peeked,<br />
run_t.bind_data passed,<br />
case when pee.bind_data =  first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) then 'NO' else 'YES' end &quot;ACS&quot;<br />
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
<p>So, I simply added this line:</p>
<p>[code language="sql"]<br />
case when pee.bind_data =  first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) then 'NO' else 'YES' end &quot;ACS&quot;<br />
[/code]</p>
<p><span style="text-decoration:underline;">This new line:</span></p>
<ol>
<li>Will result in 'YES'  <strong>if the value of a "peeked" bind variable changed compare to the previous execution</strong>.</li>
<li>Will result in "NO" if this is the first execution after the hard parse or the value of a peeked variable did not change compare to the previous execution.</li>
</ol>
<p><span style="text-decoration:underline;">Let's test it:</span></p>
<p>There is a data skew on the owner column which has one index on it. The data distribution is the following:</p>
<pre>SQL&gt; select owner,count(*) from bdt2 group by owner;

OWNER                            COUNT(*)
------------------------------ ----------
BDT                              13848830
ME                                 100098</pre>
<p><span style="text-decoration:underline;">Let's query the table that way:</span></p>
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
  13848830              2          18233

SQL&gt; set pagesi 0
SQL&gt; select * from table(dbms_xplan.display_cursor);
SQL_ID  bu9367qrhq28t, child number 0
-------------------------------------
select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2
where owner=:my_owner and created &gt; :my_date and object_id &gt;
:my_object_id

Plan hash value: 1047781245

---------------------------------------------------------------------------
| Id  | Operation          | Name | Rows  | Bytes | Cost (%CPU)| Time     |
---------------------------------------------------------------------------
|   0 | SELECT STATEMENT   |      |       |       | 12726 (100)|          |
|   1 |  SORT AGGREGATE    |      |     1 |    17 |            |          |
|*  2 |   TABLE ACCESS FULL| BDT2 |    13M|   224M| 12726   (6)| 00:01:59 |
---------------------------------------------------------------------------</pre>
<p>So a Full Table Scan occured and the "peeked" and "passed" bind variables are:</p>
<pre style="padding-left:30px;">SQL&gt; @binds_peeked_passed_acs.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  BDT                  NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO</pre>
<p>and no ACS into play.</p>
<p><span style="text-decoration:underline;">Now, let's change the bind values of the owner column and check the "peeked" and "passed" values:</span></p>
<pre style="padding-left:30px;">SQL&gt; exec :my_owner :='ME';

PL/SQL procedure successfully completed.

SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
    100098              2          18233

SQL&gt; @binds_peeked_passed_acs.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  BDT                  NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OWNER                     1 BDT                  ME                   NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO</pre>
<p>So, same "peeked" values while the "passed" ones are not the same (and still not ACS triggered) as we can check that way (See <a href="http://optimizermagic.blogspot.be/2007/12/why-are-there-more-cursors-in-11g-for.html" target="_blank">Maria Colgan's blog post</a>):</p>
<pre style="padding-left:30px;">SQL&gt; l
  1* select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t'
SQL&gt; /

CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
------------ ---------- ----------- -------------------- --------------------
           0          2      360295 Y                    N</pre>
<p><span style="text-decoration:underline;">Now let's run the query a second time with the 'ME' value for the owner column field:</span></p>
<pre style="padding-left:30px;">SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
    100098              2          18233

And the execution plan has changed:
SQL&gt; select * from table(dbms_xplan.display_cursor);
SQL_ID  bu9367qrhq28t, child number 1
-------------------------------------
select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2
where owner=:my_owner and created &gt; :my_date and object_id &gt;
:my_object_id

Plan hash value: 2372635759

------------------------------------------------------------------------------------------
| Id  | Operation                    | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
------------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT             |           |       |       |  1511 (100)|          |
|   1 |  SORT AGGREGATE              |           |     1 |    17 |            |          |
|*  2 |   TABLE ACCESS BY INDEX ROWID| BDT2      |   100K|  1661K|  1511   (1)| 00:00:15 |
|*  3 |    INDEX RANGE SCAN          | BDT_OWNER |   100K|       |   213   (1)| 00:00:02 |
------------------------------------------------------------------------------------------</pre>
<p>As you can see the execution plan changed. Well, let's see the result of my sql:</p>
<pre style="padding-left:30px;">SQL&gt;@binds_peeked_passed_acs.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  BDT                  NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OWNER                     1 BDT                  ME                   NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO
bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OWNER                     1 ME                   ME                   YES
bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OBJECT_ID                 3 1                    1                    NO</pre>
<p><strong>Great! It detected that ACS came into play for this execution</strong>. Fine but what's new compare to checking:</p>
<pre style="padding-left:30px;">SQL&gt;select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t';

CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
------------ ---------- ----------- -------------------- --------------------
           0          2      360295 Y                    N
           1          1        1576 Y                    Y</pre>
<p><strong>What's new is that you can check if ACS came into play per execution</strong>. Let's run the sql 3 times with 2 changes of the bind value and check the result:</p>
<pre style="padding-left:30px;">SQL&gt;exec :my_owner :='BDT'

PL/SQL procedure successfully completed.
SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
  13848830              2          18233

SQL&gt; exec :my_owner :='ME';

PL/SQL procedure successfully completed.

SQL&gt; select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created &gt; :my_date and object_id &gt; :my_object_id;

  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
    100098              2          18233

SQL&gt; /
  COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
---------- -------------- --------------
    100098              2          18233</pre>
<p>You don't have more informations from v$sql (<strong>you can see that ACS came into play but you don't know for which execution</strong>):</p>
<pre style="padding-left:30px;">SQL&gt; l
  1* select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t'
SQL&gt; /

CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
------------ ---------- ----------- -------------------- --------------------
           0          2      360295 Y                    N
           1          3        3152 Y                    Y
           2          1      180104 Y                    Y</pre>
<p>while <strong>you can have the details per execution that way</strong>:</p>
<pre style="padding-left:30px;">QL&gt; @binds_peeked_passed_acs.sql
Enter value for sql_id: 

SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_OWNER 1 BDT BDT NO bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05 6.788 1047781245 :MY\_OBJECT\_ID 3 1 1 NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_OWNER 1 BDT ME NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13 8.005 1047781245 :MY\_OBJECT\_ID 3 1 1 NO bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_OWNER 1 ME ME YES bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15 1.448 2372635759 :MY\_OBJECT\_ID 3 1 1 NO bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_OWNER 1 BDT BDT YES bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55 6.859 1047781245 :MY\_OBJECT\_ID 3 1 1 NO bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_OWNER 1 ME ME YES bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06 1.879 2372635759 :MY\_OBJECT\_ID 3 1 1 NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_OWNER 1 ME ME NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_DATE 2 01-jan-2001 01-jan-2001 NO bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13 1.879 2372635759 :MY\_OBJECT\_ID 3 1 1 NO

**Conclusion:**

You know for which executions ACS came into play and furthermore which "peeked" bind variable value changed compare to the previous execution (ACS column='YES').

**Remarks:**

1. You need Diagnostic and tuning licenses pack to query v$active\_session\_history and v$sql\_monitor.
2. The query rely on the fact that the sql is monitored (which &nbsp;means CPU + I/O wait time \>= 5 seconds per default that can be changed thanks to the \_sqlmon\_threshold hidden parameter)
3. If you are ready to get rid of the "passed" values, then you can check this post for non monitored sql: [Diagnose Adaptive Cursor Sharing (ACS) per execution for non monitored sql](http://bdrouvot.wordpress.com/2013/05/04/diagnose-adaptive-cursor-sharing-acs-per-execution-for-non-monitored-sql/ "Diagnose Adaptive Cursor Sharing (ACS) per execution for non monitored sql")

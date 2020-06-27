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

To understand this blog post you have to know what bind variable peeking is. You can found a very good explanation into this [Kerry Osborne's blog post](http://kerryosborne.oracle-guy.com/2009/03/bind-variable-peeking-drives-me-nuts/).

So when dealing with performance issues linked to bind variable peeking you have to know:

-   The peeked values (The ones that generate the execution plan)
-   The passed values (The ones that have been passed to the sql statement)

Kerry Osborne helped us to retrieve the **peeked** values from **v$sql\_plan** view into this [blog post](http://kerryosborne.oracle-guy.com/2009/07/creating-test-scripts-with-bind-variables/), but what about the passed values ?  For those ones, Tanel Poder helped us to retrieve the **passed** values from **v$sql\_monitor** into this [blog post](http://blog.tanelpoder.com/2010/10/18/read-currently-running-sql-statements-bind-variable-values/) (This is reliable compare to **v$sql\_bind\_capture**)

Great ! So we know how to extract the peeked and the passed values. Another interesting point is that v$sql\_monitor contains also the **sql\_exec\_id** field (see this [blog post](http://bdrouvot.wordpress.com/2013/04/19/drill-down-to-sql_id-execution-details-in-ash/ "Drill down to sql_id execution details in ASH") for more details about this field).

Here we are:  **It looks like that as of 11.2 we are able to retrieve the passed and peeked values per execution** (If the statement is "monitored" which  means CPU + I/O wait time &gt;= 5 seconds per default (can be changed thanks to the \_sqlmon\_threshold hidden parameter).

But as your are dealing with performance issues related to bind variable peeking it is likely that the sql is monitored ;-)

So let's write the sql to do so, and let's test it.

<span style="text-decoration:underline;">The sql script is the following:</span>

```
SQL> !cat binds_peeked_passed.sql  
set linesi 200 pages 999 feed off verify off  
col bind_name format a20  
col end_time format a19  
col start_time format a19  
col peeked format a20  
col passed format a20

alter session set nls_date_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls_timestamp_format='YYYY/MM/DD HH24:MI:SS';

select  
pee.sql_id,  
ash.starting_time,  
ash.end_time,  
(EXTRACT(HOUR FROM ash.run_time) * 3600  
+ EXTRACT(MINUTE FROM ash.run_time) * 60  
+ EXTRACT(SECOND FROM ash.run_time)) run_time_sec,  
pee.plan_hash_value,  
pee.bind_name,  
pee.bind_pos,  
pee.bind_data peeked,  
run_t.bind_data passed  
from  
(  
select  
p.sql_id,  
p.sql_child_address,  
p.sql_exec_id,  
c.bind_name,  
c.bind_pos,  
c.bind_data  
from  
v$sql_monitor p,  
xmltable  
(  
'/binds/bind' passing xmltype(p.binds_xml)  
columns bind_name varchar2(30) path '/bind/@name',  
bind_pos number path '/bind/@pos',  
bind_data varchar2(30) path '/bind'  
) c  
where  
p.binds_xml is not null  
) run_t  
,  
(  
select  
p.sql_id,  
p.child_number,  
p.child_address,  
c.bind_name,  
c.bind_pos,  
p.plan_hash_value,  
case  
when c.bind_type = 1 then utl_raw.cast_to_varchar2(c.bind_data)  
when c.bind_type = 2 then to_char(utl_raw.cast_to_number(c.bind_data))  
when c.bind_type = 96 then to_char(utl_raw.cast_to_varchar2(c.bind_data))  
else 'Sorry: Not printable try with DBMS_XPLAN.DISPLAY_CURSOR'  
end bind_data  
from  
v$sql_plan p,  
xmltable  
(  
'/*/peeked_binds/bind' passing xmltype(p.other_xml)  
columns bind_name varchar2(30) path '/bind/@nam',  
bind_pos number path '/bind/@pos',  
bind_type number path '/bind/@dty',  
bind_data raw(2000) path '/bind'  
) c  
where  
p.other_xml is not null  
) pee,  
(  
select  
sql_id,  
sql_exec_id,  
max(sample_time - sql_exec_start) run_time,  
max(sample_time) end_time,  
sql_exec_start starting_time  
from  
v$active_session_history  
group by sql_id,sql_exec_id,sql_exec_start  
) ash  
where  
pee.sql_id=run_t.sql_id and  
pee.sql_id=ash.sql_id and  
run_t.sql_exec_id=ash.sql_exec_id and  
pee.child_address=run_t.sql_child_address and  
pee.bind_name=run_t.bind_name and  
pee.bind_pos=run_t.bind_pos and  
pee.sql_id like nvl('&sql_id',pee.sql_id)  
order by 1,2,3,7 ;  
```

<span style="text-decoration:underline;">Now let's test it:</span>

    SQL> var my_owner varchar2(50)
    SQL> var my_date varchar2(30)
    SQL> var my_object_id number
    SQL> exec :my_owner :='BDT'

    PL/SQL procedure successfully completed.

    SQL> exec :my_date := '01-jan-2001'

    PL/SQL procedure successfully completed.

    SQL> exec :my_object_id :=1

    PL/SQL procedure successfully completed.

    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
       6974365              2          18233

    SQL> @binds_peeked_passed.sql
    Enter value for sql_id: 

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED
    ------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- --------------------
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OWNER                     1 BDT                  BDT
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OBJECT_ID                 3 1                    1

As this is the first execution then peeked values = passed values. Note that I used the "MONITOR" hint to force the sql to be monitored and then get an entry into v$sql\_monitor.

<span style="text-decoration:underline;">Let's put new passed values:</span>

    SQL> exec :my_date := '01-jan-2002'

    PL/SQL procedure successfully completed.

    SQL> exec :my_object_id :=100

    PL/SQL procedure successfully completed.

    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
       6923776            101          18233

    SQL> @binds_peeked_passed.sql
    Enter value for sql_id: 

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED
    ------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- --------------------
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OWNER                     1 BDT                  BDT
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001
    bu9367qrhq28t 2013/04/29 11:01:02 2013/04/29 11:01:07        5.678      1047781245 :MY_OBJECT_ID                 3 1                    1
    bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25        4.139      1047781245 :MY_OWNER                     1 BDT                  BDT
    bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25        4.139      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2002
    bu9367qrhq28t 2013/04/29 11:07:21 2013/04/29 11:07:25        4.139      1047781245 :MY_OBJECT_ID                 3 1                    100

So as you can see, peeked values are the same and passed are not: bind variable peeking in action ;-)

<span style="text-decoration:underline;">**Conclusion:**</span>

We are able to retrieve peeked and passed values per execution.

<span style="text-decoration:underline;">**Remarks:**</span>

1.  You need Diagnostic and tuning licenses pack to query v$active\_session\_history and v$sql\_monitor.
2.  The query is not able to retrieve the DATE values (if any) from the v$sql\_plan (check the code): This is because I don't want to create a new function into the database. If you want to extract the DATE datatype then you could create the display\_raw function (see Kerry Osborne's [blog post](http://kerryosborne.oracle-guy.com/2009/07/creating-test-scripts-with-bind-variables/) for this) and modify the sql.
3.  If you know how to extract DATE values from RAW type without creating new function please tell me so that i can update the code ;-)

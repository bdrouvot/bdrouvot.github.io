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

\[code language="sql"\]  
SQL&gt; !cat binds\_peeked\_passed.sql  
set linesi 200 pages 999 feed off verify off  
col bind\_name format a20  
col end\_time format a19  
col start\_time format a19  
col peeked format a20  
col passed format a20

alter session set nls\_date\_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls\_timestamp\_format='YYYY/MM/DD HH24:MI:SS';

select  
pee.sql\_id,  
ash.starting\_time,  
ash.end\_time,  
(EXTRACT(HOUR FROM ash.run\_time) \* 3600  
+ EXTRACT(MINUTE FROM ash.run\_time) \* 60  
+ EXTRACT(SECOND FROM ash.run\_time)) run\_time\_sec,  
pee.plan\_hash\_value,  
pee.bind\_name,  
pee.bind\_pos,  
pee.bind\_data peeked,  
run\_t.bind\_data passed  
from  
(  
select  
p.sql\_id,  
p.sql\_child\_address,  
p.sql\_exec\_id,  
c.bind\_name,  
c.bind\_pos,  
c.bind\_data  
from  
v$sql\_monitor p,  
xmltable  
(  
'/binds/bind' passing xmltype(p.binds\_xml)  
columns bind\_name varchar2(30) path '/bind/@name',  
bind\_pos number path '/bind/@pos',  
bind\_data varchar2(30) path '/bind'  
) c  
where  
p.binds\_xml is not null  
) run\_t  
,  
(  
select  
p.sql\_id,  
p.child\_number,  
p.child\_address,  
c.bind\_name,  
c.bind\_pos,  
p.plan\_hash\_value,  
case  
when c.bind\_type = 1 then utl\_raw.cast\_to\_varchar2(c.bind\_data)  
when c.bind\_type = 2 then to\_char(utl\_raw.cast\_to\_number(c.bind\_data))  
when c.bind\_type = 96 then to\_char(utl\_raw.cast\_to\_varchar2(c.bind\_data))  
else 'Sorry: Not printable try with DBMS\_XPLAN.DISPLAY\_CURSOR'  
end bind\_data  
from  
v$sql\_plan p,  
xmltable  
(  
'/\*/peeked\_binds/bind' passing xmltype(p.other\_xml)  
columns bind\_name varchar2(30) path '/bind/@nam',  
bind\_pos number path '/bind/@pos',  
bind\_type number path '/bind/@dty',  
bind\_data raw(2000) path '/bind'  
) c  
where  
p.other\_xml is not null  
) pee,  
(  
select  
sql\_id,  
sql\_exec\_id,  
max(sample\_time - sql\_exec\_start) run\_time,  
max(sample\_time) end\_time,  
sql\_exec\_start starting\_time  
from  
v$active\_session\_history  
group by sql\_id,sql\_exec\_id,sql\_exec\_start  
) ash  
where  
pee.sql\_id=run\_t.sql\_id and  
pee.sql\_id=ash.sql\_id and  
run\_t.sql\_exec\_id=ash.sql\_exec\_id and  
pee.child\_address=run\_t.sql\_child\_address and  
pee.bind\_name=run\_t.bind\_name and  
pee.bind\_pos=run\_t.bind\_pos and  
pee.sql\_id like nvl('&sql\_id',pee.sql\_id)  
order by 1,2,3,7 ;  
\[/code\]

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

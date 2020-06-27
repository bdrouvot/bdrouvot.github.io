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

As you know oracle introduced a new feature "Adaptive cursor sharing (ACS)"  in 11g. You can find a very good explanation of what it is into this [Maria Colgan's blog post](http://optimizermagic.blogspot.be/2007/12/why-are-there-more-cursors-in-11g-for.html).

So, as Maria said: "**A bind aware cursor may use different plans for different bind values, depending on how selective the predicates containing the bind variable are.**"

That's fine, but **I would like to see per execution** of a given sql\_id, if the Adaptive Cursor Sharing feature came into play.

<span style="text-decoration:underline;">**Let's define "When ACS comes into play" means: ACS comes into play for a particular execution:** </span>

1.  if the **peeked** values (The ones that generate the execution plan) **changed** compare to the previous execution.
2.  if this execution is not the first one that has been executed after the initial hard parse.

For this, I adapted the query that I use to retrieve "peeked" and "passed" bind values per execution into this [blog post](http://bdrouvot.wordpress.com/2013/04/29/bind-variable-peeking-retrieve-peeked-and-passed-values-per-execution-in-oracle-11-2/ "Bind variable peeking: Retrieve peeked and passed values per execution in oracle 11.2") that way:

```
SQL&gt;!cat binds\_peeked\_passed\_acs.sql  
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
--first\_value(pee.bind\_data) over (partition by pee.sql\_id,pee.bind\_name,pee.bind\_pos order by end\_time rows 1 preceding) previous\_peeked,  
run\_t.bind\_data passed,  
case when pee.bind\_data = first\_value(pee.bind\_data) over (partition by pee.sql\_id,pee.bind\_name,pee.bind\_pos order by end\_time rows 1 preceding) then 'NO' else 'YES' end "ACS"  
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
```

So, I simply added this line:

```
case when pee.bind\_data = first\_value(pee.bind\_data) over (partition by pee.sql\_id,pee.bind\_name,pee.bind\_pos order by end\_time rows 1 preceding) then 'NO' else 'YES' end "ACS"  
```

<span style="text-decoration:underline;">This new line:</span>

1.  Will result in 'YES'  **if the value of a "peeked" bind variable changed compare to the previous execution**.
2.  Will result in "NO" if this is the first execution after the hard parse or the value of a peeked variable did not change compare to the previous execution.

<span style="text-decoration:underline;">Let's test it:</span>

There is a data skew on the owner column which has one index on it. The data distribution is the following:

    SQL> select owner,count(*) from bdt2 group by owner;

    OWNER                            COUNT(*)
    ------------------------------ ----------
    BDT                              13848830
    ME                                 100098

<span style="text-decoration:underline;">Let's query the table that way:</span>

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
      13848830              2          18233

    SQL> set pagesi 0
    SQL> select * from table(dbms_xplan.display_cursor);
    SQL_ID  bu9367qrhq28t, child number 0
    -------------------------------------
    select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2
    where owner=:my_owner and created > :my_date and object_id >
    :my_object_id

    Plan hash value: 1047781245

    ---------------------------------------------------------------------------
    | Id  | Operation          | Name | Rows  | Bytes | Cost (%CPU)| Time     |
    ---------------------------------------------------------------------------
    |   0 | SELECT STATEMENT   |      |       |       | 12726 (100)|          |
    |   1 |  SORT AGGREGATE    |      |     1 |    17 |            |          |
    |*  2 |   TABLE ACCESS FULL| BDT2 |    13M|   224M| 12726   (6)| 00:01:59 |
    ---------------------------------------------------------------------------

So a Full Table Scan occured and the "peeked" and "passed" bind variables are:

    SQL> @binds_peeked_passed_acs.sql
    Enter value for sql_id: 

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
    ------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  BDT                  NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO

and no ACS into play.

<span style="text-decoration:underline;">Now, let's change the bind values of the owner column and check the "peeked" and "passed" values:</span>

    SQL> exec :my_owner :='ME';

    PL/SQL procedure successfully completed.

    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
        100098              2          18233

    SQL> @binds_peeked_passed_acs.sql
    Enter value for sql_id: 

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               PASSED               ACS
    ------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- -------------------- ---
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  BDT                  NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OWNER                     1 BDT                  ME                   NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO

So, same "peeked" values while the "passed" ones are not the same (and still not ACS triggered) as we can check that way (See [Maria Colgan's blog post](http://optimizermagic.blogspot.be/2007/12/why-are-there-more-cursors-in-11g-for.html)):

    SQL> l
      1* select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t'
    SQL> /

    CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
    ------------ ---------- ----------- -------------------- --------------------
               0          2      360295 Y                    N

<span style="text-decoration:underline;">Now let's run the query a second time with the 'ME' value for the owner column field:</span>

    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
        100098              2          18233

    And the execution plan has changed:
    SQL> select * from table(dbms_xplan.display_cursor);
    SQL_ID  bu9367qrhq28t, child number 1
    -------------------------------------
    select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2
    where owner=:my_owner and created > :my_date and object_id >
    :my_object_id

    Plan hash value: 2372635759

    ------------------------------------------------------------------------------------------
    | Id  | Operation                    | Name      | Rows  | Bytes | Cost (%CPU)| Time     |
    ------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT             |           |       |       |  1511 (100)|          |
    |   1 |  SORT AGGREGATE              |           |     1 |    17 |            |          |
    |*  2 |   TABLE ACCESS BY INDEX ROWID| BDT2      |   100K|  1661K|  1511   (1)| 00:00:15 |
    |*  3 |    INDEX RANGE SCAN          | BDT_OWNER |   100K|       |   213   (1)| 00:00:02 |
    ------------------------------------------------------------------------------------------

As you can see the execution plan changed. Well, let's see the result of my sql:

    SQL>@binds_peeked_passed_acs.sql
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
    bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OBJECT_ID                 3 1                    1                    NO

**Great! It detected that ACS came into play for this execution**. Fine but what's new compare to checking:

    SQL>select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t';

    CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
    ------------ ---------- ----------- -------------------- --------------------
               0          2      360295 Y                    N
               1          1        1576 Y                    Y

**What's new is that you can check if ACS came into play per execution**. Let's run the sql 3 times with 2 changes of the bind value and check the result:

    SQL>exec :my_owner :='BDT'

    PL/SQL procedure successfully completed.
    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
      13848830              2          18233

    SQL> exec :my_owner :='ME';

    PL/SQL procedure successfully completed.

    SQL> select /*+ MONITOR */ count(*),min(object_id),max(object_id) from bdt2 where owner=:my_owner and created > :my_date and object_id > :my_object_id;

      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
        100098              2          18233

    SQL> /
      COUNT(*) MIN(OBJECT_ID) MAX(OBJECT_ID)
    ---------- -------------- --------------
        100098              2          18233

You don't have more informations from v$sql (**you can see that ACS came into play but you don't know for which execution**):

    SQL> l
      1* select child_number, executions, buffer_gets,is_bind_sensitive, is_bind_aware  from v$sql where sql_id='bu9367qrhq28t'
    SQL> /

    CHILD_NUMBER EXECUTIONS BUFFER_GETS IS_BIND_SENSITIVE    IS_BIND_AWARE
    ------------ ---------- ----------- -------------------- --------------------
               0          2      360295 Y                    N
               1          3        3152 Y                    Y
               2          1      180104 Y                    Y

while **you can have the details per execution that way**:

    QL> @binds_peeked_passed_acs.sql
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
    bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OBJECT_ID                 3 1                    1                    NO
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_OWNER                     1 BDT                  BDT                  YES
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_OBJECT_ID                 3 1                    1                    NO
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_OWNER                     1 ME                   ME                   YES
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_OBJECT_ID                 3 1                    1                    NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_OWNER                     1 ME                   ME                   NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_DATE                      2 01-jan-2001          01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_OBJECT_ID                 3 1                    1                    NO

<span style="text-decoration:underline;">**Conclusion:**</span>

You know for which executions ACS came into play and furthermore which "peeked" bind variable value changed compare to the previous execution (ACS column='YES').

<span style="text-decoration:underline;">**Remarks:**</span>

1.  You need Diagnostic and tuning licenses pack to query v$active\_session\_history and v$sql\_monitor.
2.  The query rely on the fact that the sql is monitored (which  means CPU + I/O wait time &gt;= 5 seconds per default that can be changed thanks to the \_sqlmon\_threshold hidden parameter)
3.  If you are ready to get rid of the "passed" values, then you can check this post for non monitored sql: [Diagnose Adaptive Cursor Sharing (ACS) per execution for non monitored sql](http://bdrouvot.wordpress.com/2013/05/04/diagnose-adaptive-cursor-sharing-acs-per-execution-for-non-monitored-sql/ "Diagnose Adaptive Cursor Sharing (ACS) per execution for non monitored sql")

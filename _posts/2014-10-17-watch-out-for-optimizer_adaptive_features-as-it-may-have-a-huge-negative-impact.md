---
layout: post
title: Watch out for optimizer_adaptive_features as It may have a huge negative impact
date: 2014-10-17 09:23:47.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
- Rac
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/JVJC82gMTCj
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/JtzGQxtVTV
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5928792283678810112&type=U&a=Mapz
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/10/17/watch-out-for-optimizer_adaptive_features-as-it-may-have-a-huge-negative-impact/"
---

Let me describe how I discovered that the [optimizer\_adaptive\_features](http://kerryosborne.oracle-guy.com/papers/12c_Adaptive_Optimization.pdf) may have a **huge negative** impact (**specially** on RAC databases).

I upgraded a 11gR2 database to 12.1.0.2 and then I converted this database to a PDB. To do so I launched "<span id="kmPgTpl:sd_r1:0:dv_rDoc:ot71" class="kmContent">*@$ORACLE\_HOME/rdbms/admin/noncdb\_to\_pdb.sql*" (See MOS Doc ID <span id="kmPgTpl:sd_r1:0:dv_rDoc:0:ol22" class="xq">1564657.1</span> for more details).</span>

This sql script took **about 75 minutes** to complete on my 2 nodes RAC database. This is quite long and then I decided to try to reduce this duration.

To diagnose: I dropped the PDB, plugged it back, re-launched *<span id="kmPgTpl:sd_r1:0:dv_rDoc:ot71" class="kmContent">noncdb\_to\_pdb.sql</span>* and enabled [a 10046 trace on the session](http://www.gokhanatil.com/2011/05/tracing-oracle-sessions.html).

<span style="text-decoration:underline;">**The top SQL (sort by elapsed time)** for this session is the following:</span>

    SQL ID: frjd8zfy2jfdq Plan Hash: 510421217

    SELECT executions, end_of_fetch_count,              elapsed_time/px_servers
      elapsed_time,        cpu_time/px_servers     cpu_time,
      buffer_gets/executions  buffer_gets
    FROM
     (SELECT sum(executions)   as executions,                            sum(case
      when px_servers_executions > 0                              then
      px_servers_executions                                  else executions end)
      as px_servers,                sum(end_of_fetch_count) as end_of_fetch_count,
                    sum(elapsed_time) as elapsed_time,
      sum(cpu_time)     as cpu_time,                     sum(buffer_gets)  as
      buffer_gets            FROM   gv$sql
      WHERE executions > 0                                 AND sql_id = :1
                                  AND parsing_schema_name = :2)


    call     count       cpu    elapsed       disk      query    current        rows
    ------- ------  -------- ---------- ---------- ---------- ----------  ----------
    Parse    70502      4.02       4.25          0          0          0           0
    Execute  70502    264.90     759.33          0          0          0           0
    Fetch    70502    215.77    1848.46          0          0          0       70502
    ------- ------  -------- ---------- ---------- ---------- ----------  ----------
    total   211506    484.70    2612.05          0          0          0       70502

    Misses in library cache during parse: 17
    Misses in library cache during execute: 17
    Optimizer mode: CHOOSE
    Parsing user id: SYS   (recursive depth: 2)
    Number of plan statistics captured: 70502

    Rows (1st) Rows (avg) Rows (max)  Row Source Operation
    ---------- ---------- ----------  ---------------------------------------------------
             1          1          1  VIEW  (cr=0 pr=0 pw=0 time=76 us)
             1          1          1   SORT AGGREGATE (cr=0 pr=0 pw=0 time=76 us)
             0          0          1    PX COORDINATOR  (cr=0 pr=0 pw=0 time=76 us)
             0          0          0     PX SEND QC (RANDOM) :TQ10000 (cr=0 pr=0 pw=0 time=0 us)
             0          0          0      VIEW  GV$SQL (cr=0 pr=0 pw=0 time=0 us)
             0          0          0       FIXED TABLE FIXED INDEX X$KGLCURSOR_CHILD (ind:2) (cr=0 pr=0 pw=0 time=0 us)

    Elapsed times include waiting on following events:
      Event waited on                             Times   Max. Wait  Total Waited
      ----------------------------------------   Waited  ----------  ------------
      PX Deq: Join ACK                           281741        0.03        368.85
      PX Deq: reap credit                       1649736        0.00         25.22
      IPC send completion sync                   141000        0.02        135.07
      PX Deq: Parse Reply                        141004        0.04        170.64
      PX Deq: Execute Reply                      141004        0.08       1366.42
      reliable message                            70502        0.00        125.51
      PX Deq: Signal ACK EXT                     140996        0.03         13.99
      PX Deq: Slave Session Stats                140996        0.02         25.54
      enq: PS - contention                        70605        0.11        145.80
      KJC: Wait for msg sends to complete          2584        0.00          0.04
      latch free                                     16        0.00          0.00
      PX qref latch                                  14        0.00          0.00
      latch: shared pool                              4        0.00          0.00
      latch: active service list                      1        0.00          0.00
      row cache lock                                127        0.00          0.08
      library cache lock                             36        0.00          0.04
      library cache pin                              36        0.00          0.04
      gc cr grant 2-way                               1        0.00          0.00
      db file sequential read                         1        0.00          0.00
      oracle thread bootstrap                         1        0.03          0.03

As you can see this SQL elapsed time is about 2600 seconds in total (for about 70 000 executions), most of the wait time comes from "**PX %**" events and this SQL queries the **GV$SQL view**.

<span style="text-decoration:underline;">**But wait:** </span>

Why the hell this SQL (which **looks like an SQL that collects metrics** for a particular sql\_id) is somehow part of the session that launched the <span id="kmPgTpl:sd_r1:0:dv_rDoc:ot71" class="kmContent">*noncdb\_to\_pdb.sql* script? Does it make sense?</span>

I really don't think so, this sql (**sql\_id "frjd8zfy2jfdq"**) should have been triggered **by something else** (parsing or whatever) than *<span id="kmPgTpl:sd_r1:0:dv_rDoc:ot71" class="kmContent">noncdb\_to\_pdb.sql</span>*.

<span style="text-decoration:underline;">Let's prove it with a simple test case (I am alone on the database):</span>

```
alter system flush shared_pool;

select count(*) from v$sql where sql_id='frjd8zfy2jfdq';

connect / as sysdba

begin  
execute immediate 'select object_name from dba_objects';  
end;  
/

select count(*) from v$sql where sql_id='frjd8zfy2jfdq';  
```

<span style="text-decoration:underline;">With the following result:</span>

    SQL> @test_case

    System altered.


      COUNT(*)
    ----------
             0

    Connected.

    PL/SQL procedure successfully completed.


      COUNT(*)
    ----------
             2

Did you see that the **sql\_id "frjd8zfy2jfdq"** has been produced by this simple test case? (If you trace the session you would see this query into the trace file).

<span style="text-decoration:underline;">I **reproduced** this behavior on:</span>

-   12.1.0.2 database with CDB/PDB.
-   12.1.0.2 database (Non CDB).

and **was not able to reproduce** it on a 11gR2 database.

So, it looks like that this SQL is introduced by a 12.1.0.2 (12cR1) new feature. As it is somehow linked to "SQL metrics collection", I tried to disable some 12cR1 features (linked to this area) until I found the "culprit".

It did not produce any change **until I set the *optimizer\_adaptive\_features* parameter to false (true is the default).**

<span style="text-decoration:underline;">Here is the result:</span>

```
SQL> !cat test_case.sql  
alter system flush shared_pool;

select count(*) from v$sql where sql_id='frjd8zfy2jfdq';

connect / as sysdba  
alter session set optimizer_adaptive_features=false;

begin  
execute immediate 'select object_name from dba_objects';  
end;  
/

select count(*) from v$sql where sql_id='frjd8zfy2jfdq';

SQL> @test_case

System altered.

COUNT(*)  
----------  
0

Connected.

Session altered.

PL/SQL procedure successfully completed.

COUNT(*)  
----------  
0  
```

**BINGO!!!** The sql\_id "frjd8zfy2jfdq" has not been executed!

**Well, now what is the impact on *noncdb\_to\_pdb.sql* on my 2 nodes RAC database?**

I edited the script and added:

    alter session set optimizer_adaptive_features=false;

just above this line:

    exec dbms_pdb.noncdb_to_pdb(1);

**Guess what?**

1.  *noncdb\_to\_pdb.sql* took about **20 minutes to execute** (compare to **about 75 minutes** without setting *optimizer\_adaptive\_features* to false).
2.  The sql\_id "frjd8zfy2jfdq" **is not part of the trace file anymore**.

<span style="text-decoration:underline;">**Remarks:**</span>

-   The impact of this SQL is much more "visible" with a **RAC database**, as it queries a **GLOBAL V$ view** (GV$SQL) and then needs parallel query to run (If more than one instance is up)*.* With only one instance up, the 10046 trace file produced:

<!-- -->

    SQL ID: frjd8zfy2jfdq Plan Hash: 510421217

    SELECT executions, end_of_fetch_count,              elapsed_time/px_servers
      elapsed_time,        cpu_time/px_servers     cpu_time,
      buffer_gets/executions  buffer_gets
    FROM
     (SELECT sum(executions)   as executions,                            sum(case
      when px_servers_executions > 0                              then
      px_servers_executions                                  else executions end)
      as px_servers,                sum(end_of_fetch_count) as end_of_fetch_count,
                    sum(elapsed_time) as elapsed_time,
      sum(cpu_time)     as cpu_time,                     sum(buffer_gets)  as
      buffer_gets            FROM   gv$sql
      WHERE executions > 0                                 AND sql_id = :1
                                  AND parsing_schema_name = :2)


    call     count       cpu    elapsed       disk      query    current        rows
    ------- ------  -------- ---------- ---------- ---------- ----------  ----------
    Parse    69204      2.58       2.78          0          0          0           0
    Execute  69204     23.93      25.45          0          0          0           0
    Fetch    69204      2.68       2.64          0          0          0       69204
    ------- ------  -------- ---------- ---------- ---------- ----------  ----------
    total   207612     29.20      30.88          0          0          0       69204

    Misses in library cache during parse: 18
    Misses in library cache during execute: 18
    Optimizer mode: CHOOSE
    Parsing user id: SYS   (recursive depth: 2)
    Number of plan statistics captured: 62

    Rows (1st) Rows (avg) Rows (max)  Row Source Operation
    ---------- ---------- ----------  ---------------------------------------------------
             1          1          1  VIEW  (cr=0 pr=0 pw=0 time=120 us)
             1          1          1   SORT AGGREGATE (cr=0 pr=0 pw=0 time=113 us)
             0          0          0    PX COORDINATOR  (cr=0 pr=0 pw=0 time=100 us)
             0          0          0     PX SEND QC (RANDOM) :TQ10000 (cr=0 pr=0 pw=0 time=50 us)
             0          0          0      VIEW  GV$SQL (cr=0 pr=0 pw=0 time=43 us)
             0          0          0       FIXED TABLE FIXED INDEX X$KGLCURSOR_CHILD (ind:2) (cr=0 pr=0 pw=0 time=39 us)


    Elapsed times include waiting on following events:
      Event waited on                             Times   Max. Wait  Total Waited
      ----------------------------------------   Waited  ----------  ------------
      row cache lock                                  1        0.00          0.00

As you can see: elapsed time of about 30 seconds (for about 70 000 executions) and no "PX%" wait events.

-   I checked which parameters (hidden or not) changed when setting *optimizer\_adaptive\_features* to false. Then I tried one by one those parameters with my test case and discovered that setting "***\_optimizer\_dsdir\_usage\_control***" to 0 is enough to get rid of the sql\_id "frjd8zfy2jfdq".

<!-- -->

-   During the run of *noncdb\_to\_pdb.sql,* this is the **large number of executions** (about 70 000) of the sql\_id "frjd8zfy2jfdq" that produced this long "overall" duration and **highlighted** the fact that this query has to be somehow removed.

<!-- -->

-   You could meet this SQL on 12cR1 databases (CDB/PDB or non CDB).

<span style="text-decoration:underline;">**Conclusion:**</span>

-   The *optimizer\_adaptive\_features* (set to true) **may produce a huge negative impact** on the performance (specially with **RAC database)**. I provided an example of such an impact when running the *noncdb\_to\_pdb.sql* script.

<!-- -->

-   Should you meet the sql\_id "frjd8zfy2jfdq" in your database and would like to get rid of it (because you observe a negative impact): Then simply set *optimizer\_adaptive\_features* to false (or "*\_optimizer\_dsdir\_usage\_control*" to 0) at the session or system level.

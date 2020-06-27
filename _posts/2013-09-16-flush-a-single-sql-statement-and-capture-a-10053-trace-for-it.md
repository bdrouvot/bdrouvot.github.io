---
layout: post
title: Flush a single SQL statement and capture a 10053 trace for it
date: 2013-09-16 22:16:41.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ToolKit
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/09/16/flush-a-single-sql-statement-and-capture-a-10053-trace-for-it/"
---

Just a quick one, to share a simple sql script in case you need to trace the optimizer computations for a single SQL statement.

As you know, Oracle Database 11g, introduced a new diagnostic events infrastructure, which greatly simplifies the task of generating a 10053 trace for a specific SQL statement.

We can capture a 10053 trace for a specific sql\_id that way:

**alter system set events 'trace\[RDBMS.SQL\_Optimizer.\*\]\[sql:&lt;YOUR\_SQL\_ID&gt;\]';**

<span style="text-decoration:underline;">But the trace will be **triggered after a hard parse**. So, if the sql is already in the shared pool we have 2 choices:</span>

1.  **Wait until** the statement is hard parsed again (because it has been aged out of the shared pool, because of a new child cursor creation..).
2.  **Flush the sql** from the shared pool (See [Kerry Osborne's post](http://kerryosborne.oracle-guy.com/2008/09/flush-a-single-sql-statement/)) so that the next execution will generate a hard parse.

So, If you can't be patient, then you can use this script to **flush the sql and enable the 10053 trace**:

```
SQL> !cat enable_10053_sql_id.sql  
set serveroutput on  
set pagesize 9999  
set linesize 155  
var name varchar2(50)

prompt WARNING SQL_ID WILL BE PURGED FROM THE SHARED POOL  
accept sql_id prompt 'Enter value for sql_id: '

BEGIN

select address||','||hash_value into :name  
from v$sqlarea  
where sql_id like '&&sql_id';

dbms_shared_pool.purge(:name,'C',1);

END;  
/

alter system set events 'trace[RDBMS.SQL_Optimizer.*][sql:&&sql_id]';

undef sql_id  
undef name  
```

Then just wait for the next execution and you'll get the trace file.

<span style="text-decoration:underline;">To disable the trace:</span>

```
SQL> !cat disable_10053_sql_id.sql  
prompt DISABLING 10053 trace

accept sql_id prompt 'Enter value for sql_id: '

alter system set events 'trace[RDBMS.SQL_Optimizer.*][sql:&&sql_id] off';  
```

<span style="text-decoration:underline;">**Remark:**</span>

-   Starting in 11gR2 you can use DBMS\_SQLDIAG.DUMP\_TRACE as it doesn’t require you to re-execute the statement to get the trace (It will automatically trigger a hard parse, see MOS 225598.1).

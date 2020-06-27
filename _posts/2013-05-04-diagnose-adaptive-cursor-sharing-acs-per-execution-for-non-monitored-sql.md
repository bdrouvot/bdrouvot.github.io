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

Into this blog post: [Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2](http://bdrouvot.wordpress.com/2013/05/01/diagnose-adaptive-cursor-sharing-acs-per-execution-in-11-2/ "Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2")  I provided a way to check if ACS came into play per execution of a given sql.

You should read the previous post to understand this one.

As you can see I retrieved for the bind variables the "**peeked**" and the "**passed**" values.

The "**passed**" values come from the **v$sql\_monitor.binds\_xml** column:  This information could be useful but is not mandatory to check if ACS came into play (**as the check rely on the** "**peeked**" values).

So **we can get rid of the "passed"** values (and then of the v$sql\_monitor view) to check where ACS came into play per execution for non monitored sql.

For this purpose, let's modify the sql introduced into the previous post that way:

```
SQL> !cat binds_peeked_acs.sql  
set linesi 200 pages 999 feed off verify off  
col bind_name format a20  
col end_time format a19  
col start_time format a19  
col peeked format a20

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
--first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) previous_peeked,  
case when pee.bind_data = first_value(pee.bind_data) over (partition by pee.sql_id,pee.bind_name,pee.bind_pos order by end_time rows 1 preceding) then 'NO' else 'YES' end "ACS"  
from  
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
sql_child_number,  
sql_exec_id,  
max(sample_time - sql_exec_start) run_time,  
max(sample_time) end_time,  
sql_exec_start starting_time  
from  
v$active_session_history  
group by sql_id,sql_child_number,sql_exec_id,sql_exec_start  
) ash  
where  
pee.sql_id=ash.sql_id and  
pee.child_number=ash.sql_child_number and  
pee.sql_id like nvl('&sql_id',pee.sql_id)  
order by 1,2,3,7 ;  
```

Let's see the result with the same test as [Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2:](http://bdrouvot.wordpress.com/2013/05/01/diagnose-adaptive-cursor-sharing-acs-per-execution-in-11-2/ "Diagnose Adaptive Cursor Sharing (ACS) per execution in 11.2")

    SQL> @binds_peeked_acs.sql
    Enter value for sql_id: bu9367qrhq28t

    SQL_ID        STARTING_TIME       END_TIME            RUN_TIME_SEC PLAN_HASH_VALUE BIND_NAME              BIND_POS PEEKED               ACS
    ------------- ------------------- ------------------- ------------ --------------- -------------------- ---------- -------------------- ---
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OWNER                     1 BDT                  NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:17:59 2013/05/03 14:18:05        6.788      1047781245 :MY_OBJECT_ID                 3 1                    NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OWNER                     1 BDT                  NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:21:05 2013/05/03 14:21:13        8.005      1047781245 :MY_OBJECT_ID                 3 1                    NO
    bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OWNER                     1 ME                   YES
    bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:27:14 2013/05/03 14:27:15        1.448      2372635759 :MY_OBJECT_ID                 3 1                    NO
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_OWNER                     1 BDT                  YES
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:32:49 2013/05/03 14:32:55        6.859      1047781245 :MY_OBJECT_ID                 3 1                    NO
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_OWNER                     1 ME                   YES
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:33:05 2013/05/03 14:33:06        1.879      2372635759 :MY_OBJECT_ID                 3 1                    NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_OWNER                     1 ME                   NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_DATE                      2 01-jan-2001          NO
    bu9367qrhq28t 2013/05/03 14:33:12 2013/05/03 14:33:13        1.879      2372635759 :MY_OBJECT_ID                 3 1                    NO

So, same result as in the previous post except that the bind variable "passed" values have been lost.

<span style="text-decoration:underline;">**Conclusion:**</span>

We are able to check for which execution ACS came into play for non monitored sql (as we get rid of the bind variable "passed" values and as a consequence we don't query the v$sql\_monitor view anymore).

<span style="text-decoration:underline;">**Remark:**</span>

You need to purchase the Diagnostic Pack in order to be allowed to query the “v$active\_session\_history” view

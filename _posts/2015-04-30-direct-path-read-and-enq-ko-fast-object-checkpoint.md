---
layout: post
title: 'Direct Path Read and "enq: KO - fast object checkpoint"'
date: 2015-04-30 16:48:05.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/6jwhcfc27pd
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/vZ2Heab86h
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5999569734201335808&type=U&a=pJEN
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/04/30/direct-path-read-and-enq-ko-fast-object-checkpoint/"
---
<h2>Introduction</h2>
<p>When using direct path read, oracle reads the blocks from the disk. The buffer cache is not being used.</p>
<p>Then, when oracle begins to read a segment using direct path read, it flush its dirty blocks to disk (A dirty block is a block that does not have the same image in memory and on disk). During that time, the session that triggered the direct path read is waiting for the "enq: KO - fast object checkpoint" wait event.</p>
<p>In this post, I just want to see with my own eyes that the dirty blocks being flushed to disk belong only to the segment being read.</p>
<h2>Tests</h2>
<p>For the test, I used two tables on a 11.2.0.4 database. One table is made of 2 partitions and one is not partitioned (then 3 segments):</p>
<p>[code language="sql"]<br />
BDT:BDT1&gt; select SEGMENT_NAME,PARTITION_NAME from dba_segments where segment_name in ('BDT1','BDT2');</p>
<p>SEGMENT_NAME PARTITION_NAME<br />
------------ --------------<br />
BDT2<br />
BDT1         P2<br />
BDT1         P1<br />
[/code]</p>
<p>The rows distribution across segments, blocks and file relative numbers is the following:</p>
<p>[code language="sql"]<br />
BDT:BDT1&gt; l<br />
  1  select u.object_name,u.subobject_name,dbms_rowid.rowid_block_number(BDT1.rowid) BLOCK_NUMBER, dbms_rowid.rowid_relative_fno(BDT1.rowid) FNO<br />
  2	 from BDT1, user_objects u<br />
  3	 where dbms_rowid.rowid_object(BDT1.rowid) = u.object_id<br />
  4	 union all<br />
  5	 select u.object_name,u.subobject_name,dbms_rowid.rowid_block_number(BDT2.rowid) BLOCK_NUMBER, dbms_rowid.rowid_relative_fno(BDT2.rowid) FNO<br />
  6	 from BDT2, user_objects u<br />
  7	 where dbms_rowid.rowid_object(BDT2.rowid) = u.object_id<br />
  8	 order by 1,2<br />
  9*<br />
BDT:BDT1&gt; /</p>
<p>OBJECT_NAME  SUBOBJECT_NAME  BLOCK_NUMBER FNO<br />
------------ --------------- ------------ -----------<br />
BDT1	     P1              2509842	  5<br />
BDT1	     P1              2509842      5<br />
BDT1	     P1              2509842      5<br />
BDT1	     P1              2509842      5<br />
BDT1	     P2              2510866      5<br />
BDT1	     P2              2510866      5<br />
BDT1	     P2              2510866      5<br />
BDT1	     P2              2510866      5<br />
BDT1	     P2              2510866      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5<br />
BDT2                         1359091      5</p>
<p>18 rows selected.[/code]</p>
<p>so that all the rows of the segment:</p>
<ul>
<li>BDT1:P1 are in datafile 5, block 2509842</li>
<li>BDT1:P2 are in datafile 5, block 2510866</li>
<li>BDT2 are in datafile 5, block 1359091</li>
</ul>
<p><span style="text-decoration:underline;">Remark:</span></p>
<p>BDT1:P&lt;N&gt; stands for table BDT1 partition &lt;N&gt;</p>
<p>Now, let's update both tables (<strong>without</strong> committing):</p>
<p>[code language="sql"]<br />
BDT:BDT1&gt; update BDT1 set ID2=100;</p>
<p>9 rows updated.</p>
<p>BDT:BDT1&gt; update BDT2 set ID2=100;</p>
<p>9 rows updated.<br />
[/code]</p>
<p>As I do not commit (nor rollback), then those <strong>3 blocks in memory contain an active transaction</strong>.</p>
<p>In another session, let's force serial direct path read and read partition <strong>P1</strong> from <strong>BDT1 only</strong>:</p>
<p>[code language="sql"]<br />
BDT:BDT1&gt; alter session set &quot;_serial_direct_read&quot;=always;</p>
<p>Session altered.</p>
<p>BDT:BDT1&gt; select count(*) from BDT1 partition (P1);</p>
<p>COUNT(*)<br />
-----------<br />
4<br />
[/code]</p>
<p>Now, check its wait events and related statistics with snapper. A cool thing with snapper is that I can get wait events and statistics from one tool.</p>
<p>[code language="sql"]<br />
SYS:BDT1&gt; @snapper all,end 5 1 20<br />
Sampling SID 20 with interval 5 seconds, taking 1 snapshots...</p>
<p>-- Session Snapper v4.22 - by Tanel Poder ( http://blog.tanelpoder.com/snapper ) - Enjoy the Most Advanced Oracle Troubleshooting Script on the Planet! :)</p>
<p>------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------<br />
 SID @INST, USERNAME  , TYPE, STATISTIC                                                 ,         DELTA, HDELTA/SEC,    %TIME, GRAPH       , NUM_WAITS,  WAITS/SEC,   AVERAGES<br />
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------<br />
    20  @1, BDT       , STAT, physical reads direct                                     ,             1,        .02,         ,             ,          ,           ,         .5 per execution<br />
    20  @1, BDT       , WAIT, enq: KO - fast object checkpoint                          ,          1117,    18.12us,      .0%, [          ],         1,        .02,     1.12ms average wait</p>
<p>--  End of Stats snap 1, end=2015-04-29 08:25:20, seconds=61.7</p>
<p>--  End of ASH snap 1, end=, seconds=, samples_taken=0, AAS=(No ASH sampling in begin/end snapshot mode)</p>
<p>PL/SQL procedure successfully completed.<br />
[/code]</p>
<p>For brevity and clarity, I keep only the wait events and statistics of interest.</p>
<p>As we can see, the direct path read has been used and the "fast object checkpoint" occurred (as the session waited for it).</p>
<p>Now let's check which block(s) has/have been flushed to disk: to do so, I'll dump those 3 blocks from disk and check <strong>if</strong> they contain an active transaction (as the ones in memory).</p>
<p>[code language="sql"]<br />
BDT:BDT1&gt; !cat dump_blocks.sql<br />
-- BDT2<br />
alter system dump datafile 5 block 1359091;<br />
-- BDT1:P2<br />
alter system dump datafile 5 block 2510866;<br />
-- BDT1:P1<br />
alter system dump datafile 5 block 2509842;<br />
select value from v$diag_info where name like 'Default%';</p>
<p>BDT:BDT1&gt; @dump_blocks.sql</p>
<p>System altered.</p>
<p>System altered.</p>
<p>System altered.</p>
<p>VALUE<br />
------------------------------<br />
/u01/app/oracle/diag/rdbms/bdt/BDT1/trace/BDT1_ora_91815.trc</p>
<p>BDT:BDT1&gt; !view /u01/app/oracle/diag/rdbms/bdt/BDT1/trace/BDT1_ora_91815.trc<br />
[/code]</p>
<p>The Interested Transaction List (ITL) for the first block that has been dumped from disk (BDT2 Table) looks like:</p>
<pre> Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.0016f9c7
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x03   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000</pre>
<p>We can see that the block on disk does not contain an active transaction in it (There is no rows locked as Lck=0). We can conclude that the block that belongs to the <strong>BDT2 table</strong> <strong>has not been flushed</strong> to disk (The block is still dirty).</p>
<p>The Interested Transaction List (ITL) for the second block that has been dumped from disk (BDT1:P2) looks like:</p>
<pre style="padding-left:30px;"> Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.0016f99f
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x03   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000</pre>
<p>We can see that the block on disk does not contain an active transaction in it (There is no rows locked as Lck=0). We can conclude that the block that belongs to the partition <strong>P2 of the BDT1 table</strong> <strong>has not been flushed</strong> to disk (The block is still dirty).</p>
<p>The Interested Transaction List (ITL) for the third block that has been dumped from disk (BDT1:P1) looks like:</p>
<pre> Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.0016f998
0x02   0x000a.005.00001025  0x00c00211.038e.23  ----    4  fsc 0x0000.00000000
0x03   0x0000.000.00000000  0x00000000.0000.00  ----
0 fsc 0x0000.00000000

We can see that the block on disk **does contain an active** transaction in it (4 rows are locked as Lck=4). This is the block that has been read in direct path read&nbsp;during the select on BDT1:P1.

We can conclude that the block that belongs&nbsp;to **BDT1:P1 has been flushed to disk** (The block is not dirty anymore).

## Conclusion

When oracle begins to read a segment using direct path read:

- It does not flush to disk the dirty blocks from other segments.
- Only the dirty blocks of the segment being read are flushed to disk.

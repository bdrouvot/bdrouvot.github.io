---
layout: post
title: ASM Bytes Read and Smart Scan
date: 2015-04-20 15:30:05.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/QEemDzQxj2h
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/d2EWxlkR4k
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5995926225472798720&type=U&a=yoWE
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
permalink: "/2015/04/20/asm-bytes-read-and-smart-scan/"
---
<h2>Introduction</h2>
<p>I like to query the <em>v$asm_disk_iostat</em> cumulative view (at the ASM instance level) as this is a centralized location where you can find metrics for all the databases the ASM instance is servicing. One of its metric is:</p>
<p><strong>BYTES_READ</strong>: Total number of bytes read from the disk (see the <a href="http://docs.oracle.com/cd/E18283_01/server.112/e17110/dynviews_1025.htm" target="_blank">oracle documentation</a>).</p>
<p>I would like to see which value is recorded into this metric in case of smart scan. To do so, I'll launch some tests and check the output of the <a title="asm_metrics" href="https://bdrouvot.wordpress.com/asm_metrics_script/" target="_blank">asm_metrics</a> utility: It basically takes a snapshot each second (default interval) from the<em> gv$asm_disk_iostat</em> cumulative view and <strong>computes the delta</strong> with the previous snapshot.</p>
<h2>Environment</h2>
<p>ASM and database versions are 11.2.0.4. A segment of about <strong>19.5 gb</strong> has been created without any indexes, so that a full table scan is triggered during the tests.</p>
<h2>Tests</h2>
<p>During the tests:</p>
<ul>
<li>This simple query is launched:</li>
</ul>
<pre>select id from bdt where id=1;</pre>
<ul>
<li>Then, the asm metrics will be recorded that way:</li>
</ul>
<pre>./asm_metrics.pl -show=dbinst,fg,inst -dbinst=BDT2 -inst=+ASM2 -display=avg</pre>
<ul>
<li>We'll look at the output field "Kby Read/s" which is based on the BYTES_READ column coming from the <em>v$asm_disk_iostat</em> cumulative view.</li>
</ul>
<ul>
<li>The sql elapsed time and the percentage of IO saved by the offload (if any) will be checked with a query that looks like <a href="http://kerryosborne.oracle-guy.com/scripts/fsx.sql" target="_blank">fsx.sql</a>. By “percentage of IO saved” I mean the ratio of data received from the storage cells to the actual amount of data that would have had to be received on non-Exadata storage.</li>
</ul>
<h3>Test 1: Launch the query without offload</h3>
<pre style="padding-left:30px;">BDT:BDT2&gt; alter session set cell_offload_processing=false;

Session altered.

BDT:BDT2&gt; select /* NO_SMART_SCAN_NO_STORAGEIDX */ id from bdt where id=1;</pre>
<p>The elapsed time and the percentage of IO saved are:</p>
<pre>SQL_ID	       CHILD   PLAN_HASH  EXECS  AVG_ETIME AVG_PX OFFLOAD ELIGIBLE_MB  INTERCO_MB OFF_RETU_MB IO_SAVED_% SQL_TEXT
------------- ------ ----------- ------ ---------- ------ ------- ----------- ----------- ----------- ---------- ----------------------------------------------------------------------
7y7xa34jjab2q	   0   627556429      1      23.11	0 No		    0 19413.42969	    0	     .00 select /* NO_SMART_SCAN_NO_STORAGEIDX */ id from bdt where id=1</pre>
<p>The elapsed time of the sql is <strong>23.11</strong> seconds and obviously no IO have been saved by offload. As you can see about <strong>19.5 gb</strong> has been exchanged between the Oracle Database and the storage system (INTERCO_MB is based on IO_INTERCONNECT_BYTES column from v$sql).</p>
<p>The asm_metrics ouput is the following:</p>
<pre style="padding-left:30px;">06:39:47                                                                            Kby       Avg       AvgBy/               Kby       Avg        AvgBy/
06:39:47   INST     DBINST        DG            FG           DSK          Reads/s   Read/s    ms/Read   Read      Writes/s   Write/s   ms/Write   Write
06:39:47   ------   -----------   -----------   ----------   ----------   -------   -------   -------   ------    ------     -------   --------   ------
06:39:47   +ASM2                                                          869       885423    3.7       1042916   2          24        0.9        15477
06:39:47   +ASM2    BDT2                                                  869       885423    3.7       1042916   2          24        0.9        15477
06:39:47   +ASM2    BDT2                        ENKCEL01                  300       305865    3.7       1044812   1          8         1.2        15061
06:39:47   +ASM2    BDT2                        ENKCEL02                  330       335901    3.7       1041451   1          8         0.8        15061
06:39:47   +ASM2    BDT2                        ENKCEL03                  239       243657    3.6       1042564   0          8         0.6        16384
</pre>
<p>So the average Kby Read per second is 885423.</p>
<p>Then we can conclude that the BYTES_READ column records that about 885423 * 23.11 = <strong>19.5 gb</strong> has been read from disk.</p>
<p>Does it make sense? Yes.</p>
<h3>Test 2: Offload and no storage indexes</h3>
<pre>BDT:BDT2&gt; alter session set cell_offload_processing=true;

Session altered.

BDT:BDT2&gt; alter session set "_kcfis_storageidx_disabled"=true;

Session altered.

BDT:BDT2&gt; select /* WITH_SMART_SCAN_NO_STORAGEIDX */ id from bdt where id=1;</pre>
<p>The elapsed time and the percentage of IO saved are:</p>
<pre style="padding-left:30px;">SQL_ID	       CHILD   PLAN_HASH  EXECS  AVG_ETIME AVG_PX OFFLOAD ELIGIBLE_MB  INTERCO_MB OFF_RETU_MB IO_SAVED_% SQL_TEXT
------------- ------ ----------- ------ ---------- ------ ------- ----------- ----------- ----------- ---------- ----------------------------------------------------------------------
5zzvpyn94b05g	   0   627556429      1       4.11	0 Yes	  19413.42969 2.860450745 2.860450745	   99.99 select /* WITH_SMART_SCAN_NO_STORAGEIDX */ id from bdt where id=1</pre>
<p>The elapsed time of the sql is <strong>4.11</strong> seconds and 99.99 % of IO has been saved by offload. About <strong>2.8 mb</strong> have been exchanged between the Oracle Database and the storage system.</p>
<p>The asm_metrics ouput is the following:</p>
<pre style="padding-left:30px;">06:41:54                                                                            Kby       Avg       AvgBy/               Kby       Avg        AvgBy/
06:41:54   INST     DBINST        DG            FG           DSK          Reads/s   Read/s    ms/Read   Read      Writes/s   Write/s   ms/Write   Write
06:41:54   ------   -----------   -----------   ----------   ----------   -------   -------   -------   ------    ------     -------   --------   ------
06:41:54   +ASM2                                                          4862      4969898   0.0       1046671   1          12        6.4        16384
06:41:54   +ASM2    BDT2                                                  4862      4969898   0.0       1046671   1          12        6.4        16384
06:41:54   +ASM2    BDT2                        ENKCEL01                  1678      1715380   0.0       1047123   0          4         17.2       16384
06:41:54   +ASM2    BDT2                        ENKCEL02                  1844      1883738   0.0       1046067   0          4         1.0        16384
06:41:54   +ASM2    BDT2                        ENKCEL03                  1341      1370780   0.0       1046935   0          4         1.0        16384</pre>
<p>So the average Kby Read per second is 4969898.</p>
<p>Then we can conclude that the BYTES_READ column records that about 4969898 *4.11 = <strong>19.5 gb</strong> has been read from disk.</p>
<p>Does it make sense? I would say yes, because the storage indexes haven't been used. Then, during the smart scan <strong>all the datas blocks have been opened </strong>in the cells in order to extract and send back to the database layer the requested column (column projection) on the selected rows (row filtering).</p>
<h3>Test 3: Offload and storage indexes</h3>
<pre>BDT:BDT2&gt; alter session set "_kcfis_storageidx_disabled"=false;

Session altered.

BDT:BDT2&gt; select /* WITH_SMART_SCAN_AND_STORAGEIDX */ id from bdt where id=1;</pre>
<p>The elapsed time and the percentage of IO saved are:</p>
<pre style="padding-left:30px;">SQL_ID	       CHILD   PLAN_HASH  EXECS  AVG_ETIME AVG_PX OFFLOAD ELIGIBLE_MB  INTERCO_MB OFF_RETU_MB IO_SAVED_% SQL_TEXT
------------- ------ ----------- ------ ---------- ------ ------- ----------- ----------- ----------- ---------- ----------------------------------------------------------------------
3jdpqa2s4bb0v	   0   627556429      1        .09	0 Yes	  19413.42969  .062171936  .062171936	  100.00 select /* WITH_SMART_SCAN_AND_STORAGEIDX */ id from bdt where id=1</pre>
<p>The elapsed time of the sql is <strong>0.09</strong> seconds and about 100 % of IO has been saved by offload. About <strong>0.06 mb</strong> have been exchanged between the Oracle Database and the storage system.</p>
<p>The storage indexes saved a lot of reads (almost the whole table):</p>
<pre>BDT:BDT2&gt; l
  1* select n.name, s.value from v$statname n, v$mystat s where n.name='cell physical IO bytes saved by storage index' and n.STATISTIC#=s.STATISTIC#
BDT:BDT2&gt; /
NAME							     VALUE
-------------------------------------------------- ---------------
cell physical IO bytes saved by storage index	       20215947264</pre>
<p>The asm_metrics ouput is the following:</p>
<pre>06:43:58                                                                            Kby        Avg       AvgBy/               Kby       Avg        AvgBy/
06:43:58   INST     DBINST        DG            FG           DSK          Reads/s   Read/s     ms/Read   Read      Writes/s   Write/s   ms/Write   Write
06:43:58   ------   -----------   -----------   ----------   ----------   -------   -------    -------   ------    ------     -------   --------   ------
06:43:58 +ASM2 19436 19879384 0.0 1047360 0 0 0.0 0 06:43:58 +ASM2 BDT2 19436 19879384 0.0 1047360 0 0 0.0 0 06:43:58 +ASM2 BDT2 ENKCEL01 6708 6861488 0.0 1047430 0 0 0.0 0 06:43:58 +ASM2 BDT2 ENKCEL02 7369 7534840 0.0 1047045 0 0 0.0 0 06:43:58 +ASM2 BDT2 ENKCEL03 5359 5483056 0.0 1047705 0 0 0.0 0

So the average Kby&nbsp;Read per second is 19879384.

As the asm\_metrics utility's granularity to collect the data is one second and as the elapsed time is \< 1s then we can conclude that the BYTES\_READ column records&nbsp;that about&nbsp; **19.5 gb** has been read from disk.

Does it make sense? I would say no, because during the smart scan, thanks to the Storage indexes, **not all the datas blocks** have been opened in the cells in order to extract the requested column (column projection) on the selected rows (row filtering).

## Remark

You could also query the _v$asm\_disk\_iostat_ view and measure the delta for the BYTES\_READ column by your own (means without the asm\_metrics utility). The results would be the same.

## Conclusion

The BYTES\_READ column displays:

- The&nbsp;Total number of bytes read from the disk (and also transferred to the database) **without** smart scan.
- The&nbsp;Total number of bytes read from the disk (but not the bytes transferred to the database) **with** smart scan and **no** storage indexes being used.
- Neither the&nbsp;Total number of bytes read from the disk nor the bytes transferred to the database **with** smart scan and storage indexes being used.

&nbsp;


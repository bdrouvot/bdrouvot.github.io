---
layout: post
title: Report database physical IO activity per IO type and oracle files type in real
  time
date: 2014-05-06 17:24:23.000000000 +02:00
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
  publicize_facebook_url: https://facebook.com/1430435633876621
  _wpas_done_5536151: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/XVnnsdNcH9U
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/K3lASzx4LL
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5869481630728495104&type=U&a=TNWW
  _wpas_done_2077996: '1'
  _oembed_dc74347f5e56d6d4afa0557fea6f93fb: "{{unknown}}"
  _oembed_b40c34c98e244f13fcfed00dc5ecdcd5: "{{unknown}}"
  _oembed_2989304f8535849ee88b8c307fb25dd3: "{{unknown}}"
  _oembed_659945e4f953e69b2e9c6dab4366ac53: "{{unknown}}"
  _oembed_3226eb56ec1b4453e12ab9f9a00aacdf: "{{unknown}}"
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/05/06/report-database-physical-io-activity-per-io-type-and-oracle-files-type-in-real-time/"
---
<p style="text-align:center;"><span style="text-decoration:underline;color:#008000;"><strong>Introduction</strong></span></p>
<p>After I published the blog post related to the <a title="Welcome to db_io_metrics, a new utility to display database physical IO metrics in real time" href="http://bdrouvot.wordpress.com/2014/04/30/welcome-to-db_io_metrics-a-new-utility-to-display-database-physical-io-metrics-in-real-time/" target="_blank">db_io_metrics</a> utiliy (based on gv$filestat and gv$tempstat), <a href="http://externaltable.blogspot.be/" target="_blank">Luca Canali</a> suggested me to create a version based on the <a href="http://docs.oracle.com/cd/E16655_01/server.121/e17615/refrn30441.htm#REFRN30441" target="_blank">gv$iostat_file</a> cumulative view (available since 11.1).</p>
<p><span style="text-decoration:underline;">The advantages of this view are:</span></p>
<ul>
<li>it displays information about disk I/O statistics on a lot of <strong>file types</strong> (including data files, temp files,control files, log files, archive logs, and so on).</li>
<li>it takes care of the <strong>IO type</strong> (small,large,reads, writes and synchronous).</li>
</ul>
<p>Then based on this, I created a new utility: <strong>db_io_type_metrics</strong></p>
<p style="text-align:center;"><span style="text-decoration:underline;color:#008000;"><strong>db_io_type_metrics description</strong></span></p>
<p style="text-align:left;">The db_io_type_metrics.pl utility is used to display database <strong>physical IO type (small, large, synchronous, reads and writes)</strong> real-time metrics. It basically takes a snapshot each second (default interval) from the gv$iostat_file cumulative view and computes the delta with the previous snapshot. The utility is RAC and Multitenant aware.</p>
<p><span style="text-decoration:underline;"><strong>This utility:</strong></span></p>
<ul>
<li>provides useful metrics.</li>
<li>is RAC aware.</li>
<li>detects if it is connected to a multitenant database and then is able to display the containers metrics.</li>
<li>is fully customizable: you can aggregate the results depending on your needs.</li>
<li>does not install anything into the database.</li>
</ul>
<p style="color:#555555;"><span style="text-decoration:underline;"><strong>It displays the following metrics per IO Type (small, large, synchronous, reads and writes):</strong></span></p>
<ul>
<li style="color:#555555;">MB/s: Megabytes per second.</li>
<li style="color:#555555;">RQ/s: Requests per second.</li>
<li style="color:#555555;">Avg MB/RQ: Average Megabytes per request.</li>
<li style="color:#555555;">Avg ms/RQ: Average ms per request.</li>
</ul>
<p style="color:#555555;"><span style="text-decoration:underline;"><strong>At the following levels:</strong></span></p>
<ul style="color:#555555;">
<li>Database Instance.</li>
<li>Database container.</li>
<li>File Type (log file, control file, data file, temp file…) or tablespace.</li>
</ul>
<p><span style="text-decoration:underline;"><strong>Let's see the help:</strong></span></p>
<pre style="padding-left:30px;">./db_io_type_metrics.pl -help

Usage: ./db_io_type_metrics.pl [-interval] [-count] [-inst] [-cont] [-file_type_tbs] [-io_type] [-file_type] [-tbs] [-show] [-display] [-sort_field] [-help]

 Default Interval : 1 second.
 Default Count    : Unlimited

  Parameter         Comment                                                                     Default
  ---------         -------                                                                     -------
  -INST=            ALL - Show all Instance(s)                                                  ALL
                    CURRENT - Show Current Instance
  -CONT=            Container to collect (wildcard allowed)                                     ALL
  -FILE_TYPE_TBS=   Collect on File Type or on Tablespace: file_type,tbs                        FILE_TYPE
  -IO_TYPE=         IO Type to collect: reads,writes,small,large,synch                          READS
  -FILE_TYPE=       File Type to collect (in case FILE_TYPE_TBS=file_type) (wildcard allowed)   NONE
  -TBS=             Tablespace to collect (in case FILE_TYPE_TBS=tbs) (wildcard allowed)        NONE
  -SHOW=            What to show: inst,cont,file_type_tbs (comma separated list)                INST
  -DISPLAY=         What to display: snap,avg (comma separated list)                            SNAP
  -SORT_FIELD=      small_reads,small_writes,large_reads,large_writes                           NONE

Example: ./db_io_type_metrics.pl
Example: ./db_io_type_metrics.pl  -inst=CBDT1
Example: ./db_io_type_metrics.pl  -show=inst,file_type_tbs
Example: ./db_io_type_metrics.pl  -show=inst,file_type_tbs -file_type=%Data%
Example: ./db_io_type_metrics.pl  -show=inst -io_type=large
Example: ./db_io_type_metrics.pl  -show=inst -io_type=small -sort_field=small_reads
Example: ./db_io_type_metrics.pl  -show=inst,file_type_tbs -file_type_tbs=tbs -tbs=%USE%
Example: ./db_io_type_metrics.pl  -show=inst,cont
Example: ./db_io_type_metrics.pl  -show=inst,cont -cont=%P%
Example: ./db_io_type_metrics.pl  -show=inst,cont,file_type_tbs -io_type=small -sort_field=small_reads
</pre>
<p><span style="text-decoration:underline;"><strong>The main options/features are:</strong></span></p>
<ol style="color:#555555;">
<li>You can choose the number of snapshots to display and the time to wait between snapshots.</li>
<li>You can choose on which <strong>database instance</strong> to collect the metrics thanks to the -<strong>INST=</strong> parameter.</li>
<li>You can choose on which <strong>database container</strong> to collect the metrics thanks to the <strong>-CONT=</strong> parameter.</li>
<li>You can choose to collect on <strong>File Type</strong> or <strong>tablespace</strong> thanks to the<strong> -FILE_TYPE_TBS=</strong>parameter.</li>
<li>You can choose on which <strong>IO Type</strong> to collect the metrics thanks to the <strong>-IO_TYPE=</strong> parameter.</li>
<li>You can choose on which <strong>File Type</strong> to collect the metric thanks to the <strong>-FILE_TYPE</strong>=parameter (wilcard allowed).</li>
<li>You can choose on which <b>Tablespace</b> to collect the metric thanks to the <strong>-TBS</strong>=parameter (wilcard allowed).</li>
<li>You can <strong>aggregate the results</strong> on the database instances, containers, file type or tablespace level thanks to the -<strong>SHOW=</strong> parameter.</li>
<li>You can display the metrics <strong>per snapshot, the average metrics</strong> value since the collection began (that is to say since the script has been launched) or both thanks to the -<strong>DISPLAY=</strong> parameter.</li>
<li>You can sort based on the number of <strong>small_reads</strong>, number of <strong>small_writes, </strong>number of<strong> large_reads </strong>or number of<strong> large_writes </strong>thanks to the -<strong>SORT_FIELD=</strong> parameter.</li>
</ol>
<p><span style="text-decoration:underline;"><strong style="color:#555555;text-decoration:underline;"><span style="text-decoration:underline;"><strong>Let’s see some use cases:</strong></span></strong></span></p>
<p><span style="color:#555555;">Report the IO type metrics for “reads” IO per database instances, tablespaces for the SLOB tablespace:</span></p>
<pre style="padding-left:30px;"> ./db_io_type_metrics.pl  -show=inst,file_type_tbs -file_type_tbs=tbs -tbs=%SLOB% -io_type=reads
............................
Collecting 1 sec....
............................

......... SNAP TAKEN AT ...................

10:52:54                                               SMALL R   SMALL R   LARGE R   LARGE R   Avg MB/   Avg MB/   Avg ms/   Avg ms/
10:52:54   INST         FILE_TYPE_TBS                  MB/s      RQ/s      MB/s      RQ/s      SMALL R   LARGE R   SMALL R   LARGE R
10:52:54   ----------   ----------------------------   -------   -------   -------   -------   -------   -------   -------   -------
10:52:54   BDT_1                                       144       18390     0         0         0.008     0.0       0.08      0.00
10:52:54   BDT_1        SLOB                           144       18390     0         0         0.008     0.0       0.08      0.00
</pre>
<p>&nbsp;</p>
<p>Report the IO type metrics for “small” IO” and all file type:</p>
<pre style="padding-left:30px;">./db_io_type_metrics.pl  -show=file_type_tbs -file_type_tbs=file_type -io_type=small
............................
Collecting 1 sec....
............................

......... SNAP TAKEN AT ...................

10:57:01                                               SMALL R   SMALL R   SMALL W   SMALL W   Avg MB/   Avg MB/   Avg ms/   Avg ms/
10:57:01   INST         FILE_TYPE_TBS                  MB/s      RQ/s      MB/s      RQ/s      SMALL R   SMALL W   SMALL R   SMALL W
10:57:01   ----------   ----------------------------   -------   -------   -------   -------   -------   -------   -------   -------
10:57:01                Archive Log                    0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Archive Log Backup             0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Control File                   1         66        0         0         0.015     0.000     0.00      0.00
10:57:01                Data File                      125       16230     0         0         0.008     0.000     0.15      0.00
10:57:01                Data File Backup               0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Data File Copy                 0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Data File Incremental Backup   0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Data Pump Dump File            0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Flashback Log                  0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Log File                       0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Other                          0         0         0         0         0.000     0.000     0.00      0.00
10:57:01                Temp File                      0         0         0         0         0.000     0.000     0.00      0.00
</pre>
<p>&nbsp;</p>
<p><span style="color:#555555;">Report the IO type metrics for “small” IO per database instances and containers:</span></p>
<pre style="padding-left:30px;">./db_io_type_metrics.pl -show=inst,cont -io_type=small
............................
Collecting 1 sec....
............................

......... SNAP TAKEN AT ...................

13:07:46                                                                 SMALL R   SMALL R   SMALL W   SMALL W   Avg MB/   Avg MB/   Avg ms/   Avg ms/
13:07:46   INST         CONT              FILE_TYPE_TBS                  MB/s      RQ/s      MB/s      RQ/s      SMALL R   SMALL W   SMALL R   SMALL W
13:07:46   ----------   ---------------   ----------------------------   -------   -------   -------   -------   -------   -------   -------   -------
13:07:46 CBDT1 2 97 0 1 0.021 0.000 0.03 1.00 13:07:46 CBDT1 CDB$ROOT 2 97 0 1 0.021 0.000 0.03 1.00 13:07:46 CBDT1 PDB$SEED 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT1 P\_1 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT1 P\_2 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT1 P\_3 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT2 1 96 0 1 0.010 0.000 0.09 2.00 13:07:46 CBDT2 CDB$ROOT 1 96 0 1 0.010 0.000 0.09 2.00 13:07:46 CBDT2 PDB$SEED 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT2 P\_1 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT2 P\_2 0 0 0 0 0.000 0.000 0.00 0.00 13:07:46 CBDT2 P\_3 0 0 0 0 0.000 0.000 0.00 0.00

&nbsp;

Report the IO type metrics per… I let you finish the sentence as the utility is customizable enough ;-)

You can **download** it from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](http://bdrouvot.wordpress.com/db_io_type_metrics_source_code/ "db\_io\_type\_metrics\_source").[  
](http://bdrouvot.wordpress.com/db_io_metrics_source/ "db\_io\_metrics\_source")

**Conclusion:**

Three utilities are now available to report IO metrics in real time depending&nbsp;on our needs:

1. [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm\_metrics") for reads and writes metrics related to ASM.
2. [db\_io\_metrics](http://bdrouvot.wordpress.com/db_io_metrics_script/ "db\_io\_metrics") for reads and writes metrics related to data files and temp files.
3. [db\_io\_type\_metrics](http://bdrouvot.wordpress.com/db_io_type_metrics_script/ "db\_io\_type\_metrics") for reads, writes, small, large and synchronous metrics related to data files, temp files,control files, log files, archive logs, and so on.

Enjoy and don't hesitate to come back to me in case of questions, issues or suggestions.


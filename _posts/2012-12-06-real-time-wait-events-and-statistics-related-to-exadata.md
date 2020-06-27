---
layout: post
title: Real-Time Wait Events and Statistics related to Exadata
date: 2012-12-06 20:31:06.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
- ToolKit
tags:
- exadata
- perl scripts
- real-time
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:3;}s:2:"wp";a:1:{i:0;i:3;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/06/real-time-wait-events-and-statistics-related-to-exadata/"
---
<p>Into my first post related to Exadata I provided a perl script to extract real-time metrics <span style="color:#0000ff;">from the cells</span> based on their cumulative metrics (<a title="Exadata real-time metrics extracted from cumulative metrics" href="http://bdrouvot.wordpress.com/2012/11/27/exadata-real-time-metrics-extracted-from-cumulative-metrics/" target="_blank">see here</a>).</p>
<p>In this post I focus on the information available <span style="color:#0000ff;">from the database</span>: Cumulative "cell" Statistics and cumulative "cell" Wait Events related to Exadata (Uwe described the most important ones into this <a href="http://uhesse.com/2011/07/06/important-statistics-wait-events-on-exadata/" target="_blank">post</a>).</p>
<p><span style="color:#0000ff;">As usual the philosophy behind the 3 scripts used into this post is</span>: extract real-time information from the cumulative views by taking a snapshot each second (default interval) and computes the differences with the previous snapshot.</p>
<p><span style="color:#0000ff;">So to collect Real-Time Wait Events related to Exadata</span>, I use a perl script "<a title="system_event" href="http://bdrouvot.wordpress.com/system_event/" target="_blank">system_event.pl</a>" (click on the link and then on the view source button  to copy/paste the source code) already used into one of my previous post "<a title="Measure oracle real-time I/O performance" href="http://bdrouvot.wordpress.com/2012/11/20/measure-oracle-real-time-io-performance/" target="_blank">Measure oracle real-time I/O performance</a>".</p>
<p><span style="text-decoration:underline;"><span style="text-decoration:underline;">But this time I focus only on "cell" Wait Events related to Exadata that way :</span></span></p>
<pre>./system_event.pl event_like='%cell%'

02:30:41 INST_NAME	EVENT				  NB_WAITS	TIME_WAITED		ms/Wait
02:30:41 BDT1		cell smart file creation 	  7 		950197 			135.742
--------------------------------------&gt; NEW
02:30:42 INST_NAME	EVENT 				  NB_WAITS 	TIME_WAITED 		ms/Wait
02:30:42 BDT1 		cell smart file creation 	  3 		543520 			181.173
02:30:42 BDT1 		cell single block physical read   3 		4420 			1.473</pre>
<p>As you can see, during the last second the database waited 3 times on "Cell smart file creation"</p>
<p><span style="color:#0000ff;">Now to collect Real-Time Statistics related to Exadata</span>, I use the "<a title="sysstat" href="http://bdrouvot.wordpress.com/sysstat/" target="_blank">sysstat.pl</a>" script (click on the link and then on the view source button to copy/paste the source code) that way:</p>
<pre>./sysstat.pl statname_like='%cell%'

03:15:29   INST_NAME	NAME                                                               VALUE
03:15:29   BDT1		cell smart IO session cache lookups                                11
03:15:29   BDT1		cell scans                                                         17
03:15:29   BDT1		cell blocks processed by cache layer                               3551
03:15:29   BDT1		cell blocks processed by txn layer                                 3551
03:15:29   BDT1		cell blocks processed by data layer                                3551
03:15:29   BDT1		cell blocks helped by minscn optimization                          3551
03:15:29   BDT1		cell physical IO interconnect bytes returned by smart scan         1421352
03:15:29   BDT1		cell physical IO interconnect bytes                                2756648
03:15:29   BDT1		cell IO uncompressed bytes                                         29089792
03:15:29   BDT1		cell physical IO bytes eligible for predicate offload              29089792
--------------------------------------&gt; NEW
03:15:30   INST_NAME	NAME                                                               VALUE
03:15:30   BDT1		cell smart IO session cache lookups                                33
03:15:30   BDT1		cell scans                                                         33
03:15:30   BDT1		cell blocks processed by cache layer                               9896
03:15:30   BDT1		cell blocks processed by txn layer                                 9896
03:15:30   BDT1		cell blocks processed by data layer                                9896
03:15:30   BDT1		cell blocks helped by minscn optimization                          9896
03:15:30   BDT1		cell physical IO interconnect bytes returned by smart scan         6602936
03:15:30   BDT1		cell physical IO interconnect bytes                                6701240
03:15:30   BDT1		cell IO uncompressed bytes                                         81068032
03:15:30   BDT1		cell physical IO bytes eligible for predicate offload              81068032</pre>
<p>As you can see during the last second the statistic cell physical IO bytes has increased by 81068032 .<br />
If we need to diagnose more in depth and link in real-time those "cell" statistics with one or more sql_id, we can use the "<a title="sqlidstat" href="http://bdrouvot.wordpress.com/perl-scripts/" target="_blank">sqlidstat.pl</a>" (click on the link and then on the view source button  to copy/paste the source code) that way:</p>
<pre>./sqlidstat.pl statname_like='%cell%'

--------------------------------------&gt; NEW
03:15:29   SID   SQL_ID         NAME	        		   			    VALUE
03:15:29   ALL   0ab8xuf6kuud5  cell smart IO session cache hits                            4
03:15:29   ALL   0ab8xuf6kuud5  cell scans                                                  10
03:15:29   ALL   0ab8xuf6kuud5  cell blocks processed by cache layer                        1193
03:15:29   ALL   0ab8xuf6kuud5  cell blocks processed by data layer                         1193
03:15:29   ALL   0ab8xuf6kuud5  cell blocks processed by txn layer                          1193
03:15:29   ALL   0ab8xuf6kuud5  cell blocks helped by minscn optimization                   1193
03:15:29   ALL   0ab8xuf6kuud5  cell physical IO interconnect bytes returned by smart scan  356096
03:15:29   ALL   0ab8xuf6kuud5  cell physical IO interconnect bytes                         2232064
03:15:29   ALL   0ab8xuf6kuud5  cell IO uncompressed bytes                                  9773056
03:15:29   ALL   0ab8xuf6kuud5  cell physical IO bytes eligible for predicate offload       9773056
--------------------------------------
\> NEW 03:15:30 SID SQL\_ID NAME VALUE 03:15:30 ALL 0ab8xuf6kuud5 cell smart IO session cache hits 34 03:15:30 ALL 0ab8xuf6kuud5 cell scans 34 03:15:30 ALL 0ab8xuf6kuud5 cell blocks processed by cache layer 10522 03:15:30 ALL 0ab8xuf6kuud5 cell blocks processed by data layer 10522 03:15:30 ALL 0ab8xuf6kuud5 cell blocks processed by txn layer 10522 03:15:30 ALL 0ab8xuf6kuud5 cell blocks helped by minscn optimization 10522 03:15:30 ALL 0ab8xuf6kuud5 cell physical IO interconnect bytes returned by smart scan 6039016 03:15:30 ALL 0ab8xuf6kuud5 cell physical IO interconnect bytes 6039016 03:15:30 ALL 0ab8xuf6kuud5 cell IO uncompressed bytes 86196224 03:15:30 ALL 0ab8xuf6kuud5 cell physical IO bytes eligible for predicate offload 86196224

I removed the "INST\_NAME" column for lisibilty.

By default all the SID have been aggregated but you could also filter on a particular sid (see the help).  
So we can see that during the last second, most of the offload processing that we observed at the database level is related to the sql\_id&nbsp;0ab8xuf6kuud5.

Remarks :

- Those 3 scripts **are not exclusively related to Exadata** as you can use them on all the wait events or statistics, I simply used them with the event\_like or statname\_like arguments to focus on '%cell%' .
- You can choose the number of snapshots to display and the time to wait between snapshots.
- The scripts are oracle RAC aware : you can work on all the instances, a subset or the local one.
- You have to set oraenv on one instance of the database you want to diagnose first.

Part II of this subject available [here](http://bdrouvot.wordpress.com/2013/01/04/real-time-wait-events-and-statistics-related-to-exadata-part-ii/ "Real-Time Wait Events and Statistics related to Exadata : Part II").


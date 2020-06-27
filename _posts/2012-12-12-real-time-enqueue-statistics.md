---
layout: post
title: Real-Time enqueue statistics
date: 2012-12-12 21:20:45.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
- ToolKit
tags:
- perl scripts
- real-time
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:8;}s:2:"wp";a:1:{i:0;i:5;}}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/12/real-time-enqueue-statistics/"
---
<p>Again the same story : Oracle provides a useful view to check enqueue statistics (v$enqueue_statistics), but this view is a cumulative one. So how to check what's is going on my database right now with cumulative values ?</p>
<p>You have to substract values as it has been done into <a href="http://www.pythian.com/news/1008/resolving-hw-enqueue-contention/" target="_blank">Riyaj Shamsudeen's post</a>.</p>
<p>So to get real-time enqueue statistics, you can use the <a title="enqueue_statistics" href="http://bdrouvot.wordpress.com/enqueue_statistics/" target="_blank">enqueue_statistics.pl</a> script (click on the link and then on the view source button to copy/paste the source code) that basically takes a snapshot based on the gv$enqueue_statistics view each second (default interval) and computes the differences with the previous snapshot.</p>
<p><span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example:</span></span></p>
<pre>./enqueue_statistics.pl

20:01:10   INST_NAME	EQ_NAME			EQ_TYPE		REQ_REASON	TOTAL_REQ 
20:01:10   BDT_1	Job Queue Date		JD		contention	1
20:01:10   BDT_1	Job Queue		JQ		contention	2 
20:01:10   BDT_1	Job Scheduler		JS		q mem clnup lck	2
20:01:10   BDT_1	Session Migration	SE		contention	4
20:01:10   BDT_1 	Job Scheduler		JS		contention	15
20:01:10   BDT_1 	Job Scheduler		JS		queue lock	15 
20:01:10   BDT_1	Media Recovery		MR         	contention	28    
20:01:10   BDT_1	Cursor		        CU		contention	107   
20:01:10   BDT_1	Transaction		TX		contention	122
20:01:10   BDT_1	DML	                TM		contention	166
--------------------------------------&gt; NEW
20:01:11   INST_NAME	EQ_NAME			EQ_TYPE		REQ_REASON	TOTAL_REQ 
20:01:11   BDT_1	Job Queue Date		JD		contention	1
20:01:11   BDT_1	Job Queue		JQ		contention	1
20:01:11   BDT_1	Job Scheduler		JS		q mem clnup lck	1
20:01:11   BDT_1	Session Migration	SE		contention	2
20:01:11   BDT_1 	Job Scheduler		JS		contention	12
20:01:11   BDT_1 	Job Scheduler		JS		queue lock	17 
20:01:11   BDT_1	Media Recovery		MR         	contention	20    
20:01:11   BDT_1	Cursor		        CU		contention	100   
20:01:11   BDT_1	Transaction		TX		contention	134
20:01:11   BDT_1	DML	                TM		contention	185</pre>
<p>As you can see during the last second the DML enqueue has been requested 185 times.</p>
<p>For lisiblity, I removed some columns into the output that are displayed by the script: TOTAL_WAIT,SUCC_REQ,FAILED_REQ,WAIT_TIME</p>
<p>Output is ordered based on the TOTAL_REQ column but you could choose to order on any column.</p>
<p><span style="text-decoration:underline;color:#0000ff;">Let's see the help:</span></p>
<pre>./enqueue_statistics.pl help                   

Usage: enqueue_statistics.pl [Interval [Count]] [inst=] [top=] [eq_name=] [eq_type=] [req_reason=] [sort_field=]
        Default Interval : 1 second.
        Default Count    : Unlimited

  Parameter	    Comment							Default    
  ---------         -------                                                     -------
&nbsp; &nbsp; &nbsp; INST= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ALL - Show all Instance(s) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; CURRENT - Show Current Instance &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; INSTANCE\_NAME,... - choose Instance(s) to display &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;\<\< Instances are only displayed in a RAC DB \>\> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; TOP= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Number of rows to display &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; EQ\_NAME= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL - Show all ENQ NAME (wildcard allowed) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; EQ\_TYPE= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL - Show all ENQ TYPE (wildcard allowed) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; REQ\_REASON= &nbsp; &nbsp; &nbsp; ALL - Show all REASONS (wildcard allowed) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; SORT\_FIELD= &nbsp; &nbsp; &nbsp; TOTAL\_REQ|TOTAL\_WAIT|SUCC\_REQ|FAILED\_REQ|WAIT\_TIME &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; TOTAL\_REQ &nbsp; Example : enqueue\_statistics.pl Example : enqueue\_statistics.pl sort\_field=WAIT\_TIME Example : enqueue\_statistics.pl eq\_name='%sactio%' sort\_field=WAIT\_TIME Example : enqueue\_statistics.pl req\_reason='contention'

As usual:

- You can choose the number of snapshots to display and the time to wait between snapshots.
- You can choose to filter on enqueue name, enqueue type and enqueue reason (by default no filter is applied).
- You can choose the column to sort the output on (new).
- This script is oracle RAC aware&nbsp;: you can work on all the instances, a subset or the local one.
- You have to set oraenv on one instance of the database you want to diagnose first.
- The script has been tested on Linux, Unix and Windows.

Remark:

I'll provide a few more real-time&nbsp;utility&nbsp;scripts (see [this page](http://bdrouvot.wordpress.com/perl-scripts-2/ "Perl Scripts") for the existing ones) and after that I'll put all those scripts together into a single "real-time" utility script.


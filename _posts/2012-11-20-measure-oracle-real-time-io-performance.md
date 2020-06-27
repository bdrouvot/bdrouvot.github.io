---
layout: post
title: Measure oracle real-time I/O performance
date: 2012-11-20 10:30:32.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
- ToolKit
tags:
- I/O
- performance
- perl scripts
- real-time
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2094533: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  publicize_twitter_user: bdteur
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/11/20/measure-oracle-real-time-io-performance/"
---
<p>Did you already need to measure real-time I/O performance of an oracle database ?</p>
<p>AWR could provide an answer for historical period of time but what's about:  what is going on on my database right now ?</p>
<p>To measure real-time I/O performance I use one of my perl script (<a title="system_event" href="http://bdrouvot.wordpress.com/system_event/" target="_blank">system_event.pl</a>) based on the gv$system_event cumulative view.</p>
<p>As this view is a cumulative one (values are gathered since the instance started up), to extract real-time information the script takes a snapshot each second (default interval) and computes the differences with the previous snapshot.</p>
<p>Let's see an example: as this post deals with IO performance, we can focus on this particular wait class that way:</p>
<pre>./system_event.pl waitclass="User I/O"

 18:27:36 INST_NAME 	EVENT				NB_WAITS	TIME_WAITED 	ms/Wait
 18:27:36 BDTO_2 	Disk file operations I/O	1 		77 		0.077
 18:27:36 BDTO_2 	db file scattered read 		241 		1423749 	5.908
 18:27:36 BDTO_2 	db file sequential read 	260 		1168489 	4.494
 18:27:36 BDTO_2 	db file parallel read 		626 		10447664 	16.690
 --------------------------------------&gt; NEW
 18:27:37 INST_NAME 	EVENT 				NB_WAITS 	TIME_WAITED 	ms/Wait
 18:27:37 BDTO_2 	Disk file operations I/O 	1 		84 		0.084
 18:27:37 BDTO_2 	db file scattered read 		205 		1128732		5.506
 18:27:37 BDTO_2 	db file sequential read 	261 		1196085 	4.583
 18:27:37 BDTO_2 	db file parallel read 		592 		9224011 	15.581</pre>
<p>So, during the last second we got 261 "db file sequential read" events for an average of 4.583 ms per wait.</p>
<p>You could get more detailed information thanks to the gv$event_histogram view (benefit of this view is described in this <a href="http://jonathanlewis.wordpress.com/2007/01/10/event-histograms/" target="_blank">post</a>).</p>
<p>As this view is a cumulative one, to extract real-time information, I wrote the <a title="event_histogram" href="http://bdrouvot.wordpress.com/event_histogram/" target="_blank">event_histogram.pl</a> perl script.</p>
<pre>./event_histogram.pl event='db file sequential read'

18:32:52 INST_NAME 	EVENT 				WAIT_TIME_MS            COUNT
18:32:52 BDTO_1 	db file sequential read 	1 			0
18:32:52 BDTO_2 	db file sequential read 	2 			0
18:32:52 BDTO_2 	db file sequential read 	256 			0
18:32:52 BDTO_1 	db file sequential read 	4 			0
18:32:52 BDTO_2 	db file sequential read 	8 			0
18:32:52 BDTO_1 	db file sequential read 	8 			0
18:32:52 BDTO_1 	db file sequential read 	512 			0
18:32:52 BDTO_1 	db file sequential read 	16 			0
18:32:52 BDTO_2 	db file sequential read 	4 			2
18:32:52 BDTO_2 	db file sequential read 	1 			5</pre>
<p>In this example we can see that 5 "df file sequential read" took less than 1ms to complete during the last second.</p>
<p>Of course those two perl scripts deal with all oracle wait events and not only the ones related to the User I/O wait class:  then you can extend their usage area following your needs on system events.</p>
<p>See the help for more details:</p>
<pre>./system_event.pl help

Usage: ./system_event.pl [Interval [Count]] [inst] [top=] [waitclass=Concurrency|User I/O|System I/O|Administrative|Configuration|Other|Cluster|Application|Idle|Commit|Network] [event=]
Default Interval : 1 second.
 Default Count : Unlimited
Parameter 				Comment 					Default 
--------- 				------- 					-------
 INST= ALL - Show all Instance(s) ALL  CURRENT - Show Current Instance  INSTANCE\_NAME,... - choose Instance(s) to display \<\< Instances are only displayed in a RAC DB \>\> TOP= Number of rows to display 10  WAITCLASS=Concurrency|User I/O|System I/O|Administrative|Configuration|Other|Cluster|Application|Idle|Commit|Network ALL  EVENT= ALL - Show all EVENTS ALL

As usual :

- You can choose the number of snapshots to display and the time to wait between snapshots.
- You can choose to filter on wait\_class and on a particular event (by default no filter is applied).
- This script is oracle RAC aware&nbsp;: you can work on all the instances, a subset or the local one.
- You have to set oraenv on one instance of the database you want to diagnose first.
- The script has been tested on Linux, Unix and Windows.

---
layout: post
title: Real-Time CPU utilization per oracle database
date: 2012-12-04 21:22:07.000000000 +01:00
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
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:2;}s:2:"wp";a:1:{i:0;i:3;}}
  _publicize_pending: '1'
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/04/real-time-cpu-utilization-per-oracle-database/"
---
<p>Suppose you have a lot of oracle databases running on the same host, the server is almost 100% cpu used and you would like to know which database is the top cpu consumer right now.</p>
<p>The well known "top" utility could help ? This is not sure in case the culprit database has a lot of active sessions as "top" works at the process level.</p>
<p>So, to answer this need, I wrote a perl script that collects real-time processes cpu consumption at the os level (the script does not connect to any database) and aggregate the result per database.</p>
<p>The script takes a snapshot of os processes each second (default interval), computes the differences with the previous snapshot and aggregate te result per database.</p>
<p><span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example</span></span> of the <a title="os_cpu_per_db" href="http://bdrouvot.wordpress.com/os_cpu_per_dp/" target="_blank">os_cpu_per_db.pl</a> perl script (click on the link and then on the view source button to copy/paste the source code):</p>
<pre>./os_cpu_per_db.pl 

Collecting during 1 seconds......

                               SUMMARY PER DB

20:31:56       DB_NAME		CPU_SEC		NB_CPU    
20:31:56       MYDB1		0               0.0
20:31:56       MYDB3		2		2.0
20:31:56       MYDB2		23		23.0</pre>
<p>So we find out our top oracle database cpu consumer: the MYDB2 oracle database spent 23 seconds into the cpus during the last second (default collection interval) that is to say used 23 cpus.</p>
<p>Once you have the top database cpu consumer you can connect to it and diagnose more in depth (Example of cpu spike <a href="http://srivenukadiyala.wordpress.com/2012/01/30/sched_noage-and-latch-contention/" target="_blank">on this post</a>).</p>
<p>The script does a little bit more in case the cpu issue is not related to any oracle database:</p>
<ul>
<li><span style="line-height:13px;">It can display the cpu usage per os user.</span></li>
<li>It can display the top os PID and commands.</li>
</ul>
<p><span style="text-decoration:underline;color:#0000ff;">Let's see the help:</span></p>
<pre>./os_cpu_per_db.pl help                       

Usage: ./os_cpu_per_db.pl [Interval [Count]] [top=] [displayuser=[Y|N]] [displaycmd=[Y|N]] 

        Default Interval : 1 second.
        Default Count    : Unlimited

Parameter               Comment					Default    
---------               -------      		                -------
&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; TOP= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Number of rows to display &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 10 &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; DISPLAYUSER= REPORT ON USER TOO &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; N &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; DISPLAYCMD= &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; REPORT ON COMMAND TOO &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; N

- You can choose the number of snapshots to display and the time to wait between snapshots.
- You have to set .oraenv on one instance first.
- The script has been tested on Linux and Solaris.

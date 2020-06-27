---
layout: post
title: Real-Time SGA component monitoring
date: 2012-12-11 21:58:00.000000000 +01:00
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
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:6;}s:2:"wp";a:1:{i:0;i:5;}}
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/11/real-time-sga-component-monitoring/"
---
<p>From the V$SGA_RESIZE_OPS view, you observed that your database is doing frequent resize.</p>
<p>You want to know what's going on and you decided to query the v$sgastat view at regular interval to understand which component of the sga is growing.</p>
<p>The <a title="sgastat" href="http://bdrouvot.wordpress.com/sgastat/" target="_blank">sgastat.pl</a> script (click on the link and then on the view source button to copy/paste the source code) do it for you : It takes a snapshot based on the gv$sgastat view each second (default interval) and computes the differences with the previous snapshot.</p>
<p><span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example:</span></span></p>
<pre>./sgastat.pl

16:48:54   INST_NAME	POOL	        NAME		        BYTES                         
16:48:54   BDT_1	shared pool	QSMQUTL summar	        32                            
16:48:54   BDT_1	shared pool	kzull		        512                           
16:48:54   BDT_1	shared pool	kksss		        1744                          
16:48:54   BDT_1	shared pool	kksss-heap	        4280                          
16:48:54   BDT_1	shared pool	parameter handle	7192                          
16:48:54   BDT_1	shared pool	library cache		10936                         
16:48:54   BDT_1	shared pool	PCursor			12544                         
16:48:54   BDT_1	shared pool	parameter table block	51896                         
16:48:54   BDT_1	shared pool	CCursor			449688                        
16:48:54   BDT_1	shared pool	sql area		6715208                       
--------------------------------------&gt; NEW
16:48:55   INST_NAME	POOL	        NAME			BYTES                         
16:48:55   BDT_1	shared pool	kzull			128                           
16:48:55   BDT_1	shared pool	KTCCC OBJECT		144                           
16:48:55   BDT_1	shared pool	kksss			416                           
16:48:55   BDT_1	shared pool	kksss-heap		1040                          
16:48:55   BDT_1	shared pool	parameter handle	1776                          
16:48:55   BDT_1	shared pool	parameter table block	12920                         
16:48:55   BDT_1	shared pool	library cache		35904                         
16:48:55   BDT_1	shared pool	PCursor			67856                         
16:48:55   BDT_1	shared pool	CCursor			942688                        
16:48:55   BDT_1	shared pool	sql area		8876448</pre>
<p>So as you can see, the sql area component grow by about 8MB during the last second.</p>
<p><span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see the help:</span></span></p>
<pre>./sgastat.pl help

Usage: ./sgastat.pl [Interval [Count]] [inst=] [top=] [pool=] [name=] 
        Default Interval : 1 second.
        Default Count    : Unlimited

  Parameter    Comment                               			Default    
  ---------    -------                                                	-------
&nbsp; &nbsp; &nbsp; INST= &nbsp; &nbsp; &nbsp; &nbsp;ALL - Show all Instance(s) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;CURRENT - Show Current Instance &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;INSTANCE\_NAME,... - choose Instance(s) to display &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; \<\< Instances are only displayed in a RAC DB \>\> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; TOP= &nbsp; &nbsp; &nbsp; &nbsp; Number of rows to display &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; POOL= &nbsp; &nbsp; &nbsp; &nbsp;ALL - Show all POOL (wilcard allowed) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; NAME= &nbsp; &nbsp; &nbsp; &nbsp;ALL - Show all NAME (wilcard allowed) &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ALL &nbsp; &nbsp; &nbsp; &nbsp; Example : ./sgastat.pl&nbsp; Example : ./sgastat.pl pool='%shared%' Example : ./sgastat.pl name='%free%' Example : ./sgastat.pl pool='%shared%' name='%sql%'

As usual:

- You can choose the number of snapshots to display and the time to wait between snapshots.
- You can choose to filter on pool and name (by default no filter is applied).
- This script is oracle RAC aware&nbsp;: you can work on all the instances, a subset or the local one.
- You have to set oraenv on one instance of the database you want to diagnose first.
- The script has been tested on Linux, Unix and Windows.

You can found a very good study about shared pool management in [Coskan's post](http://coskan.wordpress.com/2007/09/14/what-i-learned-about-shared-pool-management/).


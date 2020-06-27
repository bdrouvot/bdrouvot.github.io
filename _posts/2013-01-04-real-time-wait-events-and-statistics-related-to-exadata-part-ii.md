---
layout: post
title: 'Real-Time Wait Events and Statistics related to Exadata : Part II'
date: 2013-01-04 18:39:04.000000000 +01:00
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
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:36;}s:2:"wp";a:1:{i:0;i:10;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/01/04/real-time-wait-events-and-statistics-related-to-exadata-part-ii/"
---
<p>Into my <a title="Real-Time Wait Events and Statistics related to Exadata" href="http://bdrouvot.wordpress.com/2012/12/06/real-time-wait-events-and-statistics-related-to-exadata/" target="_blank">first blog entry</a> on this topic I used 3 scripts to get real-time statistics at the database and session/sql_id level and wait events at the database level:</p>
<p><a title="system_event" href="http://bdrouvot.wordpress.com/system_event/" target="_blank">system_event.pl</a><br />
<a title="sysstat" href="http://bdrouvot.wordpress.com/sysstat/" target="_blank">sysstat.pl</a><br />
<a title="sqlidstat" href="http://bdrouvot.wordpress.com/sqlidstat/" target="_blank">sqlidstat.pl</a></p>
<p>This new post entry add a new one :</p>
<p><a title="sqlidevent" href="http://bdrouvot.wordpress.com/sqlidevent/" target="_blank">sqlidevent.pl</a> (click on the link and then on the view source button to copy/paste the source code) that is usefull to diagnose more in depth real-time relationship between sql_id, sessions and wait events.</p>
<p>It basically takes a snapshot each second (default interval) of the v$session_event cumulative view (values are gathered since the session started up) and computes the differences with the previous snapshot.</p>
<p><span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">For example, let's check sql_id in relation with smart table scan:</span></span></p>
<pre>./sqlidevent.pl event='%cell%'
02:00:02 INST_NAME SID	SQL_ID	        EVENT	                NB_WAITS	TIME_WAITED	ms/Wait
02:00:02 BDT1	   ALL	100dajbkzu295	cell smart table scan	9	        408816	        45.424
--------------------------------------
\> NEW 02:00:03 INST\_NAME SID SQL\_ID EVENT NB\_WAITS TIME\_WAITED ms/Wait 02:00:03 BDT1 ALL 100dajbkzu295 cell smart table scan 4 273434 68.359

As you can see during the last second 4 "cell smart table scan" wait events occurred to sql\_id "100dajbkzu295".

By default all the SID have been aggregated but you could also filter on a particular sid and/or on a particular sql\_id (see the help).

Remarks:

- Those 4 scripts **are not exclusively related to Exadata** as you can use them on all the wait events or statistics, I simply used them with filters to focus on ‘%cell%’ .
- You can choose the number of snapshots to display and the time to wait between snapshots.
- The scripts are oracle RAC aware : you can work on all the instances, a subset or the local one.
- You have to set oraenv on one instance of the database you want to diagnose first.

**Conclusion:**

We now have 4 scripts at our disposal to diagnose real-time wait events and statistics at the database level and at the session/sql\_id level.


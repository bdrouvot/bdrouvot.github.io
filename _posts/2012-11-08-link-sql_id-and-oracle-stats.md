---
layout: post
title: Link sql_id and oracle stats
date: 2012-11-08 21:28:18.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ToolKit
tags:
- perl scripts
- real-time
meta:
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  publicize_twitter_user: bdteur
  _publicize_pending: '1'
  _wpas_done_2094533: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/11/08/link-sql_id-and-oracle-stats/"
---
<p>How many times working on a performance issue, finding the Top SQL and Top Waits event has not been enough to understand what's going on ?</p>
<p>Sometimes you have to diagnose more in depth thanks to the Oracle stats reported into the V$SESSTAT view.</p>
<p>But how to quickly answer :</p>
<p>- Which sql is linked to this stat ?</p>
<p>- Which stats is linked to this sql ?</p>
<p>To answer those questions I wrote a perl script (<a title="sqlidstat" href="http://bdrouvot.wordpress.com/sqlidstat/" target="_blank">sqlidstat.pl</a>) :</p>
<ul>
<li>This perl script takes snapshots from the GV$SESSION and  GV$SESSTAT views.</li>
<li>As the GV$SESSTAT view is a cumulative one (values are gathered since the session started up), to extract real-time information the script takes a snapshot each second (default interval) and computes the differences with the previous snapshot.</li>
<li>You can choose the number of snapshots to display and the time to wait between snapshots.</li>
<li>You can choose to filter on sql_id, sid and on a particular stat (by default no filter is applied).</li>
<li><span style="color:#333399;">This script is oracle RAC aware</span> : you can work on all the instances, a subset or the local one.</li>
<li>You have to set oraenv on one instance of the database you want to diagnose first.</li>
<li>The script has been tested on Linux, Solaris and Windows.</li>
</ul>
<p><span style="text-decoration:underline;">Usage Examples:</span></p>
<p><span style="color:#0000ff;">The help : </span></p>
<pre>./sqlidstat.pl help
Usage:./sqlidstat.pl [Interval [Count]] [inst] [top=] [sql_id=] [sid=] [name="statname"]
Default Interval : 1 second.
Default Count : Unlimited

Parameter                  Comment                      Default
---------                  -------                      -------
INST= ALL - Show all Instance(s) ALL CURRENT Show Current Instance INSTANCE\_NAME,... choose Instance(s) to display \<\< Instances are only displayed in a RAC DB \>\> TOP= Number of rows to display 10 SQL\_ID= ALL - Show all sql\_id ALL SID= ALL - Show all SID ALL NAME= ALL - Show all STATNAME ALL

Launch it on all instances :

```
./sqlidstat.pl 12:17:11 INST\_NAME SQL\_ID NAME VALUE 12:17:11 BDTO\_1 ff9853fmqsbdu table scan rows gotten 52562 12:17:11 BDTO\_2 756yagxj5mmzn session uga memory max 65512 12:17:11 BDTO\_2 756yagxj5mmzn session pga memory max 65536 12:17:11 BDTO\_1 ff9853fmqsbdu session pga memory 524288 12:17:11 BDTO\_1 ff9853fmqsbdu file io wait time 634055 12:17:11 BDTO\_1 ff9853fmqsbdu session uga memory 982264 12:17:11 BDTO\_1 ff9853fmqsbdu physical read bytes 8404992 12:17:11 BDTO\_1 ff9853fmqsbdu cell physical IO interconnect bytes 8404992 12:17:11 BDTO\_1 ff9853fmqsbdu physical read total bytes 8404992 12:17:11 BDTO\_1 ff9853fmqsbdu logical read bytes from cache 23945216
```

Launch it on one instance and a particular stat :

```
./sqlidstat.pl inst=BDTO\_1 name='logical read bytes from cache' 12:27:16 INST\_NAME SQL\_ID NAME VALUE 12:27:16 BDTO\_1 ff9853fmqsbdu logical read bytes from cache 227860480
```

You could also use the well known [snapper](http://tech.e2sn.com/oracle-scripts-and-tools/session-snapper "snapper")&nbsp;script as&nbsp;Gwen Shapira who used it on a real practical case in this [post](http://www.pythian.com/news/37343/select-statement-generating-redo-and-other-mysteries-of-exadata/ "post").

I will follow that post with others posts and perl scripts&nbsp;I use to collect real time information based on wait events, latchs, sga stats...


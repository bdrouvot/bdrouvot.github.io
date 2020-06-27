---
layout: post
title: TAF is not working on Rac One Node in case of node failure ? Well it depends
date: 2013-05-16 16:38:40.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rac
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:139;}s:2:"wp";a:1:{i:0;i:32;}}
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/05/16/taf-is-not-working-on-rac-one-node-in-case-of-node-failure-well-it-depends/"
---
<p>Martin Bach did a good study of <strong>Rac One Node capability</strong> and especially of what happens with session failover during a database relocation or during a node failure into this <a href="http://martincarstenbach.wordpress.com/2011/02/16/rac-one-node-and-database-protection/" target="_blank">blog post</a>.</p>
<p><span style="text-decoration:underline;"><strong>He showed us that:</strong></span></p>
<ol>
<li>during a database relocation (initiated manually with <strong>srvctl relocate database</strong>) <strong>the TAF works as expected.</strong></li>
<li><strong>during a node failure the TAF did not work and you will be disconnected </strong>(for a running select or a new execution)</li>
</ol>
<p><strong>It's important:</strong> I said "<strong>during a node failure</strong>", it means no instances are available during the select.</p>
<p><strong>But what's about a session already connected to a TAF service before the node failure that :</strong></p>
<ul>
<li>will be <strong>inactive during</strong> the node failure</li>
<li>will <strong>launch a select after the node failure </strong>(means an instance is back)</li>
</ul>
<p><span style="text-decoration:underline;">For this one an <strong>node failure is</strong> <strong>transparent</strong>, let's see that in action:</span></p>
<p><span style="text-decoration:underline;">First check my database is a <strong>Rac One Node</strong>:</span></p>
<pre style="padding-left:30px;">srvctl config database -d BDTO | grep -i type
Type: RACOneNode</pre>
<p><span style="text-decoration:underline;">Now let's check that the <strong>bdto_taf</strong> service I'll connect on is a TAF one:</span></p>
<pre style="padding-left:30px;">srvctl config service -s BDTO_TAF -d BDTO | egrep -i "taf|failover"
Service name: BDTO_TAF
Failover type: SELECT
Failover method: BASIC
TAF failover retries: 0
TAF failover delay: 0
TAF policy specification: BASIC</pre>
<p>Well everything is in place to test.</p>
<p><span style="text-decoration:underline;">So connect to my Rac One Node database using the TAF service:</span></p>
<pre style="padding-left:30px;">sqlplus bdt/"test"@bdto_taf

SQL*Plus: Release 11.2.0.3.0 Production on Thu May 16 14:39:36 2013

Copyright (c) 1982, 2011, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.3.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Data Mining and Real Application Testing options

SQL&gt; alter session set nls_date_format='YYYY/MM/DD HH24:MI:SS';

Session altered.

SQL&gt; select sysdate||' '||host_name from v$instance;

SYSDATE||''||HOST_NAME
--------------------------------------------------------------------------------
2013/05/16 14:39:47 BDTSRV1</pre>
<p><span style="text-decoration:underline;">Now let's kill smon:</span></p>
<pre style="padding-left:30px;">ps -ef | grep -i smon | grep -i BDTO_1 
oracle    8035     1  0 14:30 ?        00:00:00 ora_smon_BDTO_1

kill -9 `ps -ef | grep -i smon | grep -i BDTO_1 | awk '{print $2}'`</pre>
<p><strong>Let's wait an instance is back on a node (Important step)</strong></p>
<p>In this case the instance has been relocated on the other node, let's see it is up:</p>
<pre style="padding-left:30px;">ps -ef | grep -i smon | grep -i BDTO_1
oracle   29266     1  0 14:54 ?        00:00:00 ora_smon_BDTO_1</pre>
<p><span style="text-decoration:underline;">and come back to our "idle" sqlplus session to re-launch the sql:</span></p>
<pre style="padding-left:30px;">SQL&gt; /

SYSDATE||''||HOST_NAME
--------------------------------------------------------------------------------
16-MAY-13 BDTSRV2

**Oh ! As you can see the "idle" sqlplus session has not been disconnected and we have been able to re-launch the query**.

As you can see we lost our **nls\_date\_format** setting. This is expected as explained into Martin's Book " **Pro Oracle Database 11g RAC on Linux**":

"

Also, note that while TAF can reauthorize a session, it does not restore the session to its previous  
state. Therefore, following an instance failure, you will need to restore any session variables, PL/SQL  
package variables, and instances of user-defined types. TAF will not restore these values automatically,  
so you will need to extend your applications to restore them manually.

"

**Conclusion:**

If you are using a TAF service to connect to a Rac One Node database (of course through the OCI), **then a node failure is transparent if the session is "idle" during the time there is no instance available**.

When you think of it, it is quite normal as the TAF is more or less nothing more than a OCI / SQL\*net feature, so once the TAF service is back, then TAF can work as expected.

**Remarks:**

- If the "idle" connection&nbsp;has an "opened" transaction, you'll receive the "ORA-25402: transaction must roll back" oracle error (This is not so transparent anymore :-) ).
- Do not test TAF connected as the oracle sys user as SYSDBA Sessions do not failover with SRVCTL TAF configured, see MOS [ID 1342992.1]
- If you are using srvctl to test TAF''s reaction on node failure then you should read the MOS note&nbsp;[ID 1324574.1] first (as the TAF may not work depending of the version and srvctl options being used)
- For a good explanation of Rac One Node, you should have a look to this&nbsp;Aman Sharma's [blog post](http://allthingsoracle.com/rac-one-node/).

**Why I wrote this post ?**

Because into Martin's [post](http://martincarstenbach.wordpress.com/2011/02/16/rac-one-node-and-database-protection/), I put a comment on&nbsp;"December 13, 2012 at 19:08" trying to explain this&nbsp;behavior. I came back to this comment yesterday and I realized that my comment was not so clear ! I hope this post is clear ;-)


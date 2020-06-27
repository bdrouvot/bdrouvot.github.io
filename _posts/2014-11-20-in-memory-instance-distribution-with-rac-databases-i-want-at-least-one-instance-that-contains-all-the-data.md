---
layout: post
title: 'In-Memory Instance distribution with RAC databases: I want at least one instance
  that contains all the data'
date: 2014-11-20 11:49:54.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- In-Memory
- Rac
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/Q3GYg4SvAAX
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/8HAkr867Bm
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5941150260284923904&type=U&a=G4A4
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _wpas_skip_tumblr: '1'
  _wpas_skip_path: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/"
---
<p>As you may know, with a RAC database, <strong>by default all objects populated into</strong><br />
<strong>In-memory will be distributed across all of the IM column stores in the cluster</strong>. It is also possible to have all the data appear in the IM column store on every node (Engineered Systems only). See this <a href="http://www.oracle.com/technetwork/database/in-memory/overview/twp-oracle-database-in-memory-2245633.html" target="_blank">white paper</a> for more details.</p>
<p><span style="text-decoration:underline;"><strong>There is 2 very interesting blog post around this subject:</strong></span></p>
<ol>
<li>Kerry Osborne show us <strong>how we can distribute the data </strong>(using the <em>distribute</em> INMEMORY attribute) across the nodes into this <a href="http://kerryosborne.oracle-guy.com/2014/09/12c-in-memory-on-rac/" target="_blank">blog post</a>.</li>
<li>Christian Antognini show us that having the data <strong>not fully populated </strong>on each instance <strong>could lead to bad performance</strong> into this <a href="http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/" target="_blank">blog post</a>.</li>
</ol>
<p><strong>But wait:</strong> If my RAC service is an <strong>active/passive</strong> one (I mean the service is started on only one instance) then I would like to have<strong> all the data fully populated into</strong> <strong>the In-Memory column store of the "active" instance  </strong>(and not distributed across the instances), right?</p>
<p><span style="text-decoration:underline;">Let's try to achieve this (all data fully populated into <strong>the In-Memory column store of the active instance</strong>):</span></p>
<p>So, by default, the IMCS distribution is the following:</p>
<pre style="padding-left:30px;">SQL&gt; alter table bdt inmemory ;

Table altered.

SQL&gt; select count(*) from bdt;

  COUNT(*)
----------
  38220000

SQL&gt; SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
FROM gv$im_user_segments
WHERE segment_name = 'BDT';

   INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- --------- ------------- ---------- -------------------
         1 COMPLETED     313720832 1610612736           732684288
         2 COMPLETED     274530304 1610612736           826236928
</pre>
<p>As you can see <strong>no instance contains all data</strong> as the BYTES_NOT_POPULATED column is  greater than 0.</p>
<p>Now let's set the hidden parameter <strong>"<em>_inmemory_auto_distribute</em>" to false</strong> and re-trigger the population on instance 1:</p>
<pre style="padding-left:30px;">SQL&gt; alter system set "_inmemory_auto_distribute"=false;

System altered.

SQL&gt; select count(*) from bdt;

  COUNT(*)
----------
  38220000

SQL&gt; SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
FROM gv$im_user_segments
WHERE segment_name = 'BDT';

   INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- --------- ------------- ---------- -------------------
         1 COMPLETED     587137024 1610612736                   0
         2 COMPLETED     274530304 1610612736           826236928
</pre>
<p><strong>BINGO!</strong> Look at the Instance 1, it now contains all the data in its IMCS (as BYTES_NOT_POPULATED=0).</p>
<p>If I connect to the Instance 2 and launch the query to trigger the population I'll get:</p>
<pre style="padding-left:30px;">SQL&gt; SELECT inst_id, populate_status, inmemory_size, bytes, bytes_not_populated
FROM gv$im_user_segments
WHERE segment_name = 'BDT';
  2    3  
   INST_ID POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- --------- ------------- ---------- -------------------
1 COMPLETED 587137024 1610612736 0 2 COMPLETED 589234176 1610612736 0

So, **both instances now contain all the data (like the duplicate attribute would have done).**

**Remarks:**

1. In case of active/passive service (means preferred/available service) then after a service failover the second instance will also contain all the data (once populated).
2. In case of active/active service then each database instance will contain all the data but not necessary at the same time (so the behaviour is a little bit different of the _duplicate_ attribute).
3. I used an hidden parameter, so you should get oracle support approval to use it.
4. I'll check the behaviour with PDB into [another post](http://bdrouvot.wordpress.com/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/ "In-Memory Instance distribution with RAC and Multitenant Environment").

**Conclusion:**

- Thanks to the hidden parameter "_\_inmemory\_auto\_distribute_" we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.
- By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
- With " **_\_inmemory\_auto\_distribute_**" set to **false** , then we are able to populate one instance (or all the instances) **with all the data**.

**Update 2015/03/27:**

- This “workaround” proposed to get rid of the default behaviour on non Engineered Systems (“Every row of the test table is stored in the IMCS of either one instance or the other”) **works for any type of service.**
- In case of preferred/available&nbsp;service, there is no need to set the hidden parameter to get the IM fully populated on one node. The secret sauce is linked to the **_parallel\_instance\_group_** parameter. See [this blog post](https://bdrouvot.wordpress.com/2015/03/26/in-memory-instance-distribution-with-rac-databases-impact-of-the-parallel_instance_group-parameter/) for more details.

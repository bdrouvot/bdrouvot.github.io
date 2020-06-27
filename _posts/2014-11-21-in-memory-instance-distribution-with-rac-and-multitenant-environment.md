---
layout: post
title: In-Memory Instance distribution with RAC and Multitenant Environment
date: 2014-11-21 13:46:40.000000000 +01:00
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
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/hr91Dnqg2hw
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/9SbFyYQG4Q
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5941542018600701952&type=U&a=IZJo
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/"
---
<p>In a previous post (see <a title="In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data" href="http://bdrouvot.wordpress.com/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/" target="_blank">In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data</a>) I provided a way to get rid of the default In-Memory Instance distribution on a RAC (non CDB) database.</p>
<p><strong><span style="text-decoration:underline;">To summarize:</span> </strong></p>
<ul>
<li>Thanks to the hidden parameter “<em>_inmemory_auto_distribute</em>” we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.</li>
<li>By <strong>default</strong> all objects populated into memory will be <strong>distributed</strong> across all of the IM column stores in the cluster.</li>
<li>With “<strong><em>_inmemory_auto_distribute</em></strong>” set to <strong>false</strong>, then we are able to populate one instance (or all the instances) <strong>with all the data</strong>.</li>
</ul>
<p>Now, let's have a look at the In-Memory distribution on a <strong>Multitenant RAC database</strong> and let's check <strong>if we can influence (get rid of the default behaviour)</strong> how the In-Memory is distributed across all the Instances per PDB.</p>
<p><span style="text-decoration:underline;"><strong>By default the IMCS distribution for a PDB is the following:</strong></span></p>
<p><span style="text-decoration:underline;"><strong>First case:</strong></span> The PDB is open <strong>on one</strong> Instance only:</p>
<pre style="padding-left:30px;">SYS@CDB$ROOT&gt; alter pluggable database BDTPDB open instances=('BDT12c3_1');

Pluggable database altered.

-- Now connect to the BDTPDB PDB as bdt/bdt
SYS@CDB$ROOT&gt; @connect_user_pdb.sql
Enter value for user: bdt
Enter value for pwd: bdt
Enter value for pdb: BDTPDB
Connected.

-- Let's enable the inmemory attribute on the BDT table
BDT@BDTPDB&gt; alter table bdt inmemory ;

Table altered.

-- Trigger the IMCS population
BDT@BDTPDB&gt; select count(*) from bdt;

  COUNT(*)
----------
  44591000

-- check the distribution
BDT@BDTPDB&gt; SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
FROM gv$im_user_segments s, v$pdbs c
WHERE s.segment_name = 'BDT'
and c.con_id=s.con_id; 

   INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- ------------------------------ --------- ------------- ---------- -------------------
         1 BDTPDB                         COMPLETED     695074816 1879048192                   0

</pre>
<p>As you can see <strong>the Instance 1 contains all the data</strong> for this PDB (as BYTES_NOT_POPULATED=0)</p>
<p><span style="text-decoration:underline;"><strong>Second case:</strong></span> The PDB is open <strong>on all</strong> the Instances:</p>
<pre style="padding-left:30px;">-- Open the BDTPDB PDB on all instances
SYS@CDB$ROOT&gt; alter pluggable database BDTPDB open instances=all;

Pluggable database altered.

-- Now connect to the BDTPDB PDB as bdt/bdt
SYS@CDB$ROOT&gt; @connect_user_pdb.sql
Enter value for user: bdt
Enter value for pwd: bdt
Enter value for pdb: BDTPDB
Connected.  

-- Let's enable the inmemory attribute on the BDT table
BDT@BDTPDB&gt; alter table bdt inmemory ;

Table altered.

-- Trigger the IMCS population
BDT@BDTPDB&gt; select count(*) from bdt;

  COUNT(*)  
----------  
  44591000

-- check the distribution
BDT@BDTPDB&gt; SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
FROM gv$im_user_segments s, v$pdbs c
WHERE s.segment_name = 'BDT'
and c.con_id=s.con_id;


   INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- ------------------------------ --------- ------------- ---------- -------------------
         1 BDTPDB                         COMPLETED     376307712 1879048192           802889728
         2 BDTPDB                         COMPLETED     298909696 1879048192          1006559232
</pre>
<p>We can see that (for this PDB), by default, <strong>no instance contains all data</strong> as the BYTES_NOT_POPULATED column is  greater than 0.</p>
<p>Now let’s set the hidden parameter <strong>“<em>_inmemory_auto_distribute</em>” to false</strong> <strong>(at the PDB level)</strong>:</p>
<pre style="padding-left:30px;">BDT@BDTPDB&gt; alter system set "_inmemory_auto_distribute"=false;
alter system set "_inmemory_auto_distribute"=false
*
ERROR at line 1:
ORA-65040: operation not allowed from within a pluggable database
</pre>
<p>So, we<strong> can't set this hidden</strong> parameter at the PDB level.</p>
<p>Ok, let's set it at the <strong>CDB level</strong>:</p>
<pre style="padding-left:30px;">SYS@CDB$ROOT&gt; alter system set "_inmemory_auto_distribute"=false;

System altered.</pre>
<p>then re-trigger the population on instance 1 (for this PDB) and check the distribution:</p>
<pre style="padding-left:30px;">SYS@CDB$ROOT&gt; @connect_user_pdb.sql
Enter value for user: bdt
Enter value for pwd: bdt
Enter value for pdb: BDTPDB
Connected.  
BDT@BDTPDB&gt; select count(*) from bdt;

  COUNT(*)  
----------  
  44591000

BDT@BDTPDB&gt; SELECT s.inst_id, c.name,s.populate_status, s.inmemory_size, s.bytes, s.bytes_not_populated
FROM gv$im_user_segments s, v$pdbs c
WHERE s.segment_name = 'BDT'
and c.con_id=s.con_id;

   INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- ------------------------------ --------- ------------- ---------- -------------------
         2 BDTPDB                         COMPLETED     297861120 1879048192          1006559232
         1 BDTPDB                         COMPLETED     675151872 1879048192                   0
</pre>
<p><strong>BINGO!</strong> Look at the Instance 1, <strong>it now contains all the data</strong> in its IMCS (as BYTES_NOT_POPULATED=0) for this PDB.</p>
<p>If I connect on the Instance 2 (for this PDB) and launch the query to trigger the population, I’ll get:</p>
<pre style="padding-left:30px;">   INST_ID NAME                           POPULATE_ INMEMORY_SIZE      BYTES BYTES_NOT_POPULATED
---------- ------------------------------ --------- ------------- ---------- -------------------
1 BDTPDB COMPLETED 675151872 1879048192 0 2 BDTPDB COMPLETED 672006144 1879048192 0

So, **both instances now contain all the data for this PDB.**

**Remarks:**

- As I said into the previous post: I used an hidden parameter, so you should get oracle support approval to use it.
- I guess (because I can't test) that in a Multitenant and Engineered Systems context it is also possible to have all the data appear in the IM column store on every node for a PDB (thanks to the _duplicate_ attribute): As Christian Antognini show us into [this blog post](http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/)for non CDB.

**Conclusion:**

The conclusion is the same than in [my previous post](http://bdrouvot.wordpress.com/2014/11/20/in-memory-instance-distribution-with-rac-databases-i-want-at-least-one-instance-that-contains-all-the-data/ "In-Memory Instance distribution with RAC databases: I want at least one instance that contains all the data") (with non CDB):

- Thanks to the hidden parameter “_\_inmemory\_auto\_distribute_” we are able to change the way the In-Memory Column Store are populated by default on an oracle RAC database.
- By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
- With “ **_\_inmemory\_auto\_distribute_** ” set to **false** , then we are able to populate one instance (or all the instances) **with all the data**.

And we can add that:

- If the PDB is open on one instance only then this instance contains all the data.
- If the PDB is open on all the instances then we have to set “ **_\_inmemory\_auto\_distribute_** ” to **false to have at least one instance that contains all the data.**
- Each PDB population **will behave the same** (distributed or not) and inherit from the CDB setting (As we **can not set the hidden** parameter at the **PDB** level).

&nbsp;


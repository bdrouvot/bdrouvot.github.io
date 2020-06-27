---
layout: post
title: 'SystemTap and Oracle RDBMS: I/O and Network Traffic'
date: 2017-09-30 16:55:36.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SystemTap
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6319922498569932800&type=U&a=QlBQ
  _publicize_job_id: '9830546747'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/914156810533253121";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2017/09/30/systemtap-and-oracle-rdbms-io-and-network-traffic/"
---
<p>Now that I am able to <a href="https://bdrouvot.wordpress.com/2017/06/05/systemtap-aggregate-by-database/" target="_blank" rel="noopener">aggregate SytemTap probes by Oracle database</a>, let’s focus on I/O and Network Traffic.</p>
<p>For this purpose a new SystemTap script (traffic_per_db.stp) has been created and has been added into this <a href="https://github.com/bdrouvot/SystemTap" target="_blank" rel="noopener">github repository</a>.</p>
<h2>traffic_per_db</h2>
<p>This script tracks the I/O and Network traffic per database and also groups the non database(s) traffic.</p>
<h3>Usage:</h3>
<pre>$&gt; stap -g ./traffic_per_db.stp &lt;oracle uid&gt; &lt;refresh time ms&gt; &lt;io|network|both&gt;</pre>
<h3>Output example (I/O only):</h3>
<pre style="padding-left:30px;">$&gt; stap -g ./traffic_per_db.stp 54321 5000 io

-------------------------------------------------------------------------------------------------------------
|                                                          I/O                                              |
-------------------------------------------------------------------------------------------------------------
                                  READS                     |                     WRITES                    |
                                                            |                                               |
                       VFS                    BLOCK         |          VFS                    BLOCK         |
            | NB                 KB | NB                 KB | NB                 KB | NB                 KB |
            | --                 -- | --                 -- | --                 -- | --                 -- |
BDTS        | 6203            49280 | 0                   0 | 11                 64 | 12                128 |
NOT_A_DB    | 45                  3 | 0                   0 | 15                  2 | 0                   0 |
-------------------------------------------------------------------------------------------------------------</pre>
<p>For this example the database files are located on a file system. In this output, we can see than the reads I/O are served by the file system cache: Reads VFS I/O and no Reads BLOCK I/O.</p>
<h3>Output example (Network only):</h3>
<h4>Example 1: The database files are located on Kernel NFS (kNFS)</h4>
<pre style="padding-left:30px;">$&gt; stap -g ./traffic_per_db.stp 54321 5000 network
-------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                Network                                                                    |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
                                              RECV                                  |                                 SENT                                  |
                                                                                    |                                                                       |
                       TCP                     UDP                     NFS          |          TCP                     UDP                     NFS          |
            | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB |
            | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- |
NOT_A_DB    | 4                  32 | 0                   0 | 61                706 | 1943              252 | 0                   0 | 5                  80 |
BVDB        | 0                   0 | 0                   0 | 1623            16825 | 113                13 | 0                   0 | 170              1437 |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
</pre>
<p>As expected we can observed NFS traffic at the database level.</p>
<h4>Example 2: The database files are located on Direct NFS (dNFS)</h4>
<pre style="padding-left:30px;">$&gt; stap -g ./traffic_per_db.stp 54321 5000 network

-------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                Network                                                                    |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
                                              RECV                                  |                                 SENT                                  |
                                                                                    |                                                                       |
                       TCP                     UDP                     NFS          |          TCP                     UDP                     NFS          |
            | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB |
            | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- |
BVDB        | 3810            18934 | 0                   0 | 0                   0 | 2059             1787 | 0                   0 | 0                   0 |
NOT_A_DB    | 3                  24 | 0                   0 | 0                   0 | 4                   2 | 0                   0 | 0                   0 |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
As you can see the NFS traffic has been replaced by a TCP traffic at the database level.

## Remarks

- In this post the word database stands for “all the foreground and background processes linked to an oracle database”.
- In a consolidated environment, having a view per database can be very useful.
- This script helps to display the traffic per database and also reports the one that is not related to databases ("NOT\_A\_DB").
- The probes documentation can be found [here](https://sourceware.org/systemtap/tapsets/).

## Conclusion

We are able to track the I/O and the Network traffic at the database level.


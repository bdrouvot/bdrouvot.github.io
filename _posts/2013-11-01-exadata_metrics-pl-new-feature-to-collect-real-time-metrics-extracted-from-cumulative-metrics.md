---
layout: post
title: 'exadata_metrics.pl: New feature to collect real-time metrics extracted from
  cumulative metrics'
date: 2013-11-01 15:39:02.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/0NVzZbwYmV
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5802050953410539520&type=U&a=Dapv
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/11/01/exadata_metrics-pl-new-feature-to-collect-real-time-metrics-extracted-from-cumulative-metrics/"
---
<p>I am a big fan of real-time metrics extraction based on the cumulative metrics: That is to say take a snapshot each second (default interval) from the cumulative metrics and <strong>computes the delta</strong> with the previous snapshot.</p>
<p>I build such tools for <a title="ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time" href="http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/" target="_blank">ASM metrics</a> and for <a title="Exadata real-time metrics extracted from cumulative metrics:  Part II" href="http://bdrouvot.wordpress.com/2013/03/05/exadata-real-time-metrics-extracted-from-cumulative-metrics-part-ii/" target="_blank">Exadata Cell metrics</a>.</p>
<p>Recently, I added a new feature to the <a title="ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time" href="http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/" target="_blank">asm_metrics.pl</a> script: The ability to display the average <strong>delta values since the collection began</strong> (Means since we launched the script).</p>
<p>Well, I just added this new feature to the exadata_metrics.pl script.</p>
<p><span style="text-decoration:underline;">Of course the old features remain:</span></p>
<ul>
<li>You can choose the number of snapshots to display and the time to wait between the snapshots.</li>
<li>You can choose to filter on name and objectname based on predicates (see the help).</li>
<li>You can work on all the cells or a subset thanks to the CELL or the GROUPFILE parameter.</li>
<li>You can decide the way to compute the metrics with no aggregation, aggregation on cell, objectname or both.</li>
</ul>
<p><span style="text-decoration:underline;">Let's see the help:</span></p>
<pre style="padding-left:30px;"> ./exadata_metrics.pl help

Usage: ./exadata_metrics.pl [Interval [Count]] [cell=|groupfile=] [display=] [show=] [top=] [name=] [name!=] [objectname=] [objectname!=]

 Default Interval : 1 second.
 Default Count : Unlimited

 Parameter                 Comment                                                      Default
 ---------                 -------                                                      -------
 CELL=                     comma-separated list of cells
 GROUPFILE=                file containing list of cells
 SHOW=                     What to show (name included): cell,objectname                ALL
 DISPLAY=                  What to display: snap,avg (comma separated list)             SNAP
 TOP=                      Number of rows to display                                    10
 NAME=                     ALL - Show all cumulative metrics (wildcard allowed)         ALL
 NAME!=                    Exclude cumulative metrics (wildcard allowed)                EMPTY
 OBJECTNAME=               ALL - Show all objects (wildcard allowed)                    ALL
 OBJECTNAME!=              Exclude objects (wildcard allowed)                           EMPTY

utility assumes passwordless SSH from this cell node to the other cell nodes
utility assumes ORACLE_HOME has been set (with celladmin user for example)

Example : ./exadata_metrics.pl cell=cell
Example : ./exadata_metrics.pl groupfile=./cell_group
Example : ./exadata_metrics.pl groupfile=./cell_group show=name
Example : ./exadata_metrics.pl cell=cell objectname='CD_disk03_cell' name!='.*RQ_W.*'
Example : ./exadata_metrics.pl cell=cell name='.*BY.*' objectname='.*disk.*' name!='GD.*' objectname!='.*disk1.*'</pre>
<p><strong>The “display” option is the new one:</strong> It allows you to display the delta snap values (as previously), the average delta values since the collection began (that is to say since the script has been launched) or both.</p>
<p><span style="text-decoration:underline;">Let's see one example:<br />
</span></p>
<p>I want to extract real-time metrics from the %IO_TM_R% cumulative ones, for 2 cells and aggregate the results for all the cells and all the objectname (means I just want to show the metrics). I want to see each snapshots <strong>and the average since</strong> we launched the script.</p>
<p>So, I launch the script that way:</p>
<pre style="padding-left:30px;">./exadata_metrics.pl cell=exacell1,exacell2 display=avg,snap name='.*IO_TM_R.*' show=name</pre>
<p>It will produce this kind of output:</p>
<pre style="padding-left:30px;">--------------------------------------
----------COLLECTING DATA-------------
--------------------------------------

......... SNAP FOR LAST COLLECTION TIME ...................

DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
--------   ----                    ----                         ----------                                                  -----
60                                 CD_IO_TM_R_LG                                                                            0.00 us
60                                 CD_IO_TM_R_SM                                                                            807269.00 us

......... AVG SINCE FIRST COLLECTION TIME...................

DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
--------   ----                    ----                         ----------                                                  -----
60                                 CD_IO_TM_R_LG                                                                            0.00 us
60                                 CD_IO_TM_R_SM                                                                            807269.00 us</pre>
<p>So, no differences between the delta snap and the average after the first snap (which is obvious :-) ).</p>
<p>Into the "SNAP" section, the delta(s) field <strong>computes the delta in seconds of the collectionTime recorded into the snapshots</strong> (see this <a title="Exadata Cell metrics: collectionTime attribute, something that matters" href="http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/" target="_blank">post </a>for more details). Into the "AVG" section, the delta(s) field is the sum of the delta(s) field from all the previous "SNAP" sections.</p>
<p>Let's have a look to the output after 2 minutes of metrics extraction to make things clear:</p>
<pre style="padding-left:30px;">--------------------------------------
----------COLLECTING DATA-------------
--------------------------------------

......... SNAP FOR LAST COLLECTION TIME ...................

DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
--------   ----                    ----                         ----------                                                  -----
61                                 CD_IO_TM_R_LG                                                                            0.00 us
61                                 CD_IO_TM_R_SM                                                                            348479.00 us

......... AVG SINCE FIRST COLLECTION TIME...................

DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
--------   ----                    ----                         ----------                                                  -----
121 CD\_IO\_TM\_R\_LG 0.00 us 121 CD\_IO\_TM\_R\_SM 577874.00 us

So, during the last 61 seconds of collection the CD\_IO\_TM\_R\_SM metric delta value is 348479.00 us.

It leads to an average of 577874.00 us since the collection began (That is to say since 121 seconds of metrics collection).

**Remarks** :

- The exadata\_metrics.pl script can be downloaded from [this repository.](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)
- The DELTA (s) field is an important one to interpret the result correctly, I strongly recommend to read this [post](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters")for a better understanding of it.

**Conclusion:**

The exadata\_metrics.pl now offers the ability to display the average delta values since the collection began (that is to say since the script has been launched).


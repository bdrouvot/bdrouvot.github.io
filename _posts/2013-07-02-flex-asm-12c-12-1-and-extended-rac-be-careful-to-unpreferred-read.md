---
layout: post
title: 'Flex ASM 12c (12.1) and Extended Rac: be careful to "unpreferred" read !'
date: 2013-07-02 09:40:24.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
tags: []
meta:
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:192;}s:2:"wp";a:1:{i:0;i:33;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  twitter_cards_summary_img_size: a:6:{i:0;i:844;i:1;i:355;i:2;i:3;i:3;s:24:"width="844"
    height="355"";s:4:"bits";i:8;s:4:"mime";s:9:"image/png";}
  geo_public: '0'
  _wpas_skip_7950430: '1'
  _wpas_skip_5547632: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/gtJKcwKJcTJ
  _wpas_skip_8482624: '1'
  _wpas_done_8482624: '1'
  _publicize_job_id: '5242981235'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/"
---
<p><strong>Update 2015/03/06: </strong>The following has been recognized as "unpublished BUG 17045279 - ASM_PREFERRED_READ DOES NOT WORK WITH FLEX ASM", which is planned to be fixed in the next upcoming release.</p>
<p><strong>Update 2017/05/20: </strong>As of 12.2, preferred reads are site-aware (extract of  <a href="https://twitter.com/OracleRACpm" target="_blank" rel="noopener noreferrer">Markus Michalewicz</a> presentation available <a href="https://www.slideshare.net/MarkusMichalewicz/oracle-extended-clusters-for-oracle-rac" target="_blank" rel="noopener noreferrer">here</a>) so that the issue described into this blog post has been addressed.</p>
<div class="page" title="Page 31">
<div class="section">
<div class="layoutArea">
<div class="column">
<p><a href="https://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/pref_site_aware/" rel="attachment wp-att-3127"><img class="aligncenter size-full wp-image-3127" src="{{ site.baseurl }}/assets/images/pref_site_aware.png" alt="" width="640" height="359" /></a></p>
<p>&nbsp;</p>
</div>
</div>
</div>
</div>
<h2><strong>Introduction</strong></h2>
<p>As you know Oracle 11g introduced a new feature called "Asm Preferred Read". It is very useful in extended RAC as it allows each node to define a preferred failure group, allowing nodes to access local failure groups in preference to remote ones. This is done thanks to the "<em>asm_preferred_read_failure_groups</em>" parameter.</p>
<p><span style="text-decoration:underline;"><strong>Fine, but remember:</strong></span></p>
<ol>
<li>This parameter has to be set at the <strong>ASM instance level</strong> (not the database instance one).</li>
<li>This is the <strong>database instance</strong> (or its shadow processes) that is doing the IOs (not the ASM instance).</li>
</ol>
<p><span style="text-decoration:underline;"><strong>Why is it important ? </strong></span>  Because with <a href="http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf" target="_blank" rel="noopener noreferrer">Flex ASM</a> in place, database instances are connection load balanced across the set of available ASM instances (that of course are not necessary "local" to the database instance anymore):</p>
<p><a href="http://bdrouvot.files.wordpress.com/2013/07/flex_asm1.png"><img class="aligncenter size-full wp-image-1151" src="{{ site.baseurl }}/assets/images/flex_asm1.png" alt="flex_asm1" width="620" height="260" /></a></p>
<p>And then you could hit what I call the "<strong>unpreferred</strong>" read behavior.</p>
<p><span style="text-decoration:underline;">Let me explain more in depth with an example:</span></p>
<p>Suppose that you have an extended 3 nodes RAC:</p>
<ul>
<li>racnode1 located in SITE A</li>
<li>racnode2 located in SITE A</li>
<li>racnode3 located in SITE B</li>
</ul>
<p>And 2 ASM instances actives:</p>
<ul>
<li>+ASM1 located in SITE A</li>
<li>+ASM3 located in SITE B</li>
</ul>
<pre style="padding-left:30px;">srvctl status asm
ASM is running on racnode3,racnode1</pre>
<p>So you created 2 failgroup SITEA and SITEB and you set the <em>asm_preferred_read_failure_groups</em> parameter that way for the DATAP diskgroup:</p>
<pre style="padding-left:30px;">SQL&gt; alter system set asm_preferred_read_failure_groups='DATAP.SITEB' sid='+ASM3';
System altered.

SQL&gt; alter system set asm_preferred_read_failure_groups='DATAP.SITEA' sid='+ASM1';
System altered.</pre>
<p>So that ASM3 prefers to read from SITEB and ASM1 from SITEA (which fully makes sense from the ASM point of view).</p>
<p><strong><span style="text-decoration:underline;">But what if  a database instance located into SITEB (racnode3) is using ASM1 located in SITEA ?</span></strong></p>
<pre style="padding-left:30px;">SQL&gt;  select I.INSTANCE_NAME,C.INSTANCE_NAME,C.DB_NAME
  2  from gv$instance I, gv$asm_client C 
  3  where C.INST_ID=I.INST_ID and C.instance_name='NOPBDT3';

INSTANCE_NAME    INSTANCE_NAME                                                    DB_NAME
---------------- ---------------------------------------------------------------- --------
+ASM1            NOPBDT3                                                          NOPBDT</pre>
<p>As you can see the NOPBDT3 database instance is using the +ASM1 instance, while the NOPBDT3 database instance is located on racnode3:</p>
<pre style="padding-left:30px;">srvctl status instance -i NOPBDT3 -d NOPBDT
Instance NOPBDT3 is running on node racnode3</pre>
<p><strong>Which means NOPBDT3 located into SITEB will prefer to request read IO from SITEA which is of course very bad.</strong></p>
<p><span style="text-decoration:underline;">Let's check this with my <a title="ASM I/O Statistics Utility" href="http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/" target="_blank" rel="noopener noreferrer">asmiostat</a> utility and Kevin Closson's <a href="http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/" target="_blank" rel="noopener noreferrer">SLOB2 kit</a>:</span></p>
<p>Let's launch SLOB locally on NOPBDT3 only:</p>
<pre style="padding-left:30px;">[oracle@racnode3 SLOB]$ ./runit.sh 3
NOTIFY: 
UPDATE_PCT == 0
RUN_TIME == 300
WORK_LOOP == 0
SCALE == 10000
WORK_UNIT == 256
ADMIN_SQLNET_SERVICE == ""
ADMIN_CONNECT_STRING == "/ as sysdba"
NON_ADMIN_CONNECT_STRING == ""
SQLNET_SERVICE_MAX == "0"</pre>
<p>And check the IO metrics with my asmiostat utility that way (I want to see Instance, Diskgroup and Failgroup):</p>
<pre style="padding-left:30px;">./real_time.pl -type=asmiostat -show=inst,dg,fg -dg=DATAP</pre>
<p>With the following output:</p>
<p><a href="http://bdrouvot.files.wordpress.com/2013/07/flex_asm_pref_read_12c.png"><img class="aligncenter size-full wp-image-1161" src="{{ site.baseurl }}/assets/images/flex_asm_pref_read_12c.png" alt="flex_asm_pref_read_12c" width="620" height="236" /></a></p>
<p>As I am the only one to work on this <a title="Build your own Flex ASM 12c lab using Virtual Box" href="http://bdrouvot.wordpress.com/2013/06/29/build-your-own-flex-asm-12c-lab-using-virtual-box/" target="_blank" rel="noopener noreferrer">Lab</a>, you can see with no doubt that the IO metrics coming from the Instance NOPBDT3 are recorded into the ASM instance +ASM1 and clearly indicates that the read IOs have been done on SITEA.</p>
<p><span style="text-decoration:underline;"><strong>How can we "fix" this ?</strong></span></p>
<p>You can temporary fix this that way (connected on the +ASM1 instance):</p>
<pre style="padding-left:30px;">SQL&gt; ALTER SYSTEM RELOCATE CLIENT 'NOPBDT3:NOPBDT';
System altered.

SQL&gt; select I.INSTANCE_NAME,C.INSTANCE_NAME,C.DB_NAME
  2  from gv$instance I, gv$asm_client C 
  3   where C.INST_ID=I.INST_ID and C.instance_name='NOPBDT3';

INSTANCE_NAME    INSTANCE_NAME                                                    DB_NAME
---------------- ---------------------------------------------------------------- --------
+ASM3 NOPBDT3 NOPBDT +ASM3 NOPBDT3 NOPBDT

That way the NPPBDT3 database instance will use the +ASM3 instance and then will launch its read IO on SITEB .

But I had to bounce the NOPBDT3 database instance so that it launchs the read IO from SITEB (If not it was still using SITEA, well maybe a subject for another post)

**Conclusion:**

[Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) is a great feature but you have to be careful if you want to use it in an extended Rac with preferred read in place. &nbsp;If not you may hit the "unpreferred" read behavior.

**UPDATES:**

- Have a look to this [new post](http://bdrouvot.wordpress.com/2013/07/05/asm-io-statistics-utility-v2/ "ASM I/O Statistics Utility V2") describing my ASM I/O Statistics V2 to see the "unpreferred" read more in depth (at the database instance level).
- The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.
- Unpreferred read still exists in 12.1.0.2: ASM1/NBDT1 located on SITE1, ASM2/NBDT2 located on SITE2, ASM1 prefers to read from SITE1 and serves NBDT2:

[![unpref_12102]({{ site.baseurl }}/assets/images/unpref_12102.png)](https://bdrouvot.files.wordpress.com/2013/07/unpref_12102.png)


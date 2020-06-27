---
layout: post
title: 'SystemTap and Oracle RDBMS: Aggregate by database'
date: 2017-06-05 20:27:33.000000000 +02:00
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
  _publicize_done_2435436: '1'
  _publicize_job_id: '5807219021'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6277576453739675648&type=U&a=7DSF
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/871810765229043712";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
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
permalink: "/2017/06/05/systemtap-aggregate-by-database/"
---
<h2>Introduction</h2>
<p>The purpose of this post is to share a way to aggregate by Oracle database within SystemTap probes. Let's describe a simple use case to make things clear.</p>
<h2>Use Case</h2>
<p>Let's say that I want to get the number and the total size of TCP messages that have been sent and received by an Oracle database. To do so, let's use 2 probes:</p>
<ul>
<li><a href="https://sourceware.org/systemtap/tapsets/API-tcp-recvmsg.html" target="_blank" rel="noopener">probe::tcp.recvmsg</a></li>
<li><a href="https://sourceware.org/systemtap/tapsets/API-tcp-sendmsg.html" target="_blank" rel="noopener">probe::tcp.sendmsg</a></li>
</ul>
<p>and fetch the command line of the processes that trigger the event thanks to the <a href="https://sourceware.org/systemtap//tapsets/API-cmdline-str.html" target="_blank" rel="noopener">cmdline_str()</a> function. In case of a process related to an oracle database, the <em>cmdline_str()</em> output would look like one of those 2:</p>
<ul>
<li>ora_&lt;<em>something</em>&gt;_&lt;<em>instance_name</em>&gt;</li>
<li>oracle&lt;<em>instance_name</em>&gt; (LOCAL=&lt;something&gt;</li>
</ul>
<p>So let's write two <a href="https://sourceware.org/systemtap/langref/Components_SystemTap_script.html#SECTION00046000000000000000" target="_blank" rel="noopener">embedded C functions</a> to extract the Instance name from each of the 2 strings described above.</p>
<h2>Functions</h2>
<h3>get_oracle_name_b:</h3>
<p>For example, this function would extract BDT from "ora_dbw0_BDT" or any "ora_&lt;<em>something</em>&gt;_BDT" string.</p>
<p>The code is the following:</p>
<p>[code language="perl"]<br />
function get_oracle_name_b:string (mystr:string) %{<br />
char *ptr;<br />
int  ch = '_';<br />
char *strargs = STAP_ARG_mystr;<br />
ptr = strchr( strchr( strargs , ch) + 1 , ch);<br />
snprintf(STAP_RETVALUE, MAXSTRINGLEN, &quot;%s&quot;,ptr + 1);<br />
%}<br />
[/code]</p>
<h3>get_oracle_name_f:</h3>
<p>For example, this function would extract BDT from "oracleBDT (LOCAL=NO)", "oracleBDT (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))" or any "oracleBDT (LOCAL=&lt;something&gt;" string.</p>
<p>The code is the following:</p>
<p>[code language="perl"]<br />
function get_oracle_name_f:string (mystr:string) %{<br />
char *ptr;<br />
int    ch = ' ';<br />
char substr_res[30];<br />
char *strargs = STAP_ARG_mystr;<br />
ptr = strchr( strargs, ch );<br />
strncpy (substr_res,strargs+6, ptr - strargs - 6);<br />
substr_res[ptr - strargs - 6]='&#092;&#048;';<br />
snprintf(STAP_RETVALUE, MAXSTRINGLEN, &quot;%s&quot;,substr_res);<br />
%}<br />
[/code]</p>
<p>Having in mind that the SystemTap aggregation operator is "&lt;&lt;&lt;" (<a href="https://sourceware.org/systemtap/langref/Statistics_aggregates.html" target="_blank" rel="noopener">as explained here</a>) we can use those 2 functions to aggregate within the probes by Instance Name (passing as parameter the <em>cmdline_str())</em> that way:</p>
<p>[code language="perl"]<br />
probe tcp.recvmsg<br />
{</p>
<p>if ( isinstr(cmdline_str() , &quot;ora_&quot; ) == 1 &amp;&amp; uid() == orauid) {<br />
tcprecv[get_oracle_name_b(cmdline_str())] &lt;&lt;&lt; size<br />
} else if ( isinstr(cmdline_str() , &quot;LOCAL=&quot; ) == 1 &amp;&amp; uid() == orauid) {<br />
tcprecv[get_oracle_name_f(cmdline_str())] &lt;&lt;&lt; size<br />
} else {<br />
tcprecv[&quot;NOT_A_DB&quot;] &lt;&lt;&lt; size<br />
}<br />
}</p>
<p>probe tcp.sendmsg<br />
{</p>
<p>if ( isinstr(cmdline_str() , &quot;ora_&quot; ) == 1 &amp;&amp; uid() == orauid) {<br />
tcpsend[get_oracle_name_b(cmdline_str())] &lt;&lt;&lt; size<br />
} else if ( isinstr(cmdline_str() , &quot;LOCAL=&quot; ) == 1 &amp;&amp; uid() == orauid) {<br />
tcpsend[get_oracle_name_f(cmdline_str())] &lt;&lt;&lt; size<br />
} else {<br />
tcpsend[&quot;NOT_A_DB&quot;] &lt;&lt;&lt; size<br />
}<br />
}<br />
[/code]</p>
<p>As you can see, non oracle database would be recorded and displayed as "NOT_A_DB".</p>
<p>Based on this, the <em>tcp_by_db.stp</em> script has been created.</p>
<h2>tcp_by_db.stp: script usage and output example</h2>
<h3>Usage:</h3>
<pre>$&gt; stap -g ./tcp_by_db.stp &lt;oracle uid&gt; &lt;refresh time ms&gt;</pre>
<h3>Output Example:</h3>
<pre>$&gt; stap -g ./tcp_by_db.stp 54321 10000
---------------------------------------------------------
NAME              NB_SENT   SENT_KB   NB_RECV   RECV_KB
---------------------------------------------------------
VBDT              5439      8231      10939     64154
NOT_A_DB          19        4         41        128
BDT               19        50        35        259

---------------------------------------------------------
NAME              NB_SENT   SENT_KB   NB_RECV   RECV_KB
---------------------------------------------------------
VBDT              267       109       391       2854
NOT_A_DB          102       19        116       680
BDT               26        55        47        326

---------------------------------------------------------
NAME              NB_SENT   SENT_KB   NB_RECV   RECV_KB
---------------------------------------------------------
VBDT 340 176 510 2940 NOT\_A\_DB 150 8 151 1165 BDT 42 77 61 423

Remarks:

- The oracle uid on my machine is 54321
- The refresh time has been set to 10 seconds
- You can see the aggregation for 2 databases on my machine and also for all that is not an oracle database

## Whole Script source code

The whole code is available at:

- [This github repository](https://github.com/bdrouvot/SystemTap)
- [This page](https://bdrouvot.wordpress.com/tcp_by_dbs-stp_script/)

## Conclusion

Thanks to the embedded C functions we have been able to aggregate by Oracle database within SystemTap probes.


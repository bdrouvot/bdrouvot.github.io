---
layout: post
title: SystemTap for PostgreSQL Toolkit
date: 2017-11-01 18:25:42.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Postgresql
- SystemTap
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/925775891619336193";}}
  _publicize_job_id: '10975415280'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6331541580134039552&type=U&a=rzBx
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2017/11/01/systemtap-for-postgresql-toolkit/"
---
<h2>Introduction</h2>
<p>The purpose of this post is to share some SystemTap tools that have been initially written for oracle and have been adapted for PostgreSQL.</p>
<p>The tools are:</p>
<ul>
<li>pg_schedtimes.stp: To track time spend in various states (run, sleep, iowait, queued)</li>
<li>pg_page_faults.stp: To report the total number of page faults and splits them into Major or Minor faults as well as Read or Write access</li>
<li>pg_traffic.stp: To track the I/O (vfs, block) and Network (tcp, udp, nfs) traffic</li>
</ul>
<p>Those tools are able to group the SystemTap probes per client connections (per database or user) and server processes.</p>
<h2>Grouping the probes</h2>
<p>As described into the <a href="https://www.postgresql.org/docs/9.6/static/monitoring-ps.html" target="_blank" rel="noopener">documentation</a>, on most platforms, PostgreSQL modifies its command title as reported by <em>ps</em>, so that individual server processes can readily be identified.</p>
<p>For example on my lab, the processes are:</p>
<pre style="padding-left:30px;"># ps -ef | grep postgres:
postgres   1460   1447  0 09:35 ?        00:00:00 postgres: logger process
postgres   1462   1447  0 09:35 ?        00:00:00 postgres: checkpointer process
postgres   1463   1447  0 09:35 ?        00:00:00 postgres: writer process
postgres   1464   1447  0 09:35 ?        00:00:00 postgres: wal writer process
postgres   1465   1447  0 09:35 ?        00:00:00 postgres: autovacuum launcher process
postgres   1466   1447  0 09:35 ?        00:00:00 postgres: stats collector process
postgres   7981   1447  0 12:56 ?        00:00:00 postgres: postgres postgres [local] idle
postgres   7983   1447  0 12:56 ?        00:00:00 postgres: bdt postgres 172.16.170.1(56175) idle
postgres   7984   1447  0 12:56 ?        00:00:00 postgres: bdt bdtdb 172.16.170.1(56203) idle</pre>
<ul>
<li>The firsts six processes are background worker processes</li>
<li>Each of the remaining processes is a server process handling one client connection. Each such process sets its command line display in the form "<em>postgres: user database host activity</em>"</li>
</ul>
<p>That said, we can fetch the command line of the processes that trigger the probe event thanks to the <a href="https://sourceware.org/systemtap//tapsets/API-cmdline-str.html" target="_blank" rel="noopener">cmdline_str()</a> function and:</p>
<ul>
<li>filter the processes</li>
<li>extract the piece of information to be used to group the probes</li>
</ul>
<p>So let’s write two <a href="https://sourceware.org/systemtap/langref/Components_SystemTap_script.html#SECTION00046000000000000000" target="_blank" rel="noopener">embedded C functions</a> to extract the relevant piece of information from each of the command line<i> </i>output described above.</p>
<h2>Functions</h2>
<h3>get_pg_dbname:</h3>
<p>[code language="perl"]<br />
function get_pg_dbname:string (mystr:string) %{<br />
char *ptr;<br />
char *ptr2;</p>
<p>int  ch = ' ';<br />
char substr_res[500];<br />
char *strargs = STAP_ARG_mystr;<br />
ptr = strchr( strchr( strargs , ch) + 1 , ch);<br />
ptr2 = strchr( ptr + 1 , ch);<br />
strncpy (substr_res,ptr, ptr2 - ptr);<br />
substr_res[ptr2 - ptr]='&#092;&#048;';<br />
snprintf(STAP_RETVALUE, MAXSTRINGLEN, &quot;%s&quot;,substr_res+1);<br />
%}<br />
[/code]</p>
<p>This function extracts the <strong>database</strong> from any "postgres: user<strong> database</strong> host activity" string</p>
<h3>get_pg_user_proc:</h3>
<p>[code language="perl"]<br />
function get_pg_user_proc:string (mystr:string) %{<br />
char *ptr;<br />
char *ptr2;</p>
<p>int ch = ' ';<br />
char substr_res[500];<br />
char *strargs = STAP_ARG_mystr;<br />
ptr = strchr( strargs , ch);<br />
ptr2 = strchr( ptr + 1 , ch);<br />
strncpy (substr_res,ptr, ptr2 - ptr);<br />
substr_res[ptr2 - ptr]='&#092;&#048;';<br />
snprintf(STAP_RETVALUE, MAXSTRINGLEN, &quot;%s&quot;,substr_res+1);<br />
%}<br />
[/code]</p>
<p>This function extracts:</p>
<ul>
<li>the <strong>user</strong> from any "postgres: <strong>user </strong>database host activity" string</li>
<li>the <strong>procname</strong> from any "postgres: <strong>procname</strong> &lt;any string&gt; process" string</li>
</ul>
<p>Having in mind that the SystemTap aggregation operator is “&lt;&lt;&lt;” (<a href="https://sourceware.org/systemtap/langref/Statistics_aggregates.html" target="_blank" rel="noopener">as explained here</a>) we can use those 2 functions to aggregate within the probes by passing as parameter the <em>cmdline_str().</em></p>
<p>Let's see the usage and output of those tools.</p>
<h2>pg_schedtimes.stp</h2>
<h3>Usage</h3>
<pre style="padding-left:30px;"> $&gt; stap -g ./pg_schedtimes.stp  &lt;pg uid&gt; &lt;refresh time ms&gt; &lt;db|user&gt; &lt;details|nodetails&gt;

 &lt;db|user&gt;: group by db or user
 &lt;details|nodetails&gt;: process server details or grouped all together
</pre>
<h3>Output example</h3>
<p>The postgres userid is 26, and we want to see the time spend in various states by client connections (grouped by database) and all the worker process.</p>
<pre style="padding-left:30px;">$&gt; stap -g ./pg_schedtimes.stp 26 10000 db details

-----------------------------------------------------------------------
                   run(us)  sleep(us) iowait(us) queued(us)  total(us)
-----------------------------------------------------------------------
NOT_PG         :      34849  430368652       1433      35876  430440810
logger         :         81    9986664          0        446    9987191
checkpointer   :         24    9986622          0        543    9987189
writer         :        331    9986227          0        629    9987187
wal            :        248    9986279          0        657    9987184
autovacuum     :        862    9983132          0       3188    9987182
stats          :       2210    9981339          0       3631    9987180
postgres       :      11058    9975156          0        948    9987162
bdtdb          :         13    9986338          0        809    9987160</pre>
<p>I can see the client connections grouped by the bdtdb and postgres databases, the worker processes and all that is not related to PostgreSQL (NOT_PG).</p>
<h2>pg_page_faults.stp</h2>
<h3>Usage</h3>
<pre style="padding-left:30px;"> $&gt; stap -g ./pg_page_faults.stp  &lt;pg uid&gt; &lt;refresh time ms&gt; &lt;db|user&gt; &lt;details|nodetails&gt;

 &lt;db|user&gt;: group by db or user
 &lt;details|nodetails&gt;: process server details or grouped all together
</pre>
<h3>Output example</h3>
<p>The postgres userid is 26, and we want to see the page faults by client connections grouped by user and no worker process details.</p>
<pre style="padding-left:30px;">$&gt; stap -g ./pg_page_faults.stp 26 10000 user nodetails

------------------------------------------------------------------------------------------
                READS_PFLT   WRITES_PFLT  TOTAL_PFLT   MAJOR_PFLT   MINOR_PFLT
------------------------------------------------------------------------------------------
bdt            : 0            294          294          0            294
NOT_PG         : 0            71           71           0            71
PG SERVER      : 0            3            3            0            3</pre>
<p>I can see the client connections grouped by the bdt user, the worker processes grouped all together as "PG SERVER" and all that is not related to PostgreSQL (NOT_PG).</p>
<h2>pg_traffic.stp</h2>
<h3>Usage</h3>
<pre style="padding-left:30px;"> $&gt; stap -g ./pg_traffic.stp &lt;pg uid&gt; &lt;refresh time ms&gt; &lt;io|network|both&gt; &lt;db|user&gt; &lt;details|nodetails&gt;

 &lt;io|network|both&gt;: see io, network or both activity
 &lt;db|user&gt;: group by db or user
 &lt;details|nodetails&gt;: process server details or grouped all together
</pre>
<h3>Output example</h3>
<p>The postgres userid is 26, and we want to see the I/O activity by client connections grouped by database and all the worker process.</p>
<pre style="padding-left:30px;">$&gt; stap -g ./pg_traffic.stp 26 10000 io db details

-------------------------------------------------------------------------------------------------------------
|                                                          I/O                                              |
-------------------------------------------------------------------------------------------------------------
                                  READS                     |                     WRITES                    |
                                                            |                                               |
                       VFS                    BLOCK         |          VFS                    BLOCK         |
            | NB                 KB | NB                 KB | NB                 KB | NB                 KB |
            | --                 -- | --                 -- | --                 -- | --                 -- |
postgres    | 189               380 | 0                   0 | 0                   0 | 0                   0 |
bdtdb       | 38                127 | 0                   0 | 0                   0 | 0                   0 |
NOT_PG      | 79                  2 | 0                   0 | 10                  2 | 0                   0 |
autovacuum  | 49                141 | 0                   0 | 2                   0 | 0                   0 |
stats       | 0                   0 | 0                   0 | 42                 96 | 0                   0 |
-------------------------------------------------------------------------------------------------------------</pre>
<p>I can see the client connections grouped by the bdtdb and postgres databases, the worker processes and all that is not related to PostgreSQL (NOT_PG).</p>
<p>For information, the network output produces something like:</p>
<pre style="padding-left:30px;">-------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                Network                                                                    |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
                                              RECV                                  |                                 SENT                                  |
                                                                                    |                                                                       |
                       TCP                     UDP                     NFS          |          TCP                     UDP                     NFS          |
            | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB | NB                 KB |
            | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- | --                 -- |
postgres    | 95                760 | 0                   0 | 0                   0 | 52                  8 | 0                   0 | 0                   0 |
NOT_PG      | 6                  48 | 0                   0 | 0                   0 | 3                   3 | 0                   0 | 0                   0 |
bdtdb       | 10                 80 | 0                   0 | 0                   0 | 5                   0 | 0                   0 | 0                   0 |
stats       | 0                   0 | 0                   0 | 0                   0 | 0                   0 | 0                   0 | 0                   0 |
-------------------------------------------------------------------------------------------------------------------------------------------------------------
## Source code

The source code is available in&nbsp;[this github repository](https://github.com/bdrouvot/SystemTap)

## Conclusion

Thanks to the embedded C functions we have been able to aggregate the probes and display the information by worker processes or client connections (grouped by database or user).


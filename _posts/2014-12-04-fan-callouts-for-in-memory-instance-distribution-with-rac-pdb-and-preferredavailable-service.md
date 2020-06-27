---
layout: post
title: FAN callouts for In-Memory Instance distribution with RAC, PDB and preferred/available
  service
date: 2014-12-04 16:24:18.000000000 +01:00
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
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/LE4sc6V8U11
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/7zjehcCsh6
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5946292734867304448&type=U&a=kol_
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
permalink: "/2014/12/04/fan-callouts-for-in-memory-instance-distribution-with-rac-pdb-and-preferredavailable-service/"
---
<p><span style="text-decoration:underline;"><strong>The context:</strong></span></p>
<p>In my previous blog post: <a title="In-Memory Instance distribution with RAC and Multitenant Environment" href="http://bdrouvot.wordpress.com/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/" target="_blank">In-Memory Instance distribution with RAC and Multitenant Environment</a>, I checked <strong>if we can influence (get rid of the default behaviour)</strong> how the In-Memory is distributed across all the Instances per PDB. I checked because:</p>
<ul>
<li>By <strong>default</strong> all objects populated into memory will be <strong>distributed</strong> across all of the IM column stores in the cluster.</li>
<li>Christian Antognini show us into this <a href="http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/" target="_blank">blog post </a>that having the data <strong>not fully populated </strong>on an Instance (default behaviour on non Engineered Systems) <strong>could lead to bad performance</strong>.</li>
</ul>
<p><span style="text-decoration:underline;">Let's summarize the findings:</span></p>
<ul>
<li>If the PDB is open <strong>on one Instance</strong>, then this Instance <strong>contains all the data</strong>.</li>
<li>If the PDB is open on all the Instances then we have to set "<strong><em>_inmemory_auto_distribute</em></strong>” to <strong>false to have at least one Instance that contains all the data.</strong></li>
</ul>
<p><span style="text-decoration:underline;"><strong>The objective:</strong></span></p>
<p>That said, let's try to reach an objective: <strong>With a service defined as preferred/available on a PDB, I want the PDB to be open on only one Instance at any time </strong>(That way we don't need to set the hidden parameter).</p>
<p>Into this <a href="http://martincarstenbach.wordpress.com/2014/02/17/rac-and-pluggable-databases/" target="_blank">blog post</a>, Martin Bach already explained the PDB open/close behaviour with a service defined as preferred/available on it. I don't want to repeat Martin's work, I just want to draw your attention on the following (<strong>with our objective in mind</strong>):</p>
<p><span style="text-decoration:underline;">The default PDB status is:</span></p>
<pre style="padding-left:30px;">SYS@CDB$ROOT&gt; select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
         1 MOUNTED
         2 MOUNTED</pre>
<p>As you can see the PDB is mounted on both instances.</p>
<p><span style="text-decoration:underline;">Now, let's create and start a preferred/available service linked to our PDB that way:</span></p>
<pre style="padding-left:30px;">srvctl add service -db SRCCONT -pdb BDTPDB -service BDTPDB_TAF -e select -m BASIC -P BASIC -r SRCCONT1 -a SRCCONT2 -w 5 -z 12
srvctl start service -d SRCCONT</pre>
<p><span style="text-decoration:underline;">Let's check the PDB status and the service status:</span></p>
<pre style="padding-left:30px;">SYS@CDB$ROOT&gt; select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
         2 MOUNTED
         1 READ WRITE</pre>
<pre style="padding-left:30px;">srvctl status service -d SRCCONT
Service BDTPDB_TAF is running on instance(s) SRCCONT1</pre>
<p>So, the PDB is <strong>open</strong> on the Instance that is hosting the service and <strong>not open</strong> on the other Instance:  <strong>The Objective is reached</strong>.</p>
<p><span style="text-decoration:underline;">Now, let's kill pmon for the SRCCONT1 Instance (that hosts the service), wait SRCCONT1 to be back and check the PDB and the service status:</span></p>
<pre style="padding-left:30px;">kill -9 `ps -ef | grep -i pmon | grep -i SRCCONT1 | awk '{print $2}'`

--PDB Status
SYS@CDB$ROOT&gt; select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
         2 READ WRITE
         1 MOUNTED

-- Service Status
srvctl status service -d SRCCONT
Service BDTPDB_TAF is running on instance(s) SRCCONT2
</pre>
<p>So, the service failed over, the PDB has been <strong>closed</strong> on the failing Instance and is <strong>open</strong> on the one that is now hosting the service: <strong>The Objective is reached</strong>.</p>
<p><span style="text-decoration:underline;">Now, let's (fail back) relocate the service manually and check the PDB and the service status:</span></p>
<pre style="padding-left:30px;">srvctl relocate service -d SRCCONT -s BDTPDB_TAF -oldinst SRCCONT2 -newinst SRCCONT1 -force

--PDB Status
SYS@CDB$ROOT&gt;  select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
         1 READ WRITE
         2 READ WRITE

-- Service Status
srvctl status service -d SRCCONT                                                                         
Service BDTPDB_TAF is running on instance(s) SRCCONT1
</pre>
<p>As you can see the service has been relocated and the PDB is now <strong>open</strong> on both Instances: <strong>The Objective is not reached.</strong></p>
<p><span style="text-decoration:underline;">To summarize:</span></p>
<ol>
<li>The objective has been reached when we created and started the service.</li>
<li>The objective has been reached when the service failed over (Due to Instance crash).</li>
<li>The objective has not been reached when the service has been relocated manually.</li>
</ol>
<p><span style="text-decoration:underline;"><strong>FAN callouts to the rescue:</strong></span></p>
<p>FAN callouts are server-side scripts or executables that run whenever a FAN event is generated. I'll use this feature to fix the third point where the objective is not reached.</p>
<p>To do so, I created the following script:</p>
<p>[code language="bash"]<br />
#!/bin/ksh<br />
#<br />
# Close the PDB on the old intance during manual service relocation<br />
#<br />
# Description:<br />
# -  This script closes the PDB on the old intance during manual service relocation<br />
#<br />
# Requirement:<br />
#  - Location of the script must be $ORA_CRS_HOME/racg/usrco/.<br />
#<br />
# Version:<br />
#  - version 1.0<br />
#  - Bertrand Drouvot<br />
#<br />
# Date: 2014/12/04<br />
#<br />
#<br />
# Env settings</p>
<p>ME=`who | cut -d&quot; &quot; -f1`<br />
PASSWDFILE=&quot;/etc/passwd&quot;<br />
HOMEDIR=`egrep &quot;^${ME}&quot; ${PASSWDFILE} | cut -d: -f 6`<br />
DATE_LOG=$(date +&quot;%y%m%d_%H%M&quot;)<br />
LOGFILE=fan_cloe_pdb_${DATE_LOG}.log</p>
<p>VAR_EVENTTYPE=$1</p>
<p>for ARGS in $*;<br />
 do<br />
        PROPERTY=`echo $ARGS | cut -f1 -d&quot;=&quot; | tr '[:lower:]' '[:upper:]'`<br />
        VALUE=`echo $ARGS | cut -f2 -d&quot;=&quot; | tr '[:lower:]' '[:upper:]'`<br />
        case $PROPERTY in<br />
                VERSION)      VAR_VERSION=$VALUE ;;<br />
                SERVICE)      VAR_SERVICE=$VALUE ;;<br />
                DATABASE)     VAR_DATABASE=$VALUE ;;<br />
                INSTANCE)     VAR_INSTANCE=$VALUE ;;<br />
                HOST)         VAR_HOST=$VALUE ;;<br />
                STATUS)       VAR_STATUS=$VALUE ;;<br />
                REASON)       VAR_REASON=$VALUE ;;<br />
                CARD)         VAR_CARDINALITY=$VALUE ;;<br />
                TIMESTAMP)    VAR_LOGDATE=$VALUE ;;<br />
                ??:??:??)     VAR_LOGTIME=$VALUE ;;<br />
        esac<br />
 done </p>
<p># Close the PDB (PDB name extracted from the service name) if service relocated manually</p>
<p>if ( ([ $VAR_EVENTTYPE = &quot;SERVICEMEMBER&quot; ]) &amp;&amp; ([ $VAR_STATUS = &quot;DOWN&quot; ]) &amp;&amp; ([ $VAR_REASON = &quot;USER&quot; ]))<br />
then</p>
<p>PATH=$PATH:/usr/local/bin<br />
export pATH<br />
ORAENV_ASK=NO<br />
export ORAENV_ASK<br />
ORACLE_SID=${VAR_INSTANCE}<br />
export ORACLE_SID<br />
. oraenv &gt; /dev/null</p>
<p># Extract PDB name based on our naming convention (You may need to change this)<br />
PDB=`echo &quot;${VAR_SERVICE}&quot; | sed 's/_TAF.*//'`</p>
<p># Close the PDB on the old instance<br />
sqlplus /nolog &lt;&lt;EOF<br />
connect / as sysdba;<br />
alter pluggable database ${PDB} close instances=('${VAR_INSTANCE}');<br />
EOF</p>
<p># Log this in a logfile<br />
echo ${*} &gt;&gt; ${HOMEDIR}/log/${LOGFILE}</p>
<p>fi<br />
[/code]</p>
<p>under <b>$GRID_HOME/racg/usrco/</b> (on both nodes). Then the script will be executed for all FAN events, but the script will start processing only for manual service relocation (thanks to the "if condition" in it).</p>
<p>The goal of the script is to <strong>close</strong> the PDB on the <strong>old Instance</strong> during manual service relocation.</p>
<p><span style="text-decoration:underline;">Now, let's check if the objective is reached. Let's relocate the service manually from SRCCONT1 to SRCCONT2:</span></p>
<pre style="padding-left:30px;">-- Current PDB Status
SYS@CDB$ROOT&gt; select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
         1 READ WRITE
         2 READ WRITE

-- Current service status
SYS@CDB$ROOT&gt; !srvctl status service -d SRCCONT
Service BDTPDB_TAF is running on instance(s) SRCCONT1

-- Relocate the service
SYS@CDB$ROOT&gt; !srvctl relocate service -d SRCCONT -s BDTPDB_TAF -oldinst SRCCONT1 -newinst SRCCONT2 -force

-- Check PDB status
SYS@CDB$ROOT&gt; select inst_id,open_mode from gv$pdbs where name='BDTPDB';

   INST_ID OPEN_MODE
---------- ----------
2 READ WRITE 1 MOUNTED -- Check service status SYS@CDB$ROOT\> !srvctl status service -d SRCCONT Service BDTPDB\_TAF is running on instance(s) SRCCONT2

So, As you can see the service has been manually relocated and the PDB has been closed to the old Instance. That way, the PDB is open on only one Instance: **The Objective is reached.**

**Remarks:**

- I used the "_force_" option during the manual relocation which disconnects all the sessions during the relocate operation. If you are not using the "force" option, then I guess you don't want to close the PDB on the old Instance ;-).
- The PDB open/close behaviour with a service linked as preferred/available on it is the one described into this post, unless you have set the [save state](http://oracle-base.com/articles/12c/multitenant-startup-and-shutdown-cdb-and-pdb-12cr1.php) on it (default is discard state).

**Conclusion:**

The objective is reached in any case: The PDB is open on **only one Instance** at any time (with a **preferred/available** service in place). That way we don't need to set the hidden parameter "_\_inmemory\_auto\_distribute_” to false to get the In-Memory data **fully populated** on one Instance.

**Update 2015/03/27** : At the PDB level, If you set the _parallel\_instance\_group_ parameter to the service on both Instances, then the In-Memory data is **fully populated** on the Instance the service is running on **even if** the PDB is open on both Instances (After the manual service relocation). You can find more details [here](https://bdrouvot.wordpress.com/2015/03/26/in-memory-instance-distribution-with-rac-databases-impact-of-the-parallel_instance_group-parameter/ "In-Memory Instance distribution with RAC databases: Impact of the parallel\_instance\_group parameter").


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
tags: [Oracle RAC, oracle]
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

<span style="text-decoration:underline;">**The context:**</span>

In my previous blog post: [In-Memory Instance distribution with RAC and Multitenant Environment](http://bdrouvot.wordpress.com/2014/11/21/in-memory-instance-distribution-with-rac-and-multitenant-environment/ "In-Memory Instance distribution with RAC and Multitenant Environment"), I checked **if we can influence (get rid of the default behaviour)** how the In-Memory is distributed across all the Instances per PDB. I checked because:

-   By **default** all objects populated into memory will be **distributed** across all of the IM column stores in the cluster.
-   Christian Antognini show us into this [blog post](http://antognini.ch/2014/11/the-importance-of-the-in-memory-duplicate-clause-for-a-rac-system/) that having the data **not fully populated** on an Instance (default behaviour on non Engineered Systems) **could lead to bad performance**.

<span style="text-decoration:underline;">Let's summarize the findings:</span>

-   If the PDB is open **on one Instance**, then this Instance **contains all the data**.
-   If the PDB is open on all the Instances then we have to set "***\_inmemory\_auto\_distribute***” to **false to have at least one Instance that contains all the data.**

<span style="text-decoration:underline;">**The objective:**</span>

That said, let's try to reach an objective: **With a service defined as preferred/available on a PDB, I want the PDB to be open on only one Instance at any time** (That way we don't need to set the hidden parameter).

Into this [blog post](http://martincarstenbach.wordpress.com/2014/02/17/rac-and-pluggable-databases/), Martin Bach already explained the PDB open/close behaviour with a service defined as preferred/available on it. I don't want to repeat Martin's work, I just want to draw your attention on the following (**with our objective in mind**):

<span style="text-decoration:underline;">The default PDB status is:</span>

    SYS@CDB$ROOT> select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             1 MOUNTED
             2 MOUNTED

As you can see the PDB is mounted on both instances.

<span style="text-decoration:underline;">Now, let's create and start a preferred/available service linked to our PDB that way:</span>

    srvctl add service -db SRCCONT -pdb BDTPDB -service BDTPDB_TAF -e select -m BASIC -P BASIC -r SRCCONT1 -a SRCCONT2 -w 5 -z 12
    srvctl start service -d SRCCONT

<span style="text-decoration:underline;">Let's check the PDB status and the service status:</span>

    SYS@CDB$ROOT> select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             2 MOUNTED
             1 READ WRITE

    srvctl status service -d SRCCONT
    Service BDTPDB_TAF is running on instance(s) SRCCONT1

So, the PDB is **open** on the Instance that is hosting the service and **not open** on the other Instance:  **The Objective is reached**.

<span style="text-decoration:underline;">Now, let's kill pmon for the SRCCONT1 Instance (that hosts the service), wait SRCCONT1 to be back and check the PDB and the service status:</span>

    kill -9 `ps -ef | grep -i pmon | grep -i SRCCONT1 | awk '{print $2}'`

    --PDB Status
    SYS@CDB$ROOT> select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             2 READ WRITE
             1 MOUNTED

    -- Service Status
    srvctl status service -d SRCCONT
    Service BDTPDB_TAF is running on instance(s) SRCCONT2

So, the service failed over, the PDB has been **closed** on the failing Instance and is **open** on the one that is now hosting the service: **The Objective is reached**.

<span style="text-decoration:underline;">Now, let's (fail back) relocate the service manually and check the PDB and the service status:</span>

    srvctl relocate service -d SRCCONT -s BDTPDB_TAF -oldinst SRCCONT2 -newinst SRCCONT1 -force

    --PDB Status
    SYS@CDB$ROOT>  select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             1 READ WRITE
             2 READ WRITE

    -- Service Status
    srvctl status service -d SRCCONT                                                                         
    Service BDTPDB_TAF is running on instance(s) SRCCONT1

As you can see the service has been relocated and the PDB is now **open** on both Instances: **The Objective is not reached.**

<span style="text-decoration:underline;">To summarize:</span>

1.  The objective has been reached when we created and started the service.
2.  The objective has been reached when the service failed over (Due to Instance crash).
3.  The objective has not been reached when the service has been relocated manually.

<span style="text-decoration:underline;">**FAN callouts to the rescue:**</span>

FAN callouts are server-side scripts or executables that run whenever a FAN event is generated. I'll use this feature to fix the third point where the objective is not reached.

To do so, I created the following script:

```
#!/bin/ksh  
#  
# Close the PDB on the old intance during manual service relocation  
#  
# Description:  
# - This script closes the PDB on the old intance during manual service relocation  
#  
# Requirement:  
# - Location of the script must be $ORA_CRS_HOME/racg/usrco/.  
#  
# Version:  
# - version 1.0  
# - Bertrand Drouvot  
#  
# Date: 2014/12/04  
#  
#  
# Env settings

ME=`who | cut -d" " -f1`  
PASSWDFILE="/etc/passwd"  
HOMEDIR=`egrep "^${ME}" ${PASSWDFILE} | cut -d: -f 6`  
DATE_LOG=$(date +"%y%m%d_%H%M")  
LOGFILE=fan_cloe_pdb_${DATE_LOG}.log

VAR_EVENTTYPE=$1

for ARGS in $*;  
do  
PROPERTY=`echo $ARGS | cut -f1 -d"=" | tr '[:lower:]' '[:upper:]'`  
VALUE=`echo $ARGS | cut -f2 -d"=" | tr '[:lower:]' '[:upper:]'`  
case $PROPERTY in  
VERSION) VAR_VERSION=$VALUE ;;  
SERVICE) VAR_SERVICE=$VALUE ;;  
DATABASE) VAR_DATABASE=$VALUE ;;  
INSTANCE) VAR_INSTANCE=$VALUE ;;  
HOST) VAR_HOST=$VALUE ;;  
STATUS) VAR_STATUS=$VALUE ;;  
REASON) VAR_REASON=$VALUE ;;  
CARD) VAR_CARDINALITY=$VALUE ;;  
TIMESTAMP) VAR_LOGDATE=$VALUE ;;  
??:??:??) VAR_LOGTIME=$VALUE ;;  
esac  
done

# Close the PDB (PDB name extracted from the service name) if service relocated manually

if ( ([ $VAR_EVENTTYPE = "SERVICEMEMBER" ]) && ([ $VAR_STATUS = "DOWN" ]) && ([ $VAR_REASON = "USER" ]))  
then

PATH=$PATH:/usr/local/bin  
export pATH  
ORAENV_ASK=NO  
export ORAENV_ASK  
ORACLE_SID=${VAR_INSTANCE}  
export ORACLE_SID  
. oraenv > /dev/null

# Extract PDB name based on our naming convention (You may need to change this)  
PDB=`echo "${VAR_SERVICE}" | sed 's/_TAF.*//'`

# Close the PDB on the old instance  
sqlplus /nolog &lt;&lt;EOF  
connect / as sysdba;  
alter pluggable database ${PDB} close instances=('${VAR_INSTANCE}');  
EOF

# Log this in a logfile  
echo ${*} >> ${HOMEDIR}/log/${LOGFILE}

fi  
```

under **$GRID\_HOME/racg/usrco/** (on both nodes). Then the script will be executed for all FAN events, but the script will start processing only for manual service relocation (thanks to the "if condition" in it).

The goal of the script is to **close** the PDB on the **old Instance** during manual service relocation.

<span style="text-decoration:underline;">Now, let's check if the objective is reached. Let's relocate the service manually from SRCCONT1 to SRCCONT2:</span>

    -- Current PDB Status
    SYS@CDB$ROOT> select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             1 READ WRITE
             2 READ WRITE

    -- Current service status
    SYS@CDB$ROOT> !srvctl status service -d SRCCONT
    Service BDTPDB_TAF is running on instance(s) SRCCONT1

    -- Relocate the service
    SYS@CDB$ROOT> !srvctl relocate service -d SRCCONT -s BDTPDB_TAF -oldinst SRCCONT1 -newinst SRCCONT2 -force

    -- Check PDB status
    SYS@CDB$ROOT> select inst_id,open_mode from gv$pdbs where name='BDTPDB';

       INST_ID OPEN_MODE
    ---------- ----------
             2 READ WRITE
             1 MOUNTED

    -- Check service status
    SYS@CDB$ROOT> !srvctl status service -d SRCCONT
    Service BDTPDB_TAF is running on instance(s) SRCCONT2

So, As you can see the service has been manually relocated and the PDB has been closed to the old Instance. That way, the PDB is open on only one Instance: **The Objective is reached.**

<span style="text-decoration:underline;">**Remarks:**</span>

-   I used the "*force*" option during the manual relocation which disconnects all the sessions during the relocate operation. If you are not using the "force" option, then I guess you don't want to close the PDB on the old Instance ;-).
-   The PDB open/close behaviour with a service linked as preferred/available on it is the one described into this post, unless you have set the [save state](http://oracle-base.com/articles/12c/multitenant-startup-and-shutdown-cdb-and-pdb-12cr1.php) on it (default is discard state).

<span style="text-decoration:underline;">**Conclusion:**</span>

The objective is reached in any case: The PDB is open on **only one Instance** at any time (with a **preferred/available** service in place). That way we don't need to set the hidden parameter "*\_inmemory\_auto\_distribute*” to false to get the In-Memory data **fully populated** on one Instance.

**Update 2015/03/27**: At the PDB level, If you set the *parallel\_instance\_group* parameter to the service on both Instances, then the In-Memory data is **fully populated** on the Instance the service is running on **even if** the PDB is open on both Instances (After the manual service relocation). You can find more details [here](https://bdrouvot.wordpress.com/2015/03/26/in-memory-instance-distribution-with-rac-databases-impact-of-the-parallel_instance_group-parameter/ "In-Memory Instance distribution with RAC databases: Impact of the parallel_instance_group parameter").

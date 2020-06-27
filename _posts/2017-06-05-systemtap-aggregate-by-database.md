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

Introduction
------------

The purpose of this post is to share a way to aggregate by Oracle database within SystemTap probes. Let's describe a simple use case to make things clear.

Use Case
--------

Let's say that I want to get the number and the total size of TCP messages that have been sent and received by an Oracle database. To do so, let's use 2 probes:

-   [probe::tcp.recvmsg](https://sourceware.org/systemtap/tapsets/API-tcp-recvmsg.html)
-   [probe::tcp.sendmsg](https://sourceware.org/systemtap/tapsets/API-tcp-sendmsg.html)

and fetch the command line of the processes that trigger the event thanks to the [cmdline\_str()](https://sourceware.org/systemtap//tapsets/API-cmdline-str.html) function. In case of a process related to an oracle database, the *cmdline\_str()* output would look like one of those 2:

-   ora\_&lt;*something*&gt;\_&lt;*instance\_name*&gt;
-   oracle&lt;*instance\_name*&gt; (LOCAL=&lt;something&gt;

So let's write two [embedded C functions](https://sourceware.org/systemtap/langref/Components_SystemTap_script.html#SECTION00046000000000000000) to extract the Instance name from each of the 2 strings described above.

Functions
---------

### get\_oracle\_name\_b:

For example, this function would extract BDT from "ora\_dbw0\_BDT" or any "ora\_&lt;*something*&gt;\_BDT" string.

The code is the following:

```
function get\_oracle\_name\_b:string (mystr:string) %{  
char \*ptr;  
int ch = '\_';  
char \*strargs = STAP\_ARG\_mystr;  
ptr = strchr( strchr( strargs , ch) + 1 , ch);  
snprintf(STAP\_RETVALUE, MAXSTRINGLEN, "%s",ptr + 1);  
%}  
```

### get\_oracle\_name\_f:

For example, this function would extract BDT from "oracleBDT (LOCAL=NO)", "oracleBDT (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq)))" or any "oracleBDT (LOCAL=&lt;something&gt;" string.

The code is the following:

```
function get\_oracle\_name\_f:string (mystr:string) %{  
char \*ptr;  
int ch = ' ';  
char substr\_res\[30\];  
char \*strargs = STAP\_ARG\_mystr;  
ptr = strchr( strargs, ch );  
strncpy (substr\_res,strargs+6, ptr - strargs - 6);  
substr\_res\[ptr - strargs - 6\]='\\0';  
snprintf(STAP\_RETVALUE, MAXSTRINGLEN, "%s",substr\_res);  
%}  
```

Having in mind that the SystemTap aggregation operator is "&lt;&lt;&lt;" ([as explained here](https://sourceware.org/systemtap/langref/Statistics_aggregates.html)) we can use those 2 functions to aggregate within the probes by Instance Name (passing as parameter the *cmdline\_str())* that way:

```
probe tcp.recvmsg  
{

if ( isinstr(cmdline\_str() , "ora\_" ) == 1 && uid() == orauid) {  
tcprecv\[get\_oracle\_name\_b(cmdline\_str())\] &lt;&lt;&lt; size  
} else if ( isinstr(cmdline\_str() , "LOCAL=" ) == 1 && uid() == orauid) {  
tcprecv\[get\_oracle\_name\_f(cmdline\_str())\] &lt;&lt;&lt; size  
} else {  
tcprecv\["NOT\_A\_DB"\] &lt;&lt;&lt; size  
}  
}

probe tcp.sendmsg  
{

if ( isinstr(cmdline\_str() , "ora\_" ) == 1 && uid() == orauid) {  
tcpsend\[get\_oracle\_name\_b(cmdline\_str())\] &lt;&lt;&lt; size  
} else if ( isinstr(cmdline\_str() , "LOCAL=" ) == 1 && uid() == orauid) {  
tcpsend\[get\_oracle\_name\_f(cmdline\_str())\] &lt;&lt;&lt; size  
} else {  
tcpsend\["NOT\_A\_DB"\] &lt;&lt;&lt; size  
}  
}  
```

As you can see, non oracle database would be recorded and displayed as "NOT\_A\_DB".

Based on this, the *tcp\_by\_db.stp* script has been created.

tcp\_by\_db.stp: script usage and output example
------------------------------------------------

### Usage:

    $> stap -g ./tcp_by_db.stp <oracle uid> <refresh time ms>

### Output Example:

    $> stap -g ./tcp_by_db.stp 54321 10000
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
    VBDT              340       176       510       2940
    NOT_A_DB          150       8         151       1165
    BDT               42        77        61        423

Remarks:

-   The oracle uid on my machine is 54321
-   The refresh time has been set to 10 seconds
-   You can see the aggregation for 2 databases on my machine and also for all that is not an oracle database

Whole Script source code
------------------------

The whole code is available at:

-   [This github repository](https://github.com/bdrouvot/SystemTap)
-   [This page](https://bdrouvot.wordpress.com/tcp_by_dbs-stp_script/)

Conclusion
----------

Thanks to the embedded C functions we have been able to aggregate by Oracle database within SystemTap probes.

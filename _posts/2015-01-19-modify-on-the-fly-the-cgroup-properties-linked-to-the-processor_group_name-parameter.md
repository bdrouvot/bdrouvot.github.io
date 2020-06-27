---
layout: post
title: Modify on the fly the cgroup properties linked to the processor_group_name
  parameter
date: 2015-01-19 17:11:43.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/X4D4dtgBXZT
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/6AScwxSCaS
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5962974515695742978&type=U&a=TXbf
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
permalink: "/2015/01/19/modify-on-the-fly-the-cgroup-properties-linked-to-the-processor_group_name-parameter/"
---
<p><span style="text-decoration:underline;"><strong>Introduction</strong></span></p>
<p>I am preparing a blog post in which I'll try to describe pros and cons of cpu binding (using the <em>processor_group_name</em> parameter) versus Instance caging.</p>
<p><span style="text-decoration:underline;">Today's comparison:</span></p>
<p>As you know you can change the <em>cpu_count</em> and <em>resource_manager_plan </em>parameters while the database Instance is up and running. Then, while the database Instance is up and running you can:</p>
<ul>
<li><strong>enable / disable Instance caging</strong>.</li>
<li><strong>change the <em>cpu_count</em></strong> value and as a consequence the cpu limitation (with the Instance caging already in place).</li>
</ul>
<p>Fine, can we do the same <strong>with cpu binding</strong>? What is the effect of changing the cgroup attributes that are linked to the <em>processor_group_name</em> parameter while the database Instance is up and running?</p>
<p>Let's test it.</p>
<p><span style="text-decoration:underline;"><strong>Initial setup</strong></span></p>
<p>The database version is 12.1.0.2 and the  <em>_enable_NUMA_support</em> parameter has been kept to its default value: FALSE.</p>
<p>With this NUMA setup on a single machine:</p>
<pre style="padding-left:30px;">NUMA node0 CPU(s):     1-10,41-50
NUMA node1 CPU(s):     11-20,51-60
NUMA node2 CPU(s):     21-30,61-70
NUMA node3 CPU(s):     31-40,71-80
NUMA node4 CPU(s):     0,81-89,120-129
NUMA node5 CPU(s):     90-99,130-139
NUMA node6 CPU(s):     100-109,140-149
NUMA node7 CPU(s):     110-119,150-159</pre>
<p>A cgroup (named oracle) has been created with those properties:</p>
<pre style="padding-left:30px;">group oracle {
   perm {
     task {
       uid = oracle;
       gid = dc_dba;
     }
     admin {
       uid = oracle;
       gid = dc_dba;
     }
   }
   cpuset {
     <strong>cpuset.mems="0"</strong>;
     <strong>cpuset.cpus="1-6"</strong>;
   }
}</pre>
<p>Means that I want the SGA to be allocated on <strong>NUMA node 0 and to use the cpus 1 to 6</strong>.</p>
<p>The database has been started using this cgroup thanks to the <em>processor_group_name </em>parameter (The database has to be restarted):</p>
<pre style="padding-left:30px;">SQL&gt; alter system set processor_group_name='oracle' scope=spfile sid='*';

System altered.</pre>
<p>During the database startup you can see the following into the alert.log file:</p>
<pre style="padding-left:30px;">Instance started in processor group oracle (<strong>NUMA Nodes: 0 CPUs: 1-6</strong>)</pre>
<p>You can also check that the SGA has been allocated from NUMA node 0:</p>
<pre style="padding-left:30px;">&gt; grep -i HugePages /sys/devices/system/node/node*/meminfo
/sys/devices/system/node/node0/meminfo:Node 0 <strong>HugePages_Total: 39303</strong>
/sys/devices/system/node/node0/meminfo:Node 0 <strong>HugePages_Free:  23589</strong>
/sys/devices/system/node/node0/meminfo:Node 0 HugePages_Surp:      0
/sys/devices/system/node/node1/meminfo:Node 1 HugePages_Total: 39302
/sys/devices/system/node/node1/meminfo:Node 1 HugePages_Free:  39046
/sys/devices/system/node/node1/meminfo:Node 1 HugePages_Surp:      0
/sys/devices/system/node/node2/meminfo:Node 2 HugePages_Total: 39302
/sys/devices/system/node/node2/meminfo:Node 2 HugePages_Free:  39047
/sys/devices/system/node/node2/meminfo:Node 2 HugePages_Surp:      0
/sys/devices/system/node/node3/meminfo:Node 3 HugePages_Total: 39302
/sys/devices/system/node/node3/meminfo:Node 3 HugePages_Free:  39047
/sys/devices/system/node/node3/meminfo:Node 3 HugePages_Surp:      0
/sys/devices/system/node/node4/meminfo:Node 4 HugePages_Total: 39302
/sys/devices/system/node/node4/meminfo:Node 4 HugePages_Free:  39047
/sys/devices/system/node/node4/meminfo:Node 4 HugePages_Surp:      0
/sys/devices/system/node/node5/meminfo:Node 5 HugePages_Total: 39302
/sys/devices/system/node/node5/meminfo:Node 5 HugePages_Free:  39047
/sys/devices/system/node/node5/meminfo:Node 5 HugePages_Surp:      0
/sys/devices/system/node/node6/meminfo:Node 6 HugePages_Total: 39302
/sys/devices/system/node/node6/meminfo:Node 6 HugePages_Free:  39047
/sys/devices/system/node/node6/meminfo:Node 6 HugePages_Surp:      0
/sys/devices/system/node/node7/meminfo:Node 7 HugePages_Total: 39302
/sys/devices/system/node/node7/meminfo:Node 7 HugePages_Free:  39047
/sys/devices/system/node/node7/meminfo:Node 7 HugePages_Surp:      0</pre>
<p>As you can see, 39303-23589 large pages have been allocated from node 0. This is the amount of large pages needed for the SGA.</p>
<p>We can check that the <em>cpu_count</em> parameter has been <strong>automatically</strong> set to 6:</p>
<pre style="padding-left:30px;">SQL&gt; select value from v$parameter where name='cpu_count';

VALUE
--------------------------------------------------------------------------------
6

We can also check the cpu affinity of a background process (_smon_ for example):

```
\> taskset -c -p `ps -ef | grep -i smon | grep BDT12CG | awk '{print $2}'` pid 1019968's current affinity list: **1-6**
```

Note that the same affinity would have been observed with a foreground process.

**Now let's change the&nbsp;_cpuset.cpus_ attribute while the database instance is up and running:**

I want the cpus 11 to 20 to be used (instead of 1 to 6):

```
\> /bin/echo **11-20** \> /cgroup/cpuset/oracle/cpuset.cpus
```

After a few seconds, those lines appear into the alert.log file:

```
Detected change in CPU count to 10
```

Great, it looks like the database is aware of the new _cpuset.cpus_ value. Let's check the _smon_ process cpu affinity :

```
\> taskset -c -p `ps -ef | grep -i smon | grep BDT12CG | awk '{print $2}'` pid 1019968's current affinity list: **11-20**
```

**BINGO** : The database is now using the new value (means the new cpus) and we don't need to bounce it!

Remarks:

- Depending of the new **_cpuset.cpus_** value the SGA allocation may not be "[optimal](https://bdrouvot.wordpress.com/2015/01/07/measure-the-impact-of-remote-versus-local-numa-node-access-thanks-to-processor_group_name/ "Measure the impact of remote versus local NUMA node access thanks to processor\_group\_name")" (If the cpus are not on the same NUMA node where the SGA is currently allocated).

- **Don't forget to update** the _/etc/cgconfig.conf_ file accordingly (If not, the restart of the _cgconfig_ service will overwrite your changes).

- I did the same change **during** a full cached [SLOB](http://kevinclosson.net/slob) run of 9 readers, and the _mpstat -P ALL 3_ output moved from:

```
02:21:54 PM CPU %usr %nice %sys %iowait %irq %soft %steal %guest %idle 02:21:57 PM all 3.79 0.01 0.21 0.01 0.00 0.00 0.00 0.00 95.97 02:21:57 PM 0 0.34 0.00 0.00 0.00 0.00 0.00 0.00 0.00 99.66 02:21:57 PM 1 98.66 0.00 1.34 0.00 0.00 0.00 0.00 0.00 0.00 02:21:57 PM 2 98.67 0.00 1.33 0.00 0.00 0.00 0.00 0.00 0.00 02:21:57 PM 3 92.31 0.00 7.69 0.00 0.00 0.00 0.00 0.00 0.00 02:21:57 PM 4 97.00 0.00 3.00 0.00 0.00 0.00 0.00 0.00 0.00 02:21:57 PM 5 98.34 0.00 1.66 0.00 0.00 0.00 0.00 0.00 0.00 02:21:57 PM 6 99.33 0.00 0.67 0.00 0.00 0.00 0.00 0.00 0.00 . . . 02:21:57 PM 11 1.34 1.68 0.34 0.00 0.00 0.00 0.00 0.00 96.64 02:21:57 PM 12 0.67 0.00 0.00 0.34 0.00 0.00 0.00 0.00 98.99 02:21:57 PM 13 0.33 0.00 0.00 0.00 0.00 0.00 0.00 0.00 99.67 02:21:57 PM 14 0.33 0.00 0.33 0.00 0.00 0.00 0.00 0.00 99.33 02:21:57 PM 15 0.34 0.00 0.34 0.00 0.00 0.00 0.00 0.00 99.33 02:21:57 PM 16 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:21:57 PM 17 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:21:57 PM 18 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:21:57 PM 19 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:21:57 PM 20 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00
```

to:

```
02:22:00 PM CPU %usr %nice %sys %iowait %irq %soft %steal %guest %idle 02:22:03 PM all 5.71 0.00 0.18 0.02 0.00 0.00 0.00 0.00 94.09 02:22:03 PM 0 0.33 0.00 0.00 0.00 0.00 0.33 0.00 0.00 99.33 02:22:03 PM 1 0.34 0.00 0.00 0.00 0.00 0.00 0.00 0.00 99.66 02:22:03 PM 2 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:22:03 PM 3 2.70 0.00 6.08 0.00 0.00 0.00 0.00 0.00 91.22 02:22:03 PM 4 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:22:03 PM 5 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 02:22:03 PM 6 0.00 0.00 0.00 0.00 0.00 0.00 0.00 0.00 100.00 . . . 02:22:03 PM 11 2.05 0.00 1.71 0.34 0.00 0.00 0.00 0.00 95.89 02:22:03 PM 12 99.00 0.00 1.00 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 13 99.33 0.00 0.67 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 14 99.00 0.00 1.00 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 15 97.32 0.00 2.68 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 16 98.67 0.00 1.33 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 17 99.33 0.00 0.67 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 18 99.00 0.00 1.00 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 19 99.00 0.00 1.00 0.00 0.00 0.00 0.00 0.00 0.00 02:22:03 PM 20 99.67 0.00 0.33 0.00 0.00 0.00 0.00 0.00 0.00
```

As you can see the process activity moved from cpus 1-6 to cpus 11-20: **This is exactly what we want**.

**Now, out of curiosity: what about the memory? let's change the&nbsp;_cpuset.mems_ property while the database is up and running:**

Let's say that we want the SGA to migrate from NUMA node 0 to node 1.

To achieve this we also need to set the cpuset [_memory\_migrate_](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpuset.html) attribute to 1 (default is 0: means memory allocation is not following _cpuset.mems_ update).

So, let's change the _memory\_migrate_ attribute to 1 and _cpuset.mems_ from node 0 to node 1:

```
/bin/echo 1 \> /cgroup/cpuset/oracle/cpuset.memory\_migrate /bin/echo 1 \> /cgroup/cpuset/oracle/cpuset.mems
```

What is the effect on the SGA allocation?

```
grep -i HugePages /sys/devices/system/node/node\*/meminfo /sys/devices/system/node/node0/meminfo:Node 0 HugePages\_Total: 39303 /sys/devices/system/node/node0/meminfo:Node 0 HugePages\_Free: 23589 /sys/devices/system/node/node0/meminfo:Node 0 HugePages\_Surp: 0 /sys/devices/system/node/node1/meminfo:Node 1 HugePages\_Total: 39302 /sys/devices/system/node/node1/meminfo:Node 1 HugePages\_Free: 39046 /sys/devices/system/node/node1/meminfo:Node 1 HugePages\_Surp: 0 /sys/devices/system/node/node2/meminfo:Node 2 HugePages\_Total: 39302 /sys/devices/system/node/node2/meminfo:Node 2 HugePages\_Free: 39047 /sys/devices/system/node/node2/meminfo:Node 2 HugePages\_Surp: 0 /sys/devices/system/node/node3/meminfo:Node 3 HugePages\_Total: 39302 /sys/devices/system/node/node3/meminfo:Node 3 HugePages\_Free: 39047 /sys/devices/system/node/node3/meminfo:Node 3 HugePages\_Surp: 0 /sys/devices/system/node/node4/meminfo:Node 4 HugePages\_Total: 39302 /sys/devices/system/node/node4/meminfo:Node 4 HugePages\_Free: 39047 /sys/devices/system/node/node4/meminfo:Node 4 HugePages\_Surp: 0 /sys/devices/system/node/node5/meminfo:Node 5 HugePages\_Total: 39302 /sys/devices/system/node/node5/meminfo:Node 5 HugePages\_Free: 39047 /sys/devices/system/node/node5/meminfo:Node 5 HugePages\_Surp: 0 /sys/devices/system/node/node6/meminfo:Node 6 HugePages\_Total: 39302 /sys/devices/system/node/node6/meminfo:Node 6 HugePages\_Free: 39047 /sys/devices/system/node/node6/meminfo:Node 6 HugePages\_Surp: 0 /sys/devices/system/node/node7/meminfo:Node 7 HugePages\_Total: 39302 /sys/devices/system/node/node7/meminfo:Node 7 HugePages\_Free: 39047 /sys/devices/system/node/node7/meminfo:Node 7 HugePages\_Surp: 0
```

**None:** as you can see the SGA is still allocated on node 0 (I observed the same during the full cached SLOB run).

Then, let's try to stop the database, add _memory\_migrate_ attribute into the _/etc/cgconfig.conf_ file that way:

```
&nbsp;&nbsp; cpuset { &nbsp;&nbsp;&nbsp;&nbsp; cpuset.mems="0"; &nbsp;&nbsp;&nbsp;&nbsp; cpuset.cpus="1-6"; &nbsp;&nbsp;&nbsp;&nbsp; **cpuset.memory\_migrate=1** ; &nbsp;&nbsp; }
```

restart the _cgconfig_ service:

```
\> service cgconfig restart
```

and check the value:

```
\> cat /cgroup/cpuset/oracle/cpuset.memory\_migrate 1
```

Well it is set to 1, let's start the database and check again:

```
\> cat /cgroup/cpuset/oracle/cpuset.memory\_migrate 0
```

**Hey! It has been set back to 0**  **after the database startup**. So, let's conclude that, by default, the memory can not be **dynamically** migrated from one NUMA node to another by **changing on the fly the _cpuset.mems_** value.

Remark:

Obviously, changing the _cpuset.mems_ attribute from node 0 to node 1:

```
\> /bin/echo 1 \> /cgroup/cpuset/oracle/cpuset.mems
```

and **bounce** the Instance will allocate the memory on the new node. Again, **don't forget to update** the _/etc/cgconfig.conf_ file accordingly (If not, the restart of the _cgconfig_ service will overwrite your changes).

**Conclusion:**

By changing the **_cpuset.cpus_ value:**

- We **have been able** to change the cpus affinity dynamically (without database Instance restart).

By changing the **_cpuset.mems_ value:**

- We **have not been able** to change the memory node dynamically (an Instance restart is needed).

**The main goal is reached:** With cpu binding already in place (using _processor\_group\_name_) we are able to change **on the fly the number of cpus a database is limited to use**.


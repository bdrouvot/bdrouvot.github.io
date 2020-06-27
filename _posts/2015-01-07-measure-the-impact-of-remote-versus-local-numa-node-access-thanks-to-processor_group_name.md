---
layout: post
title: Measure the impact of remote versus local NUMA node access thanks to processor_group_name
date: 2015-01-07 14:51:21.000000000 +01:00
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
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/PtR791B4Bmx
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/F6SfZbcucN
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5958590537656205312&type=U&a=CfqE
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/01/07/measure-the-impact-of-remote-versus-local-numa-node-access-thanks-to-processor_group_name/"
---
<p><span style="text-decoration:underline;"><strong>Introduction</strong></span></p>
<p>Starting with oracle database 12c you can use the <em>processor_group_name</em> parameter to instruct the database instance to run itself within the specified operating system processor group. You can find more detail about this feature into Nikolay Manchev's <a href="http://manchev.org/2014/03/processor-group-integration-in-oracle-database-12c" target="_blank">blog post</a>.</p>
<p>So basically, once the <em>libcgroup</em> package has been installed:</p>
<pre style="padding-left:30px;">&gt; yum install libcgroup</pre>
<p>you can create an "oracle" (for example) group by editing the <em>/etc/cgconfig.conf</em> file so that it looks like:</p>
<pre style="padding-left:30px;">&gt; cat /etc/cgconfig.conf 
mount {
        cpuset  = /cgroup/cpuset;
        cpu     = /cgroup/cpu;
        cpuacct = /cgroup/cpuacct;
        memory  = /cgroup/memory;
        devices = /cgroup/devices;
        freezer = /cgroup/freezer;
        net_cls = /cgroup/net_cls;
        blkio   = /cgroup/blkio;
}

group oracle {
   perm {
     task {
       uid = oracle;
       gid = dba;
     }
     admin {
       uid = oracle;
       gid = dba;
     }
   }
   cpuset {
     cpuset.mems="0";
     cpuset.cpus="1-9";
   }
}</pre>
<p>start the <em>cgconfig</em> service:</p>
<pre>&gt; service cgconfig start</pre>
<p>And then set the <em>processor_group_name</em> instance parameter to the group ("oracle" in our example).</p>
<p>The <strong>interesting</strong> part is that to setup the group we have to use 2 <strong>mandatory</strong> parameters into the <em>cpuset</em>:</p>
<ul>
<li><strong><span class="term">cpuset.mems</span></strong>: specifies the <strong>memory</strong> nodes that tasks in this cgroup are permitted to access.</li>
<li><strong><span class="term">cpuset.cpus</span></strong>: specifies the <strong>CPUs</strong> that tasks in this cgroup are permitted to access.</li>
</ul>
<p>So, it means that with <em>processor_group_name</em> set:</p>
<ul>
<li>The <strong>SGA</strong> will be allocated according to the <strong><span class="term">cpuset.mems</span></strong> parameter.</li>
<li>The processes will be bind to the CPUs according to the <strong><span class="term">cpuset.cpus</span></strong> parameter.</li>
</ul>
<p>Then, we can measure the impact of remote versus local NUMA node access thanks to the <em>processor_group_name</em> parameter: Let's do it.</p>
<p><span style="text-decoration:underline;"><strong>Tests setup</strong></span></p>
<p>My NUMA configuration is the following:</p>
<pre style="padding-left:30px;">&gt; numactl --hardware
available: 8 nodes (0-7)
<strong>node 0 cpus: 1 2 3 4 5 6 7 8 9</strong> 10 41 42 43 44 45 46 47 48 49 50
node 0 size: 132864 MB
node 0 free: 45326 MB
node 1 cpus: 11 12 13 14 15 16 17 18 19 20 51 52 53 54 55 56 57 58 59 60
node 1 size: 132864 MB
node 1 free: 46136 MB
node 2 cpus: 21 22 23 24 25 26 27 28 29 30 61 62 63 64 65 66 67 68 69 70
node 2 size: 132864 MB
node 2 free: 40572 MB
node 3 cpus: 31 32 33 34 35 36 37 38 39 40 71 72 73 74 75 76 77 78 79 80
node 3 size: 132864 MB
node 3 free: 47145 MB
node 4 cpus: 0 81 82 83 84 85 86 87 88 89 120 121 122 123 124 125 126 127 128 129
node 4 size: 132836 MB
node 4 free: 40573 MB
node 5 cpus: 90 91 92 93 94 95 96 97 98 99 130 131 132 133 134 135 136 137 138 139
node 5 size: 132864 MB
node 5 free: 45564 MB
node 6 cpus: 100 101 102 103 104 105 106 107 108 109 140 141 142 143 144 145 146 147 148 149
node 6 size: 132864 MB
node 6 free: 20592 MB
node 7 cpus: 110 111 112 113 114 115 116 117 118 119 150 151 152 153 154 155 156 157 158 159
node 7 size: 132864 MB
node 7 free: 44170 MB
node distances:
node   <strong>0</strong>   1   2   3   4   5   6   7 
  0:  10  13  12  12  22  22  22  22 
  1:  13  10  12  12  22  22  22  22 
  2:  12  12  10  13  22  22  22  22 
  3:  12  12  13  10  22  22  22  22 
  4:  22  22  22  22  10  13  12  12 
  5:  22  22  22  22  13  10  12  12 
  6:  22  22  22  22  12  12  10  13 
  <strong>7</strong>:  <strong>22</strong>  22  22  22  12  12  13  10</pre>
<p>As you can see:</p>
<ul>
<li>CPUs 1 to 9 are located into node 0.</li>
<li>node 0 is far from node 7 (distance equals 22).</li>
</ul>
<p><span style="text-decoration:underline;">Then I am able to test <strong>local</strong> NUMA node access by creating the group with:</span></p>
<ul>
<li><em>cpuset.cpus</em>="1-9"</li>
<li><em>cpuset.mems</em>="0"</li>
</ul>
<p>As CPUs 1-9 are located into node 0 and the SGA will be created into node 0.</p>
<p><span style="text-decoration:underline;">and I am able to test <strong>remote</strong> NUMA node access by creating the group with:</span></p>
<ul>
<li><em>cpuset.mems</em>="7"</li>
<li><em>cpuset.cpus</em>="1-9"</li>
</ul>
<p>As CPUs 1-9 are located into node 0 while the SGA will be created into node 7.</p>
<p>To test the effect of remote versus local NUMA node access, I'll use <a href="http://kevinclosson.net/slob/" target="_blank">SLOB</a>. From my point of view this is the perfect/mandatory tool to test the effect of NUMA from the oracle side.</p>
<p>For the tests I created 8 SLOB schemas (with SCALE=10000, so 80 MB), applied some parameters at the instance level for Logical IO testing (as we need FULL cached SLOB to test NUMA efficiently):</p>
<pre style="padding-left:30px;">&gt; cat lio_init.sql
alter system reset "_db_block_prefetch_limit" scope=spfile sid='*';
alter system reset "_db_block_prefetch_quota"  scope=spfile sid='*';
alter system reset "_db_file_noncontig_mblock_read_count" scope=spfile sid='*';
alter system reset "cpu_count" scope=spfile sid='*';
alter system set "db_cache_size"=20g scope=spfile sid='*';
alter system set "shared_pool_size"=10g scope=spfile sid='*';
alter system set "sga_target"=0 scope=spfile sid='*';
</pre>
<p>and <em>set processor_group_name</em> to oracle:</p>
<pre style="padding-left:30px;">SQL&gt; alter system set processor_group_name='oracle' scope=spfile sid='*';
</pre>
<p>For accurate comparison, each test will be based on <strong>the same SLOB configuration and workload</strong>: 8 readers and fix run time of 3 minutes. Then I'll compare how many logical reads have been done in both cases.</p>
<p><span style="text-decoration:underline;"><strong>Test 1: Measure local NUMA node access:</strong></span></p>
<p>As explained previously, I do it by updating <em>/etc/cgconfig.conf</em> file that way:</p>
<ul>
<li><em>cpuset.cpus</em>="1-9"</li>
<li><em>cpuset.mems</em>="0"</li>
</ul>
<p>re-start the service:</p>
<pre>&gt; service cgconfig restart</pre>
<p>and the database instance.</p>
<p><span style="text-decoration:underline;">Check that the SGA is allocated into node 0:</span></p>
<pre style="padding-left:30px;">&gt; grep -i HugePages /sys/devices/system/node/node*/meminfo

/sys/devices/system/node/node0/meminfo:<strong>Node 0 HugePages_Total</strong>: 39303
/sys/devices/system/node/node0/meminfo:<strong>Node 0 HugePages_Free</strong>:  23589
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
<p>As you can see, 39303-23589 large pages have been allocated from node 0. This is the amount of large pages needed for my SGA.</p>
<p>Then I launched SLOB 2 times with the following results:</p>
<pre style="padding-left:30px;">Load Profile                    	Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            		---------------   --------------- --------- ---------
Run 1:  Logical read (blocks):       	4,551,737.9      52,878,393.1
Run 2: 	Logical read (blocks):   	4,685,712.3      18,984,632.1


                                           	Total Wait       Wait   % DB Wait
Event                                	Waits 	Time (sec)    Avg(ms)   time Class
------------------------------    ----------- 	---------- ---------- ------ --------
Run 1: DB CPU                                       1423.2              97.8
Run 2: DB CPU                                       1431.4              99.2</pre>
<p>As you can see, during the 3 minutes run time, SLOB generated about 4 500 000 logical reads per second with <strong>local</strong> NUMA node access in place. Those are also FULL cached SLOB runs as the top event is "DB CPU" with about 100% of the DB time.</p>
<p><span style="text-decoration:underline;"><strong>Test 2: Measure remote NUMA node access:</strong></span></p>
<p>As explained previously, I do it by updating <em>/etc/cgconfig.conf</em> file that way:</p>
<ul>
<li><em>cpuset.cpus</em>="1-9"</li>
<li><em>cpuset.mems</em>="7"</li>
</ul>
<p>re-start the service:</p>
<pre>&gt; service cgconfig restart</pre>
<p>and the database instance.</p>
<p><span style="text-decoration:underline;">Check that the SGA is allocated into node 7:</span></p>
<pre style="padding-left:30px;">&gt; grep -i HugePages /sys/devices/system/node/node*/meminfo
/sys/devices/system/node/node0/meminfo:Node 0 HugePages_Total: 39303
/sys/devices/system/node/node0/meminfo:Node 0 HugePages_Free:  39047
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
/sys/devices/system/node/node7/meminfo:<strong>Node 7 HugePages_Total:</strong> 39302
/sys/devices/system/node/node7/meminfo:<strong>Node 7 HugePages_Free:</strong>  23557
/sys/devices/system/node/node7/meminfo:Node 7 HugePages_Surp:      0
</pre>
<p>As you can see, 39302-23557 large pages have been allocated from node 7. This is the amount of large pages needed for my SGA.</p>
<p>Then I launched SLOB 2 times with the following results:</p>
<pre style="padding-left:30px;">Load Profile                    	Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            		---------------   --------------- --------- ---------
Run 1:  Logical read (blocks):       	2,133,879.6      24,991,464.4
Run 2: 	Logical read (blocks):   	2,203,307.5      10,879,098.7


                                           	Total Wait       Wait   % DB Wait
Event                                	Waits 	Time (sec)    Avg(ms)   time Class
------------------------------    ----------- 	---------- ---------- ------ --------
Run 1: DB CPU 1420.7 97.7 Run 2: DB CPU 1432.6 99.0

As you can see, during the 3 minutes run time, SLOB generated about 2 100 000 logical reads per second with **remote** NUMA node access in place. Those are also FULL cached SLOB runs as the top event is "DB CPU" with about 100% of the DB time.

This is about **2.15 times less** than with the local NUMA node access. This is about the ratio of the distance between node 0 to node 7 and node 0 to node 0 (22/10).

**Remarks:**

- The SGA is fully made of large pages as I set the _use\_large\_pages_ parameter to "only".
- To check that only the CPUs 1 to 9 worked during the SLOB run you can read the mpstat.out file provided by SLOB at the end of the test.
- To check that the oracle session running the SLOB run are bind to CPUs 1 to 9 you can use _taskset_:

```
\> ps -ef | grep 641387 oracle 641387 641353 87 10:15 ? 00:00:55 oracleBDT12CG\_1 (DESCRIPTION=(LOCAL=YES)(ADDRESS=(PROTOCOL=beq))) \> taskset -c -p 641387 pid 641387's current affinity list: 1-9
```

- To check that your SLOB Logical I/O test delivers the maximum, you can read this [post](https://bdrouvot.wordpress.com/2014/04/10/slob-logical-io-testing-check-if-your-benchmark-delivers-the-maximum).
- With SLOB schema of 8 MB, I was not able to see this huge difference between remote and local NUMA node access. I guess this is due to the L3 cache of 24576K on the cpu used during my tests.

**Conclusion:**

- Thanks to the 12c database parameter _processor\_group\_name_, we have been able to measure the impact of remote versus local NUMA node access from the database point of view.
- If you plan the use the _processor\_group\_name_ database parameter, then pay attention to configure the group correctly with _cpuset.cpus_ and _cpuset.mems_ to **ensure local NUMA** node access (My tests show **2.15 times less** logical reads per second with my farest remote access).

**Useful links related to NUMA:**

- [You Buy a NUMA System, Oracle Says Disable NUMA! What Gives? Part I](http://kevinclosson.wordpress.com/2010/03/18/you-buy-a-numa-system-oracle-says-disable-numa-what-gives-part-i/) from Kevin Closson
- [You Buy a NUMA System, Oracle Says Disable NUMA! What Gives? Part II](http://kevinclosson.wordpress.com/2009/05/14/you-buy-a-numa-system-oracle-says-disable-numa-what-gives-part-ii/) from Kevin Closson
- [You Buy a NUMA System, Oracle Says Disable NUMA! What Gives? Part III](http://kevinclosson.wordpress.com/2010/03/31/you-buy-a-numa-system-oracle-says-disable-numa-what-gives-part-iii/) from Kevin Closson
- [Little things I didn’t know: difference between \_enable\_NUMA\_support and numactl](http://martincarstenbach.wordpress.com/2012/04/27/little-things-i-didnt-know-difference-between-_enable_numa_support-and-numactl/)from Martin Bach
- [Linux large pages and non-uniform memory distribution](http://martincarstenbach.wordpress.com/2013/05/17/linux-large-pages-and-non-uniform-memory-distribution/) from Martin Bach

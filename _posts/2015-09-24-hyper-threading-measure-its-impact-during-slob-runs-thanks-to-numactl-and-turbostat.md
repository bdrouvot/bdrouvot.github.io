---
layout: post
title: 'Hyper-threading: measure its impact during SLOB runs thanks to numactl and
  turbostat'
date: 2015-09-24 17:29:36.000000000 +02:00
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
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6052851184337768448&type=U&a=knvF
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/647085498968621057";}}
  _publicize_job_id: '15127592232'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/53a5u8MXKjD
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/09/24/hyper-threading-measure-its-impact-during-slob-runs-thanks-to-numactl-and-turbostat/"
---
<h2>Introduction</h2>
<p>As you probably know,<span class="s1"> a single physical CPU core with hyper-threading appears as two logical CPUs to an operating system. </span></p>
<p class="p1"><span class="s1">While the operating system sees two CPUs for each core, the actual CPU hardware only has a single set of execution resources for each core.</span></p>
<p><span class="s1">Hyper-threading can help speed your system up, but it’s nowhere near as good as having additional cores.</span></p>
<p>Out of curiosity, I checked the impact of hyper-threading on the number of logical IO per second (LIOPS) during <a href="http://kevinclosson.net/slob/" target="_blank">SLOB</a> runs. The main idea is to compare:</p>
<ul>
<li>2 FULL CACHED SLOB sessions running on the same core</li>
<li>2 FULL CACHED SLOB sessions running on two distincts cores</li>
</ul>
<p>Into this blog post, I just want to share how I did the measure and my results (of course your results could differ depending of the cpus, the slob configuration and so on..).</p>
<h2>Tests environment and setup</h2>
<ul>
<li>Oracle database 11.2.0.4.</li>
<li>SLOB with the following slob.conf in place (the rest of the file contains the default slob.conf values):</li>
</ul>
<pre style="padding-left:30px;">UPDATE_PCT=0
RUN_TIME=60
WORK_LOOP=0
SCALE=80M
WORK_UNIT=64
</pre>
<ul>
<li>The cpu_topology is the following (the <em>cpu_topology</em> script comes from Karl Arao’s cputoolkit <a href="https://karlarao.wordpress.com/scripts-resources" target="_blank">here</a>):</li>
</ul>
<pre>$ sh ./cpu_topology
	
model name	: Intel(R) Xeon(R) CPU E5-2690 0 @ 2.90GHz
processors  (OS CPU count)          0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
physical id (processor socket)      0 0 0 0 0 0 0 0 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 1 1 1 1 1 1 1 1
siblings    (logical CPUs/socket)   16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16 16
core id     (# assigned to a core)  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
cpu cores   (physical cores/socket) 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8 8

$ lscpu

Architecture: x86_64
CPU op-mode(s): 32-bit, 64-bit
Byte Order: Little Endian
CPU(s): 32
On-line CPU(s) list: 0-31
Thread(s) per core: 2
Core(s) per socket: 8
Socket(s): 2
NUMA node(s): 1
Vendor ID: GenuineIntel
CPU family: 6
Model: 45
Stepping: 7
CPU MHz: 2893.187
BogoMIPS: 5785.23
Virtualization: VT-x
L1d cache: 32K
L1i cache: 32K
L2 cache: 256K
L3 cache: 20480K
</pre>
<p>As you can see the box is made of 2 sockets, 16 cores and 32 threads (so that the database sees 32 cpus).</p>
<ul>
<li><a href="https://en.wikipedia.org/wiki/Intel_Turbo_Boost" target="_blank">Turbo boost</a> does not come into play (you'll see the proof during the tests thanks to the <em>turbostat</em> command).</li>
<li>Obviously the same SLOB configuration has been used for the tests.</li>
</ul>
<h2>Launch the tests</h2>
<h3>Test 1: launch 2 SLOB sessions on 2 distincts sockets (so on 2 distincts cores) that way:</h3>
<pre>$ cd SLOB
$ numactl --physcpubind=4,9 /bin/bash 
$ echo "startup" | sqlplus / as sysdba
$ ./runit.sh 2</pre>
<p>Cpus 4 and 9 are on 2 distincts sockets (see <em>cpu_topology</em> output).</p>
<p>The <em>turbsostat</em> command shows that they are 100% busy during the run:</p>
<pre>Package    Core     CPU Avg_MHz   %Busy Bzy_MHz TSC_MHz     SMI  CPU%c1  CPU%c3  CPU%c6  CPU%c7 CoreTmp  PkgTmp Pkg%pc2 Pkg%pc3 Pkg%pc6 Pkg%pc7 PkgWatt CorWatt RAMWatt   PKG_%   RAM_%
       -       -       -     241    8.34    2893    2893       0   91.66    0.00    0.00    0.00      76      76    0.00    0.00    0.00    0.00  116.66   88.59    0.00    0.00    0.00
       0       0       0      86    2.98    2893    2893       0   97.02    0.00    0.00    0.00      46      48    0.00    0.00    0.00    0.00   50.44   36.66    0.00    0.00    0.00
       0       0      16      70    2.41    2893    2893       0   97.59
       0       1       1      71    2.45    2893    2893       0   97.55    0.00    0.00    0.00      43
       0       1      17     131    4.54    2893    2893       0   95.46
       0       2       2      81    2.81    2893    2893       0   97.19    0.00    0.00    0.00      44
       0       2      18      73    2.53    2893    2893       0   97.47
       0       3       3      29    1.00    2893    2893       0   99.00    0.00    0.00    0.00      44
       0       3      19      21    0.72    2893    2893       0   99.28
       <strong>0       4       4    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      46</strong>
       0       4      20      40    1.39    2893    2893       0   98.61
       0       5       5      58    2.02    2893    2893       0   97.98    0.00    0.00    0.00      46
       0       5      21      82    2.82    2893    2893       0   97.18
       0       6       6     106    3.65    2893    2893       0   96.35    0.00    0.00    0.00      44
       0       6      22      42    1.46    2893    2893       0   98.54
       0       7       7      28    0.97    2893    2893       0   99.03    0.00    0.00    0.00      48
       0       7      23      15    0.52    2893    2893       0   99.48
       1       0       8     222    7.67    2893    2893       0   92.33    0.00    0.00    0.00      72      76    0.00    0.00    0.00    0.00   66.22   51.92    0.00    0.00    0.00
       1       0      24      63    2.17    2893    2893       0   97.83
      <strong> 1       1       9    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      76</strong>
       1       1      25     108    3.73    2893    2893       0   96.27
       1       2      10      86    2.98    2893    2893       0   97.02    0.00    0.00    0.00      71
       1       2      26      78    2.71    2893    2893       0   97.29
       1       3      11      57    1.96    2893    2893       0   98.04    0.00    0.00    0.00      69
       1       3      27      41    1.40    2893    2893       0   98.60
       1       4      12      56    1.93    2893    2893       0   98.07    0.00    0.00    0.00      71
       1       4      28      30    1.03    2893    2893       0   98.97
       1       5      13      32    1.12    2893    2893       0   98.88    0.00    0.00    0.00      71
       1       5      29      23    0.79    2893    2893       0   99.21
       1       6      14      49    1.68    2893    2893       0   98.32    0.00    0.00    0.00      71
       1       6      30      93    3.20    2893    2893       0   96.80
       1       7      15      41    1.41    2893    2893       0   98.59    0.00    0.00    0.00      72
       1       7      31      21    0.72    2893    2893       0   99.28</pre>
<p>Turbo boost did not play  (Bzy_MHz = TSC_MHz).</p>
<p>Those 2 sessions produced about <strong>955 000 LIOPS</strong>:</p>
<pre>Load Profile                    Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            ---------------   --------------- --------- ---------
             DB Time(s):               2.0               5.8      0.00      2.95
              DB CPU(s):               1.9               5.6      0.00      2.89
      Redo size (bytes):          16,541.5          48,682.3
  Logical read (blocks):         955,750.1       2,812,818.1
</pre>
<h3>Test 2: launch 2 SLOB sessions on 2 distincts cores on the same socket that way:</h3>
<pre>$ cd SLOB
$ numactl --physcpubind=4,5 /bin/bash 
$ echo "startup" | sqlplus / as sysdba
$ ./runit.sh 2</pre>
<p>Cpus 4 and 5 are on 2 distincts cores on the same socket (see <em>cpu_topology</em> output).</p>
<p>The <em>turbsostat</em> command shows that they are 100% busy during the run:</p>
<pre>Package    Core     CPU Avg_MHz   %Busy Bzy_MHz TSC_MHz     SMI  CPU%c1  CPU%c3  CPU%c6  CPU%c7 CoreTmp  PkgTmp Pkg%pc2 Pkg%pc3 Pkg%pc6 Pkg%pc7 PkgWatt CorWatt RAMWatt   PKG_%   RAM_%
       -       -       -     270    9.34    2893    2893       0   90.66    0.00    0.00    0.00      76      76    0.00    0.00    0.00    0.00  119.34   91.17    0.00    0.00    0.00
       0       0       0     152    5.26    2893    2893       0   94.74    0.00    0.00    0.00      47      49    0.00    0.00    0.00    0.00   55.29   41.51    0.00    0.00    0.00
       0       0      16      59    2.04    2893    2893       0   97.96
       0       1       1      62    2.13    2893    2893       0   97.87    0.00    0.00    0.00      42
       0       1      17      49    1.70    2893    2893       0   98.30
       0       2       2      59    2.03    2893    2893       0   97.97    0.00    0.00    0.00      43
       0       2      18      55    1.90    2893    2893       0   98.10
       0       3       3      10    0.33    2893    2893       0   99.67    0.00    0.00    0.00      44
       0       3      19      16    0.56    2893    2893       0   99.44
       <strong>0       4       4    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      46</strong>
       0       4      20      40    1.40    2893    2893       0   98.60
       <strong>0       5       5    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      49</strong>
       0       5      21     133    4.59    2893    2893       0   95.41
       0       6       6      71    2.47    2893    2893       0   97.53    0.00    0.00    0.00      45
       0       6      22      57    1.97    2893    2893       0   98.03
       0       7       7      20    0.70    2893    2893       0   99.30    0.00    0.00    0.00      47
       0       7      23      14    0.47    2893    2893       0   99.53
       1       0       8     364   12.58    2893    2893       0   87.42    0.00    0.00    0.00      76      76    0.00    0.00    0.00    0.00   64.05   49.66    0.00    0.00    0.00
       1       0      24      93    3.20    2893    2893       0   96.80
       1       1       9     270    9.34    2893    2893       0   90.66    0.00    0.00    0.00      76
       1       1      25     105    3.62    2893    2893       0   96.38
       1       2      10     267    9.23    2893    2893       0   90.77    0.00    0.00    0.00      75
       1       2      26     283    9.77    2893    2893       0   90.23
       1       3      11      73    2.52    2893    2893       0   97.48    0.00    0.00    0.00      73
       1       3      27      92    3.19    2893    2893       0   96.81
       1       4      12      43    1.50    2893    2893       0   98.50    0.00    0.00    0.00      75
       1       4      28      44    1.53    2893    2893       0   98.47
       1       5      13      46    1.60    2893    2893       0   98.40    0.00    0.00    0.00      75
       1       5      29      48    1.67    2893    2893       0   98.33
       1       6      14      76    2.63    2893    2893       0   97.37    0.00    0.00    0.00      75
       1       6      30     105    3.64    2893    2893       0   96.36
       1       7      15     106    3.67    2893    2893       0   96.33    0.00    0.00    0.00      75
</pre>
<p>Turbo boost did not play  (Bzy_MHz = TSC_MHz).</p>
<p>Those 2 sessions produced about <strong>920 000 LIOPS</strong>:</p>
<pre>Load Profile                    Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            ---------------   --------------- --------- ---------
             DB Time(s):               2.0               5.8      0.00      3.02
              DB CPU(s):               1.9               5.7      0.00      2.97
      Redo size (bytes):          16,014.6          47,163.1
  Logical read (blocks):         921,446.7       2,713,660.6</pre>
<p>So, let's say this is more or less the same as with 2 sessions on 2 distincts sockets.</p>
<h3>Test 3: launch 2 SLOB sessions on the same core that way:</h3>
<pre>$ cd SLOB
$ numactl --physcpubind=4,20 /bin/bash 
$ echo "startup" | sqlplus / as sysdba
$ ./runit.sh 2</pre>
<p>Cpus 4 and 20 are sharing the same core (see <em>cpu_topology</em> output).</p>
<p>The <em>turbsostat</em> command shows that they are 100% busy during the run:</p>
<pre>Package    Core     CPU Avg_MHz   %Busy Bzy_MHz TSC_MHz     SMI  CPU%c1  CPU%c3  CPU%c6  CPU%c7 CoreTmp  PkgTmp Pkg%pc2 Pkg%pc3 Pkg%pc6 Pkg%pc7 PkgWatt CorWatt RAMWatt   PKG_%   RAM_%
       -       -       -     250    8.65    2893    2893       0   91.35    0.00    0.00    0.00      76      76    0.00    0.00    0.00    0.00  114.06   85.95    0.00    0.00    0.00
       0       0       0     100    3.45    2893    2893       0   96.55    0.00    0.00    0.00      48      48    0.00    0.00    0.00    0.00   51.21   37.45    0.00    0.00    0.00
       0       0      16      41    1.41    2893    2893       0   98.59
       0       1       1      75    2.61    2893    2893       0   97.39    0.00    0.00    0.00      42
       0       1      17      58    2.01    2893    2893       0   97.99
       0       2       2      51    1.75    2893    2893       0   98.25    0.00    0.00    0.00      44
       0       2      18      40    1.38    2893    2893       0   98.62
       0       3       3      25    0.86    2893    2893       0   99.14    0.00    0.00    0.00      45
       0       3      19       9    0.30    2893    2893       0   99.70
       <strong>0       4       4    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      47
       0       4      20    2893  100.00    2893    2893       0    0.00</strong>
       0       5       5     120    4.14    2893    2893       0   95.86    0.00    0.00    0.00      44
       0       5      21      73    2.54    2893    2893       0   97.46
       0       6       6      65    2.25    2893    2893       0   97.75    0.00    0.00    0.00      45
       0       6      22      51    1.76    2893    2893       0   98.24
       0       7       7      18    0.64    2893    2893       0   99.36    0.00    0.00    0.00      48
       0       7      23       7    0.25    2893    2893       0   99.75
       1       0       8     491   16.97    2893    2893       0   83.03    0.00    0.00    0.00      76      76    0.00    0.00    0.00    0.00   62.85   48.50    0.00    0.00    0.00
       1       0      24      88    3.04    2893    2893       0   96.96
       1       1       9      91    3.16    2893    2893       0   96.84    0.00    0.00    0.00      75
       1       1      25      80    2.75    2893    2893       0   97.25
       1       2      10      87    3.00    2893    2893       0   97.00    0.00    0.00    0.00      74
       1       2      26     104    3.61    2893    2893       0   96.39
       1       3      11      62    2.16    2893    2893       0   97.84    0.00    0.00    0.00      72
       1       3      27      80    2.77    2893    2893       0   97.23
       1       4      12      36    1.24    2893    2893       0   98.76    0.00    0.00    0.00      74
       1       4      28      36    1.25    2893    2893       0   98.75
       1       5      13      47    1.62    2893    2893       0   98.38    0.00    0.00    0.00      75
       1       5      29      75    2.61    2893    2893       0   97.39
       1       6      14      52    1.79    2893    2893       0   98.21    0.00    0.00    0.00      75
       1       6      30      92    3.17    2893    2893       0   96.83
       1       7      15      39    1.35    2893    2893       0   98.65    0.00    0.00    0.00      76
       1       7      31      32    1.11    2893    2893       0   98.89</pre>
<p>Turbo boost did not play  (Bzy_MHz = TSC_MHz).</p>
<p>Those 2 sessions produced about <strong>600 000 LIOPS</strong>:</p>
<pre>Load Profile                    Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            ---------------   --------------- --------- ---------
             DB Time(s):               2.0               5.8      0.00      3.02
              DB CPU(s):               1.9               5.6      0.00      2.96
      Redo size (bytes):          16,204.0          47,101.1
  Logical read (blocks):         603,430.3       1,754,028.2</pre>
<p>So, as you can see, the SLOB runs (with the same SLOB configuration) produced about:</p>
<ul>
<li>955 000 LIOPS with the 2 sessions running on 2 distincts sockets (so on 2 distincts cores).</li>
<li>920 000 LIOPS with the 2 sessions running on 2 distincts cores on the same socket.</li>
<li>600 000 LIOPS with the 2 sessions running on the same core.</li>
</ul>
<h3>Out of curiosity, what if we disable the hyper-threading on one core and launch the 2 sessions on it?</h3>
<p>Let's put the cpu 20 offline (to be sure there is no noise on it during the test)</p>
<pre>echo 0 &gt; /sys/devices/system/cpu/cpu20/online</pre>
<p>launch the 2 sessions on the same cpu (the remaining one online for this core as the second one has just been put offline) that way:</p>
<pre>$ cd SLOB
$ numactl --physcpubind=4 /bin/bash 
$ echo "startup" | sqlplus / as sysdba
$ ./runit.sh 2</pre>
<p>The <em>turbsostat</em> command shows that cpu 4 is 100% busy during the run:</p>
<pre>Package    Core     CPU Avg_MHz   %Busy Bzy_MHz TSC_MHz     SMI  CPU%c1  CPU%c3  CPU%c6  CPU%c7 CoreTmp  PkgTmp Pkg%pc2 Pkg%pc3 Pkg%pc6 Pkg%pc7 PkgWatt CorWatt RAMWatt   PKG_%   RAM_%
       -       -       -     153    5.30    2893    2893       0   94.70    0.00    0.00    0.00      77      77    0.00    0.00    0.00    0.00  112.76   84.64    0.00    0.00    0.00
       0       0       0      56    1.94    2893    2893       0   98.06    0.00    0.00    0.00      45      46    0.00    0.00    0.00    0.00   49.86   36.07    0.00    0.00    0.00
       0       0      16      42    1.45    2893    2893       0   98.55
       0       1       1      97    3.35    2893    2893       0   96.65    0.00    0.00    0.00      42
       0       1      17      61    2.11    2893    2893       0   97.89
       0       2       2      47    1.62    2893    2893       0   98.38    0.00    0.00    0.00      44
       0       2      18      45    1.54    2893    2893       0   98.46
       0       3       3      19    0.64    2893    2893       0   99.36    0.00    0.00    0.00      44
       0       3      19      25    0.85    2893    2893       0   99.15
       <strong>0       4       4    2893  100.00    2893    2893       0    0.00    0.00    0.00    0.00      46</strong>
       0       5       5       9    0.31    2893    2893       0   99.69    0.00    0.00    0.00      44
       0       5      21      14    0.49    2893    2893       0   99.51
       0       6       6      14    0.48    2893    2893       0   99.52    0.00    0.00    0.00      44
       0       6      22      12    0.41    2893    2893       0   99.59
       0       7       7      25    0.88    2893    2893       0   99.12    0.00    0.00    0.00      46
       0       7      23       9    0.33    2893    2893       0   99.67
       1       0       8     133    4.59    2893    2893       0   95.41    0.00    0.00    0.00      76      77    0.00    0.00    0.00    0.00   62.90   48.56    0.00    0.00    0.00
       1       0      24      69    2.37    2893    2893       0   97.63
       1       1       9      89    3.07    2893    2893       0   96.93    0.00    0.00    0.00      76
       1       1      25      73    2.52    2893    2893       0   97.48
       1       2      10      98    3.38    2893    2893       0   96.62    0.00    0.00    0.00      76
       1       2      26     113    3.92    2893    2893       0   96.08
       1       3      11      76    2.62    2893    2893       0   97.38    0.00    0.00    0.00      74
       1       3      27      78    2.71    2893    2893       0   97.29
       1       4      12     194    6.71    2893    2893       0   93.29    0.00    0.00    0.00      75
       1       4      28      27    0.94    2893    2893       0   99.06
       1       5      13      62    2.14    2893    2893       0   97.86    0.00    0.00    0.00      76
       1       5      29      95    3.29    2893    2893       0   96.71
       1       6      14     101    3.48    2893    2893       0   96.52    0.00    0.00    0.00      75
       1       6      30      92    3.19    2893    2893       0   96.81
       1       7      15      64    2.20    2893    2893       0   97.80    0.00    0.00    0.00      77
       1       7      31      20    0.68    2893    2893       0   99.32</pre>
<p>and as you can see it is the only cpu available on that core.</p>
<p>Turbo boost did not play  (Bzy_MHz = TSC_MHz).</p>
<p>Those 2 sessions produced about <strong>460 000 LIOPS</strong>:</p>
<pre style="padding-left:30px;">Load Profile                    Per Second   Per Transaction  Per Exec  Per Call
~~~~~~~~~~~~~~~            ---------------   --------------- --------- ---------
DB Time(s): 2.0 5.8 0.00 2.69 DB CPU(s): 1.0 2.8 0.00 1.32 Redo size (bytes): 18,978.1 55,154.9 Logical read (blocks): 458,451.7 1,332,369.7

which makes fully sense as 2 sessions on 2 distincts cores on the same sockets produced about 920 000 LIOPS (Test 2).

## So what?

Imagine that you launched multiple full cached&nbsp;[SLOB](http://kevinclosson.net/slob/) runs on your new box to measure its behaviour/scalability (from one session up to the _cpu\_count_ value).

Example: I launched 11 full cached SLOB runs with 2,4,6,8,12,16,18,20,24,28 and 32 sessions running. The LIOPS graph is the following:

[![Screen Shot 2015-09-22 at 09.29.08]({{ site.baseurl }}/assets/images/screen-shot-2015-09-22-at-09-29-08.png)](https://bdrouvot.files.wordpress.com/2015/09/screen-shot-2015-09-22-at-09-29-08.png)

Just check the curve of the graph (not the values). As you can see, once the number of sessions reached the number of cores (16) the LIOPS is still increasing but its increase rate is slow down. One of the reason is the one we observed during our previous tests: 2 sessions running on the same core don't perform as fast as 2 sessions running on 2 distincts cores.

## Remarks

Worth watching presentation around oracle and cpu stuff:

- Karl Arao: [Where did my cpu go](https://www.youtube.com/watch?v=WXktSUbE4AU)
- Kevin Closson: [Modern platform topics for modern DBAs](https://www.youtube.com/watch?v=S8Ih1NpOlNI&feature=youtu.be)
- Kevin Closson:&nbsp;[SLOB – For More Than I/O!](https://www.youtube.com/watch?v=FMuON94f4O8&feature=youtu.be)

The absolute LIOPS numbers are relatively low. It would have been possible to increase those numbers&nbsp;during the tests by increasing the _WORK\_UNIT_ slob parameter (see this [blog post](https://bdrouvot.wordpress.com/2014/04/10/slob-logical-io-testing-check-if-your-benchmark-delivers-the-maximum/)).

To get the turbostat utility, copy/paste the [turbostat.c](http://lxr.free-electrons.com/source/tools/power/x86/turbostat/turbostat.c) and&nbsp;[msr-index.h](http://lxr.free-electrons.com/source/arch/x86/include/asm/msr-index.h)&nbsp;contents into a directory and compile turbostat.c that way:

```
$ gcc -Wall -DMSRHEADER='"./msr-index.h"' turbostat.c -o turbostat
```

## Conclusion

During the&nbsp;test I observed the following (of course your results could differ depending of the cpus, the slob configuration and so on..):

- 955 000 LIOPS with the 2 sessions running on 2 distincts sockets (so on 2 distincts cores)
- 920 000 LIOPS&nbsp;with the 2 sessions running on 2 distincts cores&nbsp;on the same socket
- 600 000 LIOPS with the 2 sessions running on the same core
- 460 000 LIOPS with the 2 sessions running on the same core with hyper-threading disabled

Hyper-threading helps to increase the overall LIOPS throughput of the system, but once the numbers of sessions is \> number of cores then the average LIOPS per session decrease.


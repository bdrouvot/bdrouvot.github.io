---
layout: post
title: Graphing ASM performance metrics
date: 2014-07-08 20:22:35.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Tableau
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/eHDThRuzdwt
  _wpas_done_5547632: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"101126738655139704850";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/bExh5OqxaX
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5892356917539401728&type=U&a=yMyy
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/07/08/graphing-asm-performance-metrics/"
---
<p>ASM metrics are a goldmine, they provide a lot of informations. As you may know, the <a title="ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time" href="http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/" target="_blank">asm_metrics utility</a> extracts them in real-time.</p>
<p>But sometimes it is not easy to understand the values without the help of a graph. Look at this example: <a href="http://www.oraclerealworld.com/if-i-cant-picture-it-i-cant-understand-it/" target="_blank">If I cant' picture it, I can't understand it</a>.</p>
<p>So depending on your needs, depending on what you are looking for with the ASM metrics: A picture may help.</p>
<p><strong>So let's graph the output of the <em>asm_metrics</em> utility:</strong> For this I created the <em><strong>csv_asm_metrics</strong></em> utility to produce a csv file from the output of the <em>asm_metrics</em> utility.</p>
<p>Once you get the csv file you can graph the metrics with your favourite visualization tool (I'll use <a href="http://www.tableausoftware.com/public//" target="_blank">Tableau </a>as an example).</p>
<p>First you have to launch the <em>asm_metrics</em> utility that way (To ensure that <strong>all the fields are displayed</strong>):</p>
<ul>
<li><em>-show=inst,dbinst,fg,dg,dsk</em> for ASM &gt;= 11g</li>
<li><em>-show=inst,fg,dg,dsk</em> for ASM &lt; 11g</li>
</ul>
<p>and redirect the output to a text file:</p>
<pre style="padding-left:30px;">./asm_metrics.pl -show=inst,dbinst,fg,dg,dsk &gt; asm_metrics.txt</pre>
<p><span style="text-decoration:underline;">Remark:</span> You can use the <em>-interval</em> parameter to collect data with an interval greater than one second (the default interval), as it could produce a huge output file.</p>
<p>The output file looks like:</p>
<pre style="padding-left:30px;">............................
Collecting 1 sec....
............................

......... SNAP TAKEN AT ...................

13:48:54                                                                              Kby       Avg       AvgBy/               Kby       Avg        AvgBy/ 
13:48:54   INST     DBINST        DG            FG           DSK            Reads/s   Read/s    ms/Read   Read      Writes/s   Write/s   ms/Write   Write  
13:48:54   ------   -----------   -----------   ----------   ----------     -------   -------   -------   ------    ------     -------   --------   ------ 
13:48:54   +ASM1                                                            6731      54224     1.4       8249      42         579       3.0        14117  
13:48:54   +ASM1    BDT10_1                                                 2         32        0.2       16384     0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA                                      2         32        0.2       16384     0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST31                      0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST31       HOST31CA0D1C   0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST31       HOST31CA0D1D   0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST32                      2         32        0.2       16384     0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST32       HOST32CA0D1C   0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       DATA          HOST32       HOST32CA0D1D   2         32        0.2       16384     0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       FRA                                       0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       FRA           HOST31                      0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       FRA           HOST31       HOST31CC8D0F   0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       FRA           HOST32                      0         0         0.0       0         0          0         0.0        0      
13:48:54   +ASM1    BDT10_1       FRA           HOST32       HOST32CC8D0F   0         0         0.0       0         0          0         0.0        0      

and so on...</pre>
<p>Now let's produce the csv file with the <em>csv_asm_metrics</em> utility. Let's see the help:</p>
<pre style="padding-left:30px;">./csv_asm_metrics.pl -help

Usage: ./csv_asm_metrics.pl [-if] [-of] [-d] [-help]

  Parameter         Comment
  ---------         -------
-if= Input file name (output of asm\_metrics) -of= Output file name (the csv file) -d= Day of the first snapshot (YYYY/MM/DD) Example: ./csv\_asm\_metrics.pl -if=asm\_metrics.txt -of=asm\_metrics.csv -d='2014/07/04'

and generate the csv file that way:

```
./csv\_asm\_metrics.pl -if=asm\_metrics.txt -of=asm\_metrics.csv -d='2014/07/04'
```

The csv file looks like:

```
Snap Time,INST,DBINST,DG,FG,DSK,Reads/s,Kby Read/s,ms/Read,By/Read,Writes/s,Kby Write/s,ms/Write,By/Write 2014/07/04 13:48:54,+ASM1,BDT10\_1,DATA,HOST31,HOST31CA0D1C,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,DATA,HOST31,HOST31CA0D1D,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,DATA,HOST32,HOST32CA0D1C,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,DATA,HOST32,HOST32CA0D1D,2,32,0.2,16384,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,FRA,HOST31,HOST31CC8D0F,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,FRA,HOST32,HOST32CC8D0F,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,REDO1,HOST31,HOST31CC0D13,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,REDO1,HOST32,HOST32CC0D13,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,REDO2,HOST31,HOST31CC0D12,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT10\_1,REDO2,HOST32,HOST32CC0D12,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT11\_1,DATA,HOST31,HOST31CA0D1C,0,0,0.0,0,0,0,0.0,0 2014/07/04 13:48:54,+ASM1,BDT11\_1,DATA,HOST31,HOST31CA0D1D,0,0,0.0,0,2,16,0.5,8448
```

**As you can see:**

1. The day has been added (to create a date) and next ones will be calculated (should the snaps cross multiple days).
2. Only the rows that contain all the fields have been recorded into the csv file (The script does not record the other ones as they represent aggregated values).

Now I can import this csv file into Tableau.

You can imagine **a lot of graphs** thanks to the measures collected (_Reads/s, Kby Read/s, ms/Read, By/Read, Writes/s, Kby Write/s, ms/Write, By/Write_) and&nbsp;all those dimensions (_Snap Time, INST, DBINST, DG, FG, DSK_).

Let's graph the throughput and latency per failgroup for example.

**Important remark regarding some averages computation/display:**

The _ms/Read_ and _By/Read_ measures **depend on the number of reads**. So the averages have to be calculated using **Weighted Averages**. (The same apply for _ms/Write_ and _By/Write_).

Let's create the calculated field in Tableau for those Weighted Averages:

[![Screen Shot 2014-07-07 at 20.13.28]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-13-28.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-13-28.png)

so that weighted Average ms/Read is:

[![Screen Shot 2014-07-07 at 20.16.19]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-16-19.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-16-19.png)

Weighted Average By/Read:

[![Screen Shot 2014-07-07 at 20.21.12]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-21-12.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-21-12.png)

The same way you have to create:

- Weighted Average ms/Write =&nbsp;sum([ms/Write]\*[Writes/s])/sum([Writes/s])
- Weighted Average By/Write =&nbsp;sum([By/Write]\*[Writes/s])/sum([Writes/s])

Now let's display the average read latency by Failgroup (using the previous calculated weighted average):

Drag the _Snap Time_ dimension to the "columns" shelf and choose "exact date":

[![Screen Shot 2014-07-07 at 20.27.23]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-27-23.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-27-23.png)Drag the _Weighted Average ms/Read_ calculated field to the "Rows" shelf:

[![Screen Shot 2014-07-07 at 20.29.41]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-29-41.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-29-41.png)Drag the _FG_ dimension to the "Color Marks" shelf:

[![Screen Shot 2014-07-07 at 20.32.33]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-32-33.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-32-33.png)So that the graph looks like:

[![Screen Shot 2014-07-07 at 20.33.15]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-33-15.png)](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-33-15.png)Create the same graph for the "Kby Read/s" measure (except that I want to see the sum (i.e the throughput and not the average) and put those 2 graphs into the same dashboard:

[![Screen Shot 2014-07-07 at 20.39.42]({{ site.baseurl }}/assets/images/screen-shot-2014-07-07-at-20-39-42.png)](http://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-39-42.png)

Here we are.

**Conclusion:**

- We can create a csv file from the output of the _asm\_metrics_ utility thanks to _csv\_asm\_metrics_.

- To do so, we have to collect all the fields of _asm\_metrics_ with those options:

  - _-show=inst,dbinst,fg,dg,dsk_ for ASM \>= 11g
  - _-show=inst,fg,dg,dsk_ for ASM \< 11g

- Once you uploaded the csv file into your favourite visualization tool, don't forget to calculate **weighted averages** for _ms/Read, By/Read, ms/Write_ and _By/Write_ if you plan to graph the averages.

- You can imagine&nbsp; **a lot of graphs** &nbsp;thanks to the measures collected (_Reads/s, Kby Read/s, ms/Read, By/Read, Writes/s, Kby Write/s, ms/Write, By/Write_) and all those dimensions (_Snap Time, INST, DBINST, DG, FG, DSK_).

You can **download** the _csv\_asm\_metrics_ utility from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](http://bdrouvot.wordpress.com/csv_asm_metrics_source/ "csv\_asm\_metrics\_source").

**UPDATE** : You can see some use cases [here](http://bdrouvot.wordpress.com/2014/07/12/asm-performance-metrics-visualization-use-cases/ "ASM performance metrics visualization: Use cases").


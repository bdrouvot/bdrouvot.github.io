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

ASM metrics are a goldmine, they provide a lot of informations. As you may know, the [asm\_metrics utility](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time") extracts them in real-time.

But sometimes it is not easy to understand the values without the help of a graph. Look at this example: [If I cant' picture it, I can't understand it](http://www.oraclerealworld.com/if-i-cant-picture-it-i-cant-understand-it/).

So depending on your needs, depending on what you are looking for with the ASM metrics: A picture may help.

**So let's graph the output of the *asm\_metrics* utility:** For this I created the ***csv\_asm\_metrics*** utility to produce a csv file from the output of the *asm\_metrics* utility.

Once you get the csv file you can graph the metrics with your favourite visualization tool (I'll use [Tableau](http://www.tableausoftware.com/public//) as an example).

First you have to launch the *asm\_metrics* utility that way (To ensure that **all the fields are displayed**):

-   *-show=inst,dbinst,fg,dg,dsk* for ASM &gt;= 11g
-   *-show=inst,fg,dg,dsk* for ASM &lt; 11g

and redirect the output to a text file:

    ./asm_metrics.pl -show=inst,dbinst,fg,dg,dsk > asm_metrics.txt

<span style="text-decoration:underline;">Remark:</span> You can use the *-interval* parameter to collect data with an interval greater than one second (the default interval), as it could produce a huge output file.

The output file looks like:

    ............................
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

    and so on...

Now let's produce the csv file with the *csv\_asm\_metrics* utility. Let's see the help:

    ./csv_asm_metrics.pl -help

    Usage: ./csv_asm_metrics.pl [-if] [-of] [-d] [-help]

      Parameter         Comment
      ---------         -------
      -if=              Input file name (output of asm_metrics)
      -of=              Output file name (the csv file)
      -d=               Day of the first snapshot (YYYY/MM/DD)

    Example: ./csv_asm_metrics.pl -if=asm_metrics.txt -of=asm_metrics.csv -d='2014/07/04'

and generate the csv file that way:

    ./csv_asm_metrics.pl -if=asm_metrics.txt -of=asm_metrics.csv -d='2014/07/04'

The csv file looks like:

    Snap Time,INST,DBINST,DG,FG,DSK,Reads/s,Kby Read/s,ms/Read,By/Read,Writes/s,Kby Write/s,ms/Write,By/Write
    2014/07/04 13:48:54,+ASM1,BDT10_1,DATA,HOST31,HOST31CA0D1C,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,DATA,HOST31,HOST31CA0D1D,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,DATA,HOST32,HOST32CA0D1C,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,DATA,HOST32,HOST32CA0D1D,2,32,0.2,16384,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,FRA,HOST31,HOST31CC8D0F,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,FRA,HOST32,HOST32CC8D0F,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,REDO1,HOST31,HOST31CC0D13,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,REDO1,HOST32,HOST32CC0D13,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,REDO2,HOST31,HOST31CC0D12,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT10_1,REDO2,HOST32,HOST32CC0D12,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT11_1,DATA,HOST31,HOST31CA0D1C,0,0,0.0,0,0,0,0.0,0
    2014/07/04 13:48:54,+ASM1,BDT11_1,DATA,HOST31,HOST31CA0D1D,0,0,0.0,0,2,16,0.5,8448

<span style="text-decoration:underline;">**As you can see:**</span>

1.  The day has been added (to create a date) and next ones will be calculated (should the snaps cross multiple days).
2.  Only the rows that contain all the fields have been recorded into the csv file (The script does not record the other ones as they represent aggregated values).

Now I can import this csv file into Tableau.

You can imagine **a lot of graphs** thanks to the measures collected (*Reads/s, Kby Read/s, ms/Read, By/Read, Writes/s, Kby Write/s, ms/Write, By/Write*) and all those dimensions (*Snap Time, INST, DBINST, DG, FG, DSK*).

Let's graph the throughput and latency per failgroup for example.

<span style="text-decoration:underline;">**Important remark regarding some averages computation/display:**</span>

The *ms/Read* and *By/Read* measures **depend on the number of reads**. So the averages have to be calculated using **Weighted Averages**. (The same apply for *ms/Write* and *By/Write*).

Let's create the calculated field in Tableau for those Weighted Averages:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-13-28.png" class="aligncenter size-full wp-image-2024" width="453" height="575" alt="Screen Shot 2014-07-07 at 20.13.28" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-13-28.png)

<span style="text-decoration:underline;">so that weighted Average ms/Read is:</span>

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-16-19.png" class="aligncenter size-full wp-image-2025" width="640" height="234" alt="Screen Shot 2014-07-07 at 20.16.19" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-16-19.png)

<span style="text-decoration:underline;">Weighted Average By/Read:</span>

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-21-12.png" class="aligncenter size-full wp-image-2026" width="640" height="230" alt="Screen Shot 2014-07-07 at 20.21.12" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-21-12.png)

<span style="text-decoration:underline;">The same way you have to create:</span>

-   Weighted Average ms/Write = sum(\[ms/Write\]\*\[Writes/s\])/sum(\[Writes/s\])
-   Weighted Average By/Write = sum(\[By/Write\]\*\[Writes/s\])/sum(\[Writes/s\])

Now let's display the average read latency by Failgroup (using the previous calculated weighted average):

Drag the *Snap Time* dimension to the "columns" shelf and choose "exact date":

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-27-23.png" class="aligncenter size-full wp-image-2027" width="343" height="591" alt="Screen Shot 2014-07-07 at 20.27.23" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-27-23.png)Drag the *Weighted Average ms/Read* calculated field to the "Rows" shelf:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-29-41.png" class="aligncenter size-full wp-image-2028" width="640" height="421" alt="Screen Shot 2014-07-07 at 20.29.41" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-29-41.png)Drag the *FG* dimension to the "Color Marks" shelf:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-32-33.png" class="aligncenter size-full wp-image-2029" width="160" height="274" alt="Screen Shot 2014-07-07 at 20.32.33" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-32-33.png)So that the graph looks like:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-33-15.png" class="aligncenter size-full wp-image-2030" width="640" height="419" alt="Screen Shot 2014-07-07 at 20.33.15" />](https://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-33-15.png)Create the same graph for the "Kby Read/s" measure (except that I want to see the sum (i.e the throughput and not the average) and put those 2 graphs into the same dashboard:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-07-07-at-20-39-42.png" class="aligncenter size-full wp-image-2032" width="640" height="383" alt="Screen Shot 2014-07-07 at 20.39.42" />](http://bdrouvot.files.wordpress.com/2014/07/screen-shot-2014-07-07-at-20-39-42.png)

Here we are.

<span style="text-decoration:underline;">**Conclusion:**</span>

-   We can create a csv file from the output of the *asm\_metrics* utility thanks to *csv\_asm\_metrics*.

<!-- -->

-   To do so, we have to collect all the fields of *asm\_metrics* with those options:

    <!-- -->

    -   *-show=inst,dbinst,fg,dg,dsk* for ASM &gt;= 11g
    -   *-show=inst,fg,dg,dsk* for ASM &lt; 11g

<!-- -->

-   Once you uploaded the csv file into your favourite visualization tool, don't forget to calculate **weighted averages** for *ms/Read, By/Read, ms/Write* and *By/Write* if you plan to graph the averages.

<!-- -->

-   You can imagine **a lot of graphs** thanks to the measures collected (*Reads/s, Kby Read/s, ms/Read, By/Read, Writes/s, Kby Write/s, ms/Write, By/Write*) and all those dimensions (*Snap Time, INST, DBINST, DG, FG, DSK*).

You can **download** the *csv\_asm\_metrics* utility from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](http://bdrouvot.wordpress.com/csv_asm_metrics_source/ "csv_asm_metrics_source").

**UPDATE**: You can see some use cases [here](http://bdrouvot.wordpress.com/2014/07/12/asm-performance-metrics-visualization-use-cases/ "ASM performance metrics visualization: Use cases").

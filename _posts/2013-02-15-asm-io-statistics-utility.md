---
layout: post
title: ASM I/O Statistics Utility
date: 2013-02-15 15:58:44.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Perl Scripts
- ToolKit
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:64;}s:2:"wp";a:1:{i:0;i:20;}}
  _wpas_skip_2077996: '1'
  twitter_cards_summary_img_size: a:6:{i:0;i:778;i:1;i:53;i:2;i:3;i:3;s:23:"width="778"
    height="53"";s:4:"bits";i:8;s:4:"mime";s:9:"image/png";}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/02/15/asm-io-statistics-utility/"
---

When I need to deal with ASM I/O statistics, the tools provided by Oracle (asmcmd iostat and asmiostat.sh from MOS \[ID 437996.1\]) do not suit my needs.

Then, I decided to create my own asmiostat utility that is helpful for 3 main reasons:

1.  <span style="color:#0000ff;">It provides useful real-time metrics.</span>
2.  <span style="line-height:13px;color:#0000ff;">You can aggregate the results following your needs **in a customizable way**.</span>
3.  <span style="color:#0000ff;">It does not need any change to the source: Simply download it and use it.</span>

The script takes a snapshot each second (default interval) from the  gv$asm\_disk\_stat cumulative view (instead of gv$asm\_disk because the information is exactly the same) and computes the differences with the previous snapshot.

The only difference with gv$asm\_disk\_stat is the information available in memory while v$asm\_disk access the disks to re-collect some information. Since the information required doesn't require to "re-collect" it from the disks, gv$asm\_disk\_stat is more appropriated here.

<span style="text-decoration:underline;">So, let's have a look of the metrics collected by the script:</span>

[<img src="{{ site.baseurl }}/assets/images/asm_metrics.png" class="aligncenter size-full wp-image-678" width="620" height="42" alt="asm_metrics" />](http://bdrouvot.files.wordpress.com/2013/02/asm_metrics.png)

<span style="text-decoration:underline;">Description is the following:</span>

-   Reads/s: Number of read per second.
-   KbyRead/s: Kbytes read per second.
-   Avg ms/Read: ms per read in average.
-   AvgBy/Read: Average Bytes per read.
-   Read Errors: Number of Errors.
-   Same metrics are provided for Write Operations.

<span style="color:#0000ff;">The interesting part is that you can decide how those metrics have to be calculated/aggregated</span>: I will give an example below and explain how to use the script to get this result.

Suppose I want to display the metrics by Diskgroup (default behavior), the output will be like:

[<img src="{{ site.baseurl }}/assets/images/asm_dg.png" class="aligncenter size-full wp-image-679" width="620" height="142" alt="asm_dg" />](http://bdrouvot.files.wordpress.com/2013/02/asm_dg.png)

You see the blank values for: INST (instance), FG (Failgroup) and DSK (disks)? It means that those values have been aggregated.

Of course, you can display them as well, for example let's display INST too:

[<img src="{{ site.baseurl }}/assets/images/asm_inst_dg.png" class="aligncenter size-full wp-image-680" width="620" height="249" alt="asm_inst_dg" />](http://bdrouvot.files.wordpress.com/2013/02/asm_inst_dg.png)

As you can see you now have the metrics for the diskgroups by Instance <span style="color:#0000ff;">and also</span> for the Instance itself (The row with the blank DG Field).

<span style="color:#0000ff;">You can "play" with those fields (Inst, dg, fg, dsk) as you want. Display them (or not) to get their metrics using the **&lt;-show&gt;** argument of the script.</span>

You can <span style="color:#0000ff;">also filter</span> on INST, DG and FG:  For example let's display the metrics for the DATA diskgroup and its associated disks and failgroups:

[<img src="{{ site.baseurl }}/assets/images/asm_inst_dg_fg_dsk.png" class="aligncenter size-full wp-image-681" width="620" height="196" alt="asm_inst_dg_fg_dsk" />](http://bdrouvot.files.wordpress.com/2013/02/asm_inst_dg_fg_dsk.png)

<span style="text-decoration:underline;">Now let's see the utility usage:</span>

The utility has been implemented as a part of the [real\_time.pl](http://bdrouvot.wordpress.com/real_time/ "real_time") script (Click on the link, and then on the view source button and then copy/paste the source code. You can also download the script from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit?pli=1) to avoid copy/paste (click on the link))

This script collects also a lot of useful real-time metrics: See description of this script into this [post](http://bdrouvot.wordpress.com/2013/01/30/real-time-database-utilities-grouped-into-a-single-script/ "Real-Time database utilities grouped into a single script").

<span style="text-decoration:underline;">The help associated to amsiostat:</span>

     ./real_time.pl -type=asmiostat -help

    Usage: ./real_time.pl -type=asmiostat [-interval] [-count] [-inst] [-dg] [-fg] [-show] [-help]
     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter    Comment                                                      Default
      ---------    -------                                                      -------
      -INST=       ALL - Show all Instance(s)                                   ALL
                   CURRENT - Show Current Instance
                   INSTANCE_NAME,... - choose Instance(s) to display

      -DG=         Diskgroup to collect (comma separated list)                  ALL
      -FG=         Failgroup to collect (comma separated list)                  ALL
      -SHOW=       What to show: inst,fg,dg,dsk (comma separated list)          DG

    Example: ./real_time.pl -type=asmiostat
    Example: ./real_time.pl -type=asmiostat -inst=+ASM1
    Example: ./real_time.pl -type=asmiostat -dg=DATA -show=dg
    Example: ./real_time.pl -type=asmiostat -dg=data -show=inst,dg,fg
    Example: ./real_time.pl -type=asmiostat -show=dg,dsk
    Example: ./real_time.pl -type=asmiostat -show=inst,dg,fg,dsk

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on INST, DG and FG (by default no filter is applied).
-   You can customize the output (means the fields on which the metrics are reported) following your need thanks to the **&lt;-show&gt;** argument.
-   You have to set oraenv on one ASM instance.
-   The script has been tested on Linux, Unix and Windows.

I hope this will be useful for you. If you have any suggestions, any metrics that you want to be integrated: Please do not hesitate to come back to me.

<span style="text-decoration:underline;color:#000000;">UPDATES:</span>

-   See how you can use this new utility to get ASM preferred read metrics in this [post](http://bdrouvot.wordpress.com/2013/02/18/asm-preferred-read-collect-performance-metrics/ "ASM Preferred Read: Collect performance metrics").
-   [ASM I/O Statistics Utility V2](http://bdrouvot.wordpress.com/2013/07/05/asm-io-statistics-utility-v2/ "ASM I/O Statistics Utility V2") is ready.
-   The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

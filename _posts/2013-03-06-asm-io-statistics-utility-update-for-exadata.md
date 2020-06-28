---
layout: post
title: 'ASM I/O Statistics Utility: Update for Exadata'
date: 2013-03-06 11:17:45.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Exadata
- Perl Scripts
- ToolKit
tags: [ASM, oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:89;}s:2:"wp";a:1:{i:0;i:22;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/03/06/asm-io-statistics-utility-update-for-exadata/"
---

In this previous [post](http://bdrouvot.wordpress.com/2013/02/21/exadata-storage-cells-io-performance-metrics-and-io-distribution-with-db-servers/ "Exadata: Storage Cells IO performance metrics and IO distribution with DB servers") (You should read it to understand what will follow) I explained how my asmiostat utility could be useful for the Exadata community. For this, I made one assumption:

-   Each storage cell constitutes a separate failure group (in most common Exadata configuration) (see [Expert Oracle Exadata Book](http://www.expertoracleexadata.com/) for more details)

And I concluded with:

-   In case your Exadata configuration does not follow this rule:  One  failure group per storage cell, just be aware that I will update my asmiostat utility so that it will be able to group by storage cells in any case (thanks to the IP located into the disks path). I’ll keep you posted once ready.

<span style="text-decoration:underline;color:#0000ff;">Here we are</span>: I updated my asmiostat utility so that<span style="text-decoration:underline;color:#0000ff;"> you can choose to focus on IP (Exadata Cells)</span> instead of Failgroup.

So that now, you can measure the performance and the IO load across the DB servers and the Cells that way:

    ./real_time.pl -type=asmiostat -show=ip,inst

with the following ouput:

    Collecting 1 sec....
    ............................
    03:32:18                                                               Kby      Avg       AvgBy/    Read                Kby       Avg        AvgBy/    Write
    03:32:18   INST     DG          IP (Cells)        DSK        Reads/s   Read/s   ms/Read   Read      Errors   Writes/s   Write/s   ms/Write   Write     Errors
    03:32:18   ------   ---------   ---------------   --------   -------   ------   -------   ------    ------   --------   -------   --------   ------    ------
    03:32:18   +ASM                                              48        1600     10.6      34133     0        7          144       18.0       21065     0
    03:32:18   +ASM                 192.168.56.111               12        424      13.4      36181     0        3          48        32.4       16384     0
    03:32:18   +ASM                 192.168.56.101               36        1176     9.7       33451     0        4          96        7.2        24576     0

You can also choose to filter on some IP adresses (see the help):

    ./real_time.pl -type=asmiostat -help

    Usage: ./real_time.pl -type=asmiostat [-interval] [-count] [-inst] [-dg] [-fg] [-ip] [-show] [-help]
     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter    Comment                                                      Default
      ---------    -------                                                      -------
      -INST=       ALL - Show all Instance(s)                                   ALL
                   CURRENT - Show Current Instance
                   INSTANCE_NAME,... - choose Instance(s) to display

      -DG=         Diskgroup to collect (comma separated list)                  ALL
      -FG=         Failgroup to collect (comma separated list)                  ALL
      -IP=         IP (Exadata Cells) to collect (Wildcard allowed)             ALL
      -SHOW=       What to show: inst,fg|ip,dg,dsk (comma separated list)       DG

    Example: ./real_time.pl -type=asmiostat
    Example: ./real_time.pl -type=asmiostat -inst=+ASM1
    Example: ./real_time.pl -type=asmiostat -dg=DATA -show=dg
    Example: ./real_time.pl -type=asmiostat -dg=data -show=inst,dg,fg
    Example: ./real_time.pl -type=asmiostat -show=dg,dsk
    Example: ./real_time.pl -type=asmiostat -show=inst,dg,fg,dsk
    Example: ./real_time.pl -type=asmiostat -show=ip -ip='%10%'

<span style="text-decoration:underline;color:#0000ff;">Remarks:</span>

-   To get the asmiostat utility included into the [real\_time.pl](http://bdrouvot.wordpress.com/real_time/ "real_time") script:  Click on the link, and then on the view source button and then copy/paste the source code. You can also download the script from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit?pli=1) to avoid copy/paste (click on the link)
-   For a full description of my asmiostat utility see this [post](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility").

**UPDATE:** The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

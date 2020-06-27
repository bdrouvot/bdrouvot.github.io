---
layout: post
title: Find out the most physical IO consumers through ASM in real time
date: 2013-09-05 17:13:27.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/09/05/find-out-the-most-physical-io-consumers-through-asm-in-real-time/"
---

Well, suppose you are using ASM and it is servicing a lot of databases per machine. Suddenly the number of IOPS increased in such a way that your sysadmin/storage guy warn you up (I know this is not the real life, we are supposing ;-) ).

So I would like to find out which database(s) are responsible for this load. Of course I could connect on each database and check the oracle statistics but there is a simpler/faster way:  **Extract this information from ASM**.

Into a previous post [ASM I/O Statistics Utility V2](http://bdrouvot.wordpress.com/2013/07/05/asm-io-statistics-utility-v2/ "ASM I/O Statistics Utility V2"), I introduced a new feature of my asmiostat utility: ability to sort on reads or writes.

**I just added a new sort option: iops** (which is simply Reads/s+Writes/s).

<span style="text-decoration:underline;">Let's see the help:</span>

    ./real_time.pl -type=asmiostat -h

    Usage: ./real_time.pl -type=asmiostat [-interval] [-count] [-inst] [-dbinst] [-dg] [-fg] [-ip] [-show] [-sort_field] [-help]
     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter         Comment                                                           Default
      ---------         -------                                                           -------
      -INST=            ALL - Show all Instance(s)                                        ALL
                        CURRENT - Show Current Instance
                        INSTANCE_NAME,... - choose Instance(s) to display

      -DBINST=          Database Instance to collect (Wildcard allowed)                   ALL
      -DG=              Diskgroup to collect (comma separated list)                       ALL
      -FG=              Failgroup to collect (comma separated list)                       ALL
      -IP=              IP (Exadata Cells) to collect (Wildcard allowed)                  ALL
      -SHOW=            What to show: inst,dbinst,fg|ip,dg,dsk (comma separated list)     DG
      -SORT_FIELD=      reads|writes|iops                                                 NONE

    Example: ./real_time.pl -type=asmiostat
    Example: ./real_time.pl -type=asmiostat -inst=+ASM1
    Example: ./real_time.pl -type=asmiostat -dg=DATA -show=dg
    Example: ./real_time.pl -type=asmiostat -dg=data -show=inst,dg,fg
    Example: ./real_time.pl -type=asmiostat -show=dg,dsk
    Example: ./real_time.pl -type=asmiostat -show=inst,dg,fg,dsk
    Example: ./real_time.pl -type=asmiostat -interval=5 -count=3 -sort_field=iops

As you can see, we can now sort on iops.

<span style="text-decoration:underline;">**Let's now launch the script to retrieve the databases ordered by iops in real time:**</span>

-   Using **-show=dbinst** option as I want to see the databases.
-   Using **-sort\_field=iops** option as I want to order by iops.

<!-- -->

    ./real_time.pl -type=asmiostat -show=dbinst -sort_field=iops
    ............................
    Collecting 1 sec....
    ............................
    17:11:51                                                                             Kby      Avg       AvgBy/               Kby       Avg        AvgBy/
    17:11:51   INST     DBINST        DG            FG            DSK          Reads/s   Read/s   ms/Read   Read      Writes/s   Write/s   ms/Write   Write
    17:11:51   ------   -----------   -----------   -----------   ----------   -------   ------   -------   ------    ------     -------   --------   ------
    17:11:51            IATEBDTO_1                                             575       63696    2.2       113434    56         896       1.9        16384
    17:11:51            SMTBDTO_2                                              577       28816    1.1       51140     0          0         0.0        0
    17:11:51            BDTO_1                                                 59        920      0.3       15967     30         464       1.7        15838  
    17:11:51            BDTO_2                                                 3         48       0.6       16384     0          0         0.0        0      
    17:11:51            BKP10GR2_1                                             2         32       0.0       16384     0          0         0.0        0      
    17:11:51            JCAASM_1                                               2         32       0.9       16384     0          0         0.0        0      
    17:11:51            MILASM_1                                               2         32       0.4       16384     0          0         0.0        0

As you can see the IATEBDTO\_1 instance is the one that recorded the most physical IO activity (Reads/s + Writes/s) during the last second.

**So we are able to find out quickly which databases are the most physical IO consumers in real time thanks to the ASM metrics**.

<span style="text-decoration:underline;">**Remarks:**</span>

-   **This is real time information: **the script takes a snapshot each second (default interval) from the gv$asm\_disk\_iostat (or gv$asm\_disk\_stat depending of the version) cumulative view and **computes the delta** with the previous snapshot.
-   You can display the database instances (**show=dbinst**) as of 11gr1 (as it is based on the gv$asm\_disk\_iostat view).
-   You can also find out which host is the most responsible for the physical IO thanks to the **show=inst** option (as it will display the ASM instances) (if not a 12c Flex ASM).
-   You can also find out which diskgroup is the most responsible for the physical IO thanks to the **show=dg** option.
-   You can also find out.... (I let you finish the sentence **following your needs: failgroups, disks, databases per diskgroup...** as the asmiostat utility output is customizable (see this [post](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility")).
-   You can download the asmiostat utility (which is part of the real\_time.pl script) from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU).

**UPDATE:** The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

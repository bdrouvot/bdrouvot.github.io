---
layout: post
title: Welcome to db_io_metrics, a new utility to display database physical IO metrics
  in real time
date: 2014-04-30 15:34:46.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
tags: [oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5867279720030183424&type=U&a=JmRK
  publicize_facebook_url: https://facebook.com/1428505997402918
  _wpas_done_5536151: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/VpiYnJWG49C
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/rnxBQrgBKP
  _wpas_done_2225791: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_5536151: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/04/30/welcome-to-db_io_metrics-a-new-utility-to-display-database-physical-io-metrics-in-real-time/"
---

The db\_io<span class="skimlinks-unlinked">\_metrics.pl</span> utility is used to display database physical IO real-time metrics. It basically takes a snapshot each second (default interval) from the gv$filestat and gv$tempstat cumulative views and **computes the delta** with the previous snapshot. The utility is RAC and Multitenant aware.

<span style="text-decoration:underline;">**This utility:**</span>

-   provides useful metrics.
-   is RAC aware.
-   detects if it is connected to a multitenant database and then is able to display the containers metrics.
-   is fully customizable: you can aggregate the results depending on your needs.
-   does not install anything into the database.

<span style="text-decoration:underline;">**It displays the following metrics:**</span>

-   Reads/s: Number of read per second.
-   KbyRead/s: Kbytes read per second.
-   Avg ms/Read: ms per read in average.
-   AvgBy/Read: Average Bytes per read.
-   Same metrics are provided for Write Operations.

<span style="text-decoration:underline;">**At the following levels:**</span>

-   Database Instance.
-   Database container.
-   Filesystem or ASM Diskgroup.
-   Tablespace.
-   Datafile.

<span style="text-decoration:underline;">**Let’s see the help:**</span>

    ./db_io_metrics.pl -help

    Usage: ./db_io_metrics.pl [-interval] [-count] [-inst] [-cont] [-fs_dg] [-tbs] [-file] [-fs_delimiter] [-show] [-display] [-sort_field] [-help]

     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter         Comment                                                           Default
      ---------         -------                                                           -------
      -INST=            ALL - Show all Instance(s)                                        ALL
                        CURRENT - Show Current Instance
      -CONT=            Container to collect (wildcard allowed)                           ALL
      -FS_DG=           Filesystem or Diskgroup to collect (wildcard allowed)             ALL
      -TBS=             Tablespace to collect (wildcard allowed)                          ALL
      -FILE=            Datafile to collect (wildcard allowed)                            ALL
      -FS_DELIMITER=    Folder which follows the FS mount point                           ORADATA
      -SHOW=            What to show: inst,cont,fs_dg,tbs,file (comma separated list)     INST
      -DISPLAY=         What to display: snap,avg (comma separated list)                  SNAP
      -SORT_FIELD=      reads|writes|iops                                                 NONE

    Example: ./db_io_metrics.pl
    Example: ./db_io_metrics.pl  -inst=BDT_1
    Example: ./db_io_metrics.pl  -show=inst,tbs
    Example: ./db_io_metrics.pl  -show=inst,tbs -tbs=%UNDO%
    Example: ./db_io_metrics.pl  -show=fs_dg
    Example: ./db_io_metrics.pl  -show=inst,fs_dg -display=avg
    Example: ./db_io_metrics.pl  -show=inst,fs_dg -sort_field=reads
    Example: ./db_io_metrics.pl  -show=inst,tbs,cont
    Example: ./db_io_metrics.pl  -show=inst,tbs,cont -cont=%P_%
    Example: ./db_io_metrics.pl  -show=inst,cont -sort_field=iops

<span style="text-decoration:underline;">**The main options/features are:**</span>

1.  You can choose the number of snapshots to display and the time to wait between snapshots.
2.  You can choose on which **database instance** to collect the metrics thanks to the -**INST=** parameter.
3.  You can choose on which **database container** to collect the metrics thanks to the **-CONT=** parameter (wilcard allowed).
4.  You can choose on which **Diskgroup or Filesystem** to collect the metrics thanks to the** -FS\_DG=** parameter (wildcard allowed).
5.  You can choose on which **tablespace** to collect the metrics thanks to the **-TBS=** parameter (wildcard allowed).
6.  You can choose on which **datafile** to collect the metrics thanks to the **-FILE=** parameter (wildcard allowed).
7.  You can choose which folder is your Filesystem delimiter thanks to the **-FS\_DELIMITER**= parameter.
8.  You can **aggregate the results** on the database instances, containers, diskgroups or filesystems, tablespaces level thanks to the -**SHOW=** parameter.
9.  You can display the metrics **per snapshot, the average metrics** value since the collection began (that is to say since the script has been launched) or both thanks to the -**DISPLAY=** parameter.
10. You can sort based on the number of **reads**, number of **writes** or number of **IOPS** (reads+writes) thanks to the -**SORT\_FIELD=** parameter.

<span style="text-decoration:underline;">**Let's see some use cases:**</span>

Report the physical IO metrics for the database instances:

    ./db_io_metrics.pl -show=inst
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    12:55:31                                                                                Kby      Avg       AvgBy/               Kby       Avg        AvgBy/
    12:55:31   INST         CONT         FS_DG      TBS            FILE           Reads/s   Read/s   ms/Read   Read      Writes/s   Write/s   ms/Write   Write
    12:55:31   ----------   ----------   --------   ------------   ------------   -------   ------   -------   ------    --------   -------   --------   ------
    12:55:31   CBDT1                                                              376.0     3008     5.8       8192      0.0        0         0.0        0
    12:55:31   CBDT2                                                              346.0     2768     15.6      8192      0.0        0         0.0        0

Report the physical IO metrics per database instances and per containers and sort by iops:

    ./db_io_metrics.pl -show=inst,cont -sort_field=iops
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    12:57:59                                                                                Kby      Avg       AvgBy/               Kby       Avg        AvgBy/
    12:57:59   INST         CONT         FS_DG      TBS            FILE           Reads/s   Read/s   ms/Read   Read      Writes/s   Write/s   ms/Write   Write
    12:57:59   ----------   ----------   --------   ------------   ------------   -------   ------   -------   ------    --------   -------   --------   ------
    12:57:59   CBDT2                                                              293.0     2344     18.8      8192      0.0        0         0.0        0
    12:57:59   CBDT2        CDB$ROOT                                              150.0     1200     18.4      8192      0.0        0         0.0        0
    12:57:59   CBDT2        P_1                                                   143.0     1144     19.2      8192      0.0        0         0.0        0
    12:57:59   CBDT2        PDB$SEED                                              0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT2        P_2                                                   0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT2        P_3                                                   0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT1                                                              274.0     2192     9.1       8192      0.0        0         0.0        0
    12:57:59   CBDT1        CDB$ROOT                                              274.0     2192     9.1       8192      0.0        0         0.0        0
    12:57:59   CBDT1        PDB$SEED                                              0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT1        P_1                                                   0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT1        P_2                                                   0.0       0        0.0       0         0.0        0         0.0        0
    12:57:59   CBDT1        P_3                                                   0.0       0        0.0       0         0.0        0         0.0        0

Report the physical IO metrics per database instances and per filesystem or ASM diskgroup:

    ./db_io_metrics.pl -show=inst,fs_dg
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    13:00:39                                                                                Kby      Avg       AvgBy/               Kby       Avg        AvgBy/
    13:00:39   INST         CONT         FS_DG      TBS            FILE           Reads/s   Read/s   ms/Read   Read      Writes/s   Write/s   ms/Write   Write
    13:00:39   ----------   ----------   --------   ------------   ------------   -------   ------   -------   ------    --------   -------   --------   ------
    13:00:39   CBDT1                                                              349.0     2792     8.3       8192      0.0        0         0.0        0
    13:00:39   CBDT1                     DATA                                     349.0     2792     8.3       8192      0.0        0         0.0        0
    13:00:39   CBDT2                                                              272.0     2176     25.4      8192      0.0        0         0.0        0
    13:00:39   CBDT2                     DATA                                     272.0     2176     25.4      8192      0.0        0         0.0        0

Report the physical IO metrics per... I let you finish the sentence as the utility is customizable enough ;-)

<span style="text-decoration:underline;">**Remarks:**</span>

-   The metrics are "only" linked to the datafiles and tempfiles (See the [oracle documentation for more details](http://docs.oracle.com/cd/E16655_01/server.121/e17615/refrn30087.htm#REFRN30087)).
-   In case your database is on Filesystems, you have to change the "FS\_DELIMITER" argument to aggregate the metrics at the Filesystem level. For example, if the FS are :

<!-- -->

    Filesystem             size   used  avail capacity  Mounted on
    devora11-data/u500     960G   927G    33G    97%    /ec/dev/server/oracle/devora11/u500
    devora11-data/u501     960G   767G   193G    80%    /ec/dev/server/oracle/devora11/u501
    devora11-data/u502     500G   445G    55G    89%    /ec/dev/server/oracle/devora11/u502

And the datafiles are located "behind" the oradata folder:

    /ec/dev/server/oracle/devora11/u500/oradata/BDT
    /ec/dev/server/oracle/devora11/u501/oradata/BDT
    /ec/dev/server/oracle/devora11/u502/oradata/BDT

Then you can launch the utility that way:

    ./db_io_metrics.pl -show=inst,fs_dg -fs_delimiter=oradata
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    11:00:35                                                                                            Kby      Avg       AvgBy/               Kby       Avg        AvgBy/
    11:00:35   INST         FS_DG                       TBS              FILE             Reads/s   Read/s   ms/Read   Read      Writes/s   Write/s   ms/Write   Write
    11:00:35   ----------   ----------------            --------------   --------------   -------   ------   -------   ------    --------   -------   --------   ------
    11:00:35   BDT                                                                        129.0     1032     8.3       8192      0.0        0         0.0        0
    11:00:35   BDT          /ec/dev/server/oracle/devora11/u500/                                      67.0      536      10.0      8192      0.0        0         0.0        0
    11:00:35   BDT          /ec/dev/server/oracle/devora11/u501/                                      22.0      176      5.5       8192      0.0        0         0.0        0
    11:00:35   BDT          /ec/dev/server/oracle/devora11/u502/                                      40.0      320      7.0       8192      0.0        0         0.0        0

 

-   If you don't want to see the FS (do not use *-show=fs\_dg*), then there is no need to specify the *-fs\_delimiter* argument.
-   Reading the good article [A Closer Look at CALIBRATE\_IO](http://externaltable.blogspot.be/2014/04/a-closer-look-at-calibrateio.html) from Luca Canali gave me the idea to create this utility.
-   If you are interested in those real-time metrics at the ASM level, you can have a look to the [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm_metrics") utility.

You can **download** it from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](http://bdrouvot.wordpress.com/db_io_metrics_source/ "db_io_metrics_source").

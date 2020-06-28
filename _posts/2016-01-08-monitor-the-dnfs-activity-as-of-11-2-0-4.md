---
layout: post
title: Monitor the dNFS activity as of 11.2.0.4
date: 2016-01-08 16:53:59.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
- ToolKit
tags: [oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/685489652522827776";}}
  _publicize_job_id: '18544080011'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6091255340907716609&type=U&a=xlAt
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/gFvYsMYx5gf
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/01/08/monitor-the-dnfs-activity-as-of-11-2-0-4/"
---

As you may know, there is some views available since 11g to monitor the [dnfs](http://www.oracle.com/technetwork/articles/directnfsclient-11gr1-twp-129785.pdf) activity:

-   **v$dnfs\_servers**: Shows a table of servers accessed using Direct NFS.
-   **v$dnfs\_files**: Shows a table of files now open with Direct NFS.
-   **v$dnfs\_channels**: Shows a table of open network paths (or channels) to servers for which Direct NFS is providing files.
-   **v$dnfs\_stats**: Shows a table of performance statistics for Direct NFS.

One interesting thing is that the v$dnfs\_stats view provides two new columns as of 11.2.0.4:

-   NFS\_READBYTES: Number of bytes read from NFS server
-   NFS\_WRITEBYTES: Number of bytes written to NFS server

See the oracle documentation [here](http://docs.oracle.com/cd/E11882_01/server.112/e40402/dynviews_1120.htm#REFRN30495).

Then, by sampling the view we can provide those metrics:

-   Reads/s: Number of read per second.
-   KbyRead/s: Kbytes read per second (as of 11.2.0.4).
-   AvgBy/Read: Average bytes per read (as of 11.2.0.4).
-   Writes/s: Number of Write per second.
-   KbyWrite/s: Kbytes write per second (as of 11.2.0.4).
-   AvgBy/Write: Average bytes per write (as of 11.2.0.4).

To do so, I just created a very simple **db\_io\_dnfs\_metrics.pl** utility. It basically takes a snapshot each second (default interval) from the **gv$dnfs\_stats** view and computes the delta with the previous snapshot. The utility is RAC aware.

**Let's see the help:**

    $>./db_io_dnfs_metrics.pl -help

    Usage: ./db_io_dnfs_metrics.pl [-interval] [-count] [-inst] [-display] [-sort_field] [-help]

     Default Interval : 1 second.
     Default Count    : Unlimited

      Parameter         Comment                                                           Default
      ---------         -------                                                           -------
      -INST=            ALL - Show all Instance(s)                                        ALL
                        CURRENT - Show Current Instance
      -DISPLAY=         What to display: snap,avg (comma separated list)                  SNAP
      -SORT_FIELD=      reads|writes|iops                                                 NONE

    Example: ./db_io_dnfs_metrics.pl
    Example: ./db_io_dnfs_metrics.pl  -inst=BDT_1
    Example: ./db_io_dnfs_metrics.pl  -sort_field=reads

**This is a very simple utility, the options/features are:**

1.  You can choose the number of snapshots to display and the time to wait between the snapshots.
2.  In case of RAC, you can choose on which **database instance** to collect the metrics thanks to the –**INST=** parameter.
3.  You can display the metrics **per snapshot, the average metrics** value since the collection began (that is to say since the script has been launched) or both thanks to the –**DISPLAY=** parameter.
4.  In case of RAC, you can sort the instances based on the number of **reads**, number of **writes**,** **number of IOPS (reads+writes) thanks to the –**SORT\_FIELD=** parameter.

**Examples:**

Collecting on a single Instance:

    $>./db_io_dnfs_metrics.pl
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    14:26:18                          Kby       AvgBy/               Kby       AvgBy/
    14:26:18   INST         Reads/s   Read/s    Read      Writes/s   Write/s   Write      IOPS       MB/s
    14:26:18   ----------   -------   -------   -------   --------   -------   --------   --------   --------
    14:26:18   VSBDT        321       2568      8192      0          0         0          321        2.5
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    14:26:19                          Kby       AvgBy/               Kby       AvgBy/
    14:26:19   INST         Reads/s   Read/s    Read      Writes/s   Write/s   Write      IOPS       MB/s
    14:26:19   ----------   -------   -------   -------   --------   -------   --------   --------   --------
    14:26:19   VSBDT        320       2560      8192      1          16        16384      321        2.5

Collecting on a RAC database, sorting the Instances by the number of read:

    $>./db_io_dnfs_metrics.pl -sort_field=reads
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    17:21:21                          Kby       AvgBy/               Kby       AvgBy/
    17:21:21   INST         Reads/s   Read/s    Read      Writes/s   Write/s   Write      IOPS       MB/s
    17:21:21   ----------   -------   -------   -------   --------   -------   --------   --------   --------
    17:21:21   VBDTO_1      175       44536     260599    0          0         0          175        43.5
    17:21:21   VBDTO_2      69        2272      33718     0          0         0          69         2.2
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    17:21:22                          Kby       AvgBy/               Kby       AvgBy/
    17:21:22   INST         Reads/s   Read/s    Read      Writes/s   Write/s   Write      IOPS       MB/s
    17:21:22   ----------   -------   -------   -------   --------   -------   --------   --------   --------
    17:21:22   VBDTO_2      151       36976     250751    0          0         0          151        36.1
    17:21:22   VBDTO_1      131       33408     261143    0          0         0          131        32.6
    ............................
    Collecting 1 sec....
    ............................

    ......... SNAP TAKEN AT ...................

    17:21:23                          Kby       AvgBy/               Kby       AvgBy/
    17:21:23   INST         Reads/s   Read/s    Read      Writes/s   Write/s   Write      IOPS       MB/s
    17:21:23   ----------   -------   -------   -------   --------   -------   --------   --------   --------
    17:21:23   VBDTO_2      133       33592     258633    0          0         0          133        32.8
    17:21:23   VBDTO_1      121       31360     265394    0          0         0          121        30.6

**Remarks:**

-   The utility works only as of 11.2.0.4.
-   You can **download** it from [this repository](https://docs.google.com/folderview?id=0B7Jf_4JdsptpRHdyOWk1VTdUdEU) or copy the source code from [this page](https://bdrouvot.wordpress.com/db_io_dnfs_metrics_script/ "db_io_type_metrics_source").
-   This utility joins the [db\_io\* family](https://bdrouvot.wordpress.com/2014/05/09/db_iometrics-family/).
-   For version prior to 11.2.0.4, you may find [Glenn Fawcett's script](https://glennfawcett.wordpress.com/2010/02/18/simple-script-to-monitor-dnfs-activity/) useful.

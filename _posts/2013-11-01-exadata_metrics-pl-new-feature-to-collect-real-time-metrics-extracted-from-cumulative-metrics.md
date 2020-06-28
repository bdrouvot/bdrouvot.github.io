---
layout: post
title: 'exadata_metrics.pl: New feature to collect real-time metrics extracted from
  cumulative metrics'
date: 2013-11-01 15:39:02.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
tags: [exadata, oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/0NVzZbwYmV
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5802050953410539520&type=U&a=Dapv
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/11/01/exadata_metrics-pl-new-feature-to-collect-real-time-metrics-extracted-from-cumulative-metrics/"
---

I am a big fan of real-time metrics extraction based on the cumulative metrics: That is to say take a snapshot each second (default interval) from the cumulative metrics and **computes the delta** with the previous snapshot.

I build such tools for [ASM metrics](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time") and for [Exadata Cell metrics](http://bdrouvot.wordpress.com/2013/03/05/exadata-real-time-metrics-extracted-from-cumulative-metrics-part-ii/ "Exadata real-time metrics extracted from cumulative metrics:  Part II").

Recently, I added a new feature to the [asm\_metrics.pl](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time") script: The ability to display the average **delta values since the collection began** (Means since we launched the script).

Well, I just added this new feature to the exadata\_metrics.pl script.

<span style="text-decoration:underline;">Of course the old features remain:</span>

-   You can choose the number of snapshots to display and the time to wait between the snapshots.
-   You can choose to filter on name and objectname based on predicates (see the help).
-   You can work on all the cells or a subset thanks to the CELL or the GROUPFILE parameter.
-   You can decide the way to compute the metrics with no aggregation, aggregation on cell, objectname or both.

<span style="text-decoration:underline;">Let's see the help:</span>

     ./exadata_metrics.pl help

    Usage: ./exadata_metrics.pl [Interval [Count]] [cell=|groupfile=] [display=] [show=] [top=] [name=] [name!=] [objectname=] [objectname!=]

     Default Interval : 1 second.
     Default Count : Unlimited

     Parameter                 Comment                                                      Default
     ---------                 -------                                                      -------
     CELL=                     comma-separated list of cells
     GROUPFILE=                file containing list of cells
     SHOW=                     What to show (name included): cell,objectname                ALL
     DISPLAY=                  What to display: snap,avg (comma separated list)             SNAP
     TOP=                      Number of rows to display                                    10
     NAME=                     ALL - Show all cumulative metrics (wildcard allowed)         ALL
     NAME!=                    Exclude cumulative metrics (wildcard allowed)                EMPTY
     OBJECTNAME=               ALL - Show all objects (wildcard allowed)                    ALL
     OBJECTNAME!=              Exclude objects (wildcard allowed)                           EMPTY

    utility assumes passwordless SSH from this cell node to the other cell nodes
    utility assumes ORACLE_HOME has been set (with celladmin user for example)

    Example : ./exadata_metrics.pl cell=cell
    Example : ./exadata_metrics.pl groupfile=./cell_group
    Example : ./exadata_metrics.pl groupfile=./cell_group show=name
    Example : ./exadata_metrics.pl cell=cell objectname='CD_disk03_cell' name!='.*RQ_W.*'
    Example : ./exadata_metrics.pl cell=cell name='.*BY.*' objectname='.*disk.*' name!='GD.*' objectname!='.*disk1.*'

**The “display” option is the new one:** It allows you to display the delta snap values (as previously), the average delta values since the collection began (that is to say since the script has been launched) or both.

<span style="text-decoration:underline;">Let's see one example:  
</span>

I want to extract real-time metrics from the %IO\_TM\_R% cumulative ones, for 2 cells and aggregate the results for all the cells and all the objectname (means I just want to show the metrics). I want to see each snapshots **and the average since** we launched the script.

So, I launch the script that way:

    ./exadata_metrics.pl cell=exacell1,exacell2 display=avg,snap name='.*IO_TM_R.*' show=name

It will produce this kind of output:

    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    ......... SNAP FOR LAST COLLECTION TIME ...................

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    60                                 CD_IO_TM_R_LG                                                                            0.00 us
    60                                 CD_IO_TM_R_SM                                                                            807269.00 us

    ......... AVG SINCE FIRST COLLECTION TIME...................

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    60                                 CD_IO_TM_R_LG                                                                            0.00 us
    60                                 CD_IO_TM_R_SM                                                                            807269.00 us

So, no differences between the delta snap and the average after the first snap (which is obvious :-) ).

Into the "SNAP" section, the delta(s) field **computes the delta in seconds of the collectionTime recorded into the snapshots** (see this [post](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters") for more details). Into the "AVG" section, the delta(s) field is the sum of the delta(s) field from all the previous "SNAP" sections.

Let's have a look to the output after 2 minutes of metrics extraction to make things clear:

    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    ......... SNAP FOR LAST COLLECTION TIME ...................

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    61                                 CD_IO_TM_R_LG                                                                            0.00 us
    61                                 CD_IO_TM_R_SM                                                                            348479.00 us

    ......... AVG SINCE FIRST COLLECTION TIME...................

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    121                                CD_IO_TM_R_LG                                                                            0.00 us
    121                                CD_IO_TM_R_SM                                                                            577874.00 us

So, during the last 61 seconds of collection the CD\_IO\_TM\_R\_SM metric delta value is 348479.00 us.

It leads to an average of 577874.00 us since the collection began (That is to say since 121 seconds of metrics collection).

<span style="text-decoration:underline;">**Remarks**:</span>

-   The exadata\_metrics.pl script can be downloaded from [this repository.](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)
-   The DELTA (s) field is an important one to interpret the result correctly, I strongly recommend to read this [post](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters") for a better understanding of it.

<span style="text-decoration:underline;">**Conclusion:**</span>

The exadata\_metrics.pl now offers the ability to display the average delta values since the collection began (that is to say since the script has been launched).

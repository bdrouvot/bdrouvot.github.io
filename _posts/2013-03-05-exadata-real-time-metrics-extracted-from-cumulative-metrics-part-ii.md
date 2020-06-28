---
layout: post
title: 'Exadata real-time metrics extracted from cumulative metrics:  Part II'
date: 2013-03-05 19:33:13.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
- ToolKit
tags: [exadata, oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:87;}s:2:"wp";a:1:{i:0;i:22;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/03/05/exadata-real-time-metrics-extracted-from-cumulative-metrics-part-ii/"
---

Into this [post](http://bdrouvot.wordpress.com/2012/11/27/exadata-real-time-metrics-extracted-from-cumulative-metrics/ "Exadata real-time metrics extracted from cumulative metrics") I introduced my [exadata\_metrics.pl](http://bdrouvot.wordpress.com/exadata_metrics/ "exadata_metrics") script that I use to collect real-time cell's metrics from the cumulative ones.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">I added new features on it that you may find useful/helpful</span></span>:

1.  First  I added the possibility to aggregate the results based on the cell, on the metricobjectname or both: That is to say<span style="color:#0000ff;"> you can customize the way</span> the metrics are computed.
2.  Secondly I added the possibility to use a "groupfile" (a file containing a list of cells: same format as the dcli utility) as input (could be useful if your exadata has a lot of cells ;-) )

<span style="text-decoration:underline;color:#0000ff;">Lets see examples to make it clear</span>: For this I will focus on the CD\_IO\_TM\_R\_SM metric and on 2 cells only (for output simplicity) during the last 100 seconds.

**To collect the metrics without aggregation**:

    ./exadata_metrics.pl 100 groupfile=./cell_group name='CD_IO_TM_R_SM'

**The output will be like**:

    07:59:46   CELL                    NAME                         OBJECTNAME                                                  VALUE
    07:59:46   ----                    ----                         ----------                                                  -----
    07:59:46   cell2                   CD_IO_TM_R_SM                CD_disk05_cell                                              0.00 us
    07:59:46   cell2                   CD_IO_TM_R_SM                FD_03_cell                                                  0.00 us
    07:59:46   cell2                   CD_IO_TM_R_SM                CD_disk03_cell                                              36479.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk06_cell                                              41572.00 us
    07:59:46   cell2                   CD_IO_TM_R_SM                CD_disk02_cell                                              167822.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk02_cell                                              522659.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk04_cell                                              523456.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk01_cell                                              553921.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                FD_02_cell                                                  580027.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                FD_03_cell                                                  801521.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                FD_01_cell                                                  845028.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                FD_00_cell                                                  1228914.00 us
    07:59:46   cell2                   CD_IO_TM_R_SM                CD_disk01_cell                                              1229929.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk05_cell                                              1314321.00 us
    07:59:46   cell                    CD_IO_TM_R_SM                CD_disk03_cell                                              4055302.00 us

**With an aggregation on the objectname (I use the "show" option has I want to display cell and name)**:

    ./exadata_metrics.pl 100 groupfile=./cell_group name='CD_IO_TM_R_SM' show=cell,name

**The output would have been like**:

    07:59:48   CELL                    NAME                         OBJECTNAME                                                  VALUE
    07:59:48   ----                    ----                         ----------                                                  -----
    07:59:48   cell2                   CD_IO_TM_R_SM                                                                            1434230.00 us
    07:59:48   cell                    CD_IO_TM_R_SM                                                                            10466721.00 us

As you can see, the objectname has disappeared as the metrics have been aggregated on it.

**Let's collect one more time with no aggregation**:

    ./exadata_metrics.pl 100 groupfile=./cell_group name='CD_IO_TM_R_SM'

**With no aggregation the output will be like**:

    09:37:01   CELL                    NAME                         OBJECTNAME                                                  VALUE
    09:37:01   ----                    ----                         ----------                                                  -----
    09:37:01   cell2                   CD_IO_TM_R_SM                CD_disk09_cell                                              0.00 us
    09:37:01   cell2                   CD_IO_TM_R_SM                CD_disk05_cell                                              0.00 us
    09:37:01   cell2                   CD_IO_TM_R_SM                FD_03_cell                                                  0.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk12_cell                                              0.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk01_cell                                              879.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk06_cell                                              41629.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                FD_03_cell                                                  111676.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                FD_02_cell                                                  233388.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                FD_01_cell                                                  253784.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk04_cell                                              519624.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk05_cell                                              587848.00 us
    09:37:01   cell2                   CD_IO_TM_R_SM                CD_disk03_cell                                              1949331.00 us
    09:37:01   cell2                   CD_IO_TM_R_SM                CD_disk02_cell                                              2016198.00 us
    09:37:01   cell                    CD_IO_TM_R_SM                CD_disk03_cell                                              3220328.00 us
    09:37:01   cell2                   CD_IO_TM_R_SM                CD_disk01_cell                                              3631941.00 us

**With an aggregation on the cell and the objectname (**I use the "show" option has I want to display name)****:

    ./exadata_metrics.pl 100 groupfile=./cell_group name='CD_IO_TM_R_SM' show=name

**The output would have been like**:

    09:36:59   CELL                    NAME                         OBJECTNAME                                                  VALUE
    09:36:59   ----                    ----                         ----------                                                  -----
    09:36:59                           CD_IO_TM_R_SM                                                                            12566626.00 us

As you can see, the objectname and the cell have disappeared as the metrics have been aggregated.

<span style="text-decoration:underline;color:#0000ff;">Let's see the help:</span>

     ./exadata_metrics.pl help

    Usage: ./exadata_metrics.pl [Interval [Count]] [cell=|groupfile=] [show=] [top=] [name=] [name!=] [objectname=] [objectname!=]

     Default Interval : 1 second.
     Default Count : Unlimited

     Parameter                 Comment                                                      Default
     ---------                 -------                                                      -------
     CELL=                     comma-separated list of cells
     GROUPFILE=                file containing list of cells
     SHOW=                     What to show (name included): cell,objectname                ALL
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

<span style="text-decoration:underline;color:#0000ff;">Conclusion:</span>

-   You are able to collect real-time metrics based on cumulative metrics.
-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on name and objectname based on predicates (see the help).
-   You can work on all the cells or a subset thanks to the CELL or the GROUPFILE parameter.
-   You can decide the way to compute the metrics with no aggregation, aggregation on cell, objectname or both.

To get the [exadata\_metrics.pl](http://bdrouvot.wordpress.com/exadata_metrics/ "exadata_metrics") script:  Click on the link, and then on the view source button and then copy/paste the source code. You can also download the script from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit?pli=1) to avoid copy/paste (click on the link).

Any remarks, suggestions, questions are welcome.

<span style="text-decoration:underline;">**Update:**</span>

You should read this post for a better interpretation of the utility: [Exadata Cell metrics: collectionTime attribute, something that matters](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters")

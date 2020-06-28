---
layout: post
title: Exadata real-time metrics extracted from cumulative metrics
date: 2012-11-27 08:55:26.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
- Perl Scripts
- ToolKit
tags:
- exadata
- oracle
meta:
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  _publicize_pending: '1'
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
permalink: "/2012/11/27/exadata-real-time-metrics-extracted-from-cumulative-metrics/"
---

Exadata provides a lot of useful metrics to monitor the Cells.

The Metrics can be of various types:

-   <span style="color:#0000ff;">Cumulative</span>: Cumulative statistics since the metric was created.
-   <span style="color:#0000ff;">Instantaneous</span>: Value at the time that the metric is collected.
-   <span style="color:#0000ff;">Rate</span>: Rates computed by averaging statistics over observation periods.
-   <span style="color:#0000ff;">Transition</span>: Are collected at the time when the value of the metrics has changed, and typically captures important transitions in hardware status.

You can found some information on how to exploit those metrics in those posts:

[UWE HESSE's post ](http://uhesse.com/2011/02/15/exadata-part-v-monitoring-with-database-control/)

[TANEL PODER's post](http://blog.tanelpoder.com/tag/exadata/)

But I think those types of metrics are  not enough to answer all the basic questions.

<span style="color:#0000ff;"><span style="text-decoration:underline;">Let me explain why with 2 examples:</span></span>

Let's have a look to the metrics GD\_IO\_RQ\_W\_SM and GD\_IO\_RQ\_W\_SM\_SEC (restricted to one Grid Disk for lisibility):

    dcli -c cell1 cellcli -e "list metriccurrent attributes name,metricType,metricObjectName,metricValue where name like \'.*GD_IO_RQ_W_SM.*\' and metricObjectName ='data_CD_disk01_cell'"

    cell1: GD_IO_RQ_W_SM        Cumulative  data_CD_disk01_cell     2,930 IO requests
    cell1: GD_IO_RQ_W_SM_SEC    Rate        data_CD_disk01_cell     0.3 IO/sec

So we can observe that this "cumulative" metric shows the number of small write I/O requests while its associated "rate" metric shows the number of small write I/O requests per seconds.

or

Let's have a look to the metrics CD\_IO\_TM\_W\_SM and CD\_IO\_TM\_W\_SM\_RQ (restricted to one Cell Disk for lisibility):

    dcli -c cell1 cellcli -e "list metriccurrent attributes name,metricType,metricObjectName,metricValue where name like \'.*CD_IO_TM_W.*SM.*\' and metricobjectname='CD_disk07_cell'"

    cell1: CD_IO_TM_W_SM        Cumulative  CD_disk07_cell      1,512,939 us
    cell1: CD_IO_TM_W_SM_RQ     Rate        CD_disk07_cell      168 us/request

So we can observe that this "cumulative" metric shows the small write I/O latency in us while its associated "rate" metric shows the small write I/O latency in us per request.

<span style="text-decoration:underline;color:#0000ff;">But how can I answer those questions:</span>

1.  How many small write I/O requests have been done during the last 80 seconds? (Unfortunately 0.3 \* 80 will not necessary provide the right answer as it depends of the "observation period" of the rate metrics)
2.  What was the small write I/O latency during the last 80 second ?

You could ask for the same kind of questions on all cumulative metrics.

To answer all those questions I created a perl script [exadata\_metrics.pl](http://bdrouvot.wordpress.com/exadata_metrics/ "exadata_metrics") (click on the link and then on the view source button  to copy/paste the source code) that extracts exadata real-time information metrics based on cumulative metrics.

<span style="text-decoration:underline;color:#0000ff;">That is to say the script works with **all the cumulative metrics** (the following command list all of them) :</span>

    cellcli -e "list metriccurrent attributes name,metricType where metricType='Cumulative'"

To extract real-time information the script takes a snapshot of cumulative metrics each second (default interval) and computes the differences with the previous snapshot.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">So, to get the answer to our first question :</span></span>

    ./exadata_metrics.pl 80 cell=cell1 name='GD_IO_RQ_W_SM' metricobjectname='data_CD_disk01_cell'

    04:30:38 CELL   NAME            OBJECTNAME      VALUE
    04:30:38 cell1  GD_IO_RQ_W_SM       data_CD_disk01_cell     0.00 IO requests
    --------------------------------------> NEW
    04:31:58 CELL   NAME            OBJECTNAME      VALUE
    04:31:58 cell1  GD_IO_RQ_W_SM       data_CD_disk01_cell     20.00 IO requests

As you can see 20 small write I/O requests have been generated during the last 80 seconds (which is different from 0.3\*80).

<span style="text-decoration:underline;color:#0000ff;">To get the answer to our second question :</span>

    ./exadata_metrics.pl 80 cell=cell1 name_like='.*CD_IO_TM_W.*SM.*' metricobjectname='CD_disk07_cell'

    06:48:33 CELL   NAME            OBJECTNAME      VALUE
    06:48:33 cell1  CD_IO_TM_W_SM       CD_disk07_cell      0.00 us
    --------------------------------------> NEW
    06:49:53 CELL   NAME            OBJECTNAME      VALUE
    06:49:53 cell1  CD_IO_TM_W_SM       CD_disk07_cell      3613.00 us

As you can see we the small write I/O latency has been  3613 us during the last 80 seconds.

<span style="text-decoration:underline;color:#0000ff;">Let's see the help of the script:</span>

    ./exadata_metrics.pl help

    Usage: ./exadata_metrics.pl [Interval [Count]] [cell=] [top=] [name=] [metricobjectname=] [name_like=] [metricobjectname_like=]

    Default Interval : 1 second.
    Default Count : Unlimited

    Parameter       Comment                     Default
    ---------       -------                     -------
    CELL=           comma separated list of cell to display
    TOP=            Number of rows to display           10
    NAME=           ALL - Show all cumulative metrics       ALL
    NAME_LIKE=      ALL - Show all cumulative metrics       ALL
    METRICOBJECTNAME=   ALL - Show all objects              ALL
    METRICOBJECTNAME_LIKE=  ALL - Show all objects              ALL

    Example: ./exadata_metrics.pl cell=cell1,cell2 name_like='.*FC.*'
    Example: ./exadata_metrics.pl cell=cell1,cell2 name='CD_IO_BY_W_LG'
    Example: ./exadata_metrics.pl cell=cell1,cell2 name='CD_IO_BY_W_LG' metricobjectname_like='.*disk.*'

The script is based on the dcli and the cellcli commands and their regular expressions (wich are described into [Kerry Osborne's  post](http://kerryosborne.oracle-guy.com/2010/11/cellcli-command-syntax-top-10/)).

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on name and metricobjectname based on like or equal predicates.
-   You can work on all the cells or a subset thanks to the mandatory CELL parameter.
-   A cell os user allowed to run dcli without password (celladmin for example) can launch the script (ORACLE\_HOME must be set).

Please don't hesitate to tell me if this is useful for you and if you find any issues with this script.

<span style="text-decoration:underline;">**Updates:**</span>

-   New features have been added, please see this [post](http://bdrouvot.wordpress.com/2013/03/05/exadata-real-time-metrics-extracted-from-cumulative-metrics-part-ii/ "Exadata real-time metrics extracted from cumulative metrics:  Part II").
-   You should read this post for a better interpretation of the utility: [Exadata Cell metrics: collectionTime attribute, something that matters](http://bdrouvot.wordpress.com/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/ "Exadata Cell metrics: collectionTime attribute, something that matters")

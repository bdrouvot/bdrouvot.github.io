---
layout: post
title: 'Exadata Cell metrics: collectionTime attribute, something that matters'
date: 2013-09-13 18:29:40.000000000 +02:00
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
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/09/13/exadata-cell-metrics-collectiontime-attribute-something-that-matters/"
---

Exadata provides a lot of useful metrics to monitor the Cells.

<span style="text-decoration:underline;">The Metrics can be of various types:</span>

-   Cumulative: Cumulative statistics since the metric was created.
-   Instantaneous: Value at the time that the metric is collected.
-   Rate: Rates computed by averaging statistics over observation periods.
-   Transition: Are collected at the time when the value of the metrics has changed, and typically captures important transitions in hardware status.

One attribute of the cumulative metric is the **collectionTime.**

<span style="text-decoration:underline;">For example, let's have a look to one of them:</span>

    CellCLI> list METRICCURRENT DB_IO_WT_SM detail
             name:                   DB_IO_WT_SM
             alertState:             normal
             collectionTime:         2013-09-12T23:46:14+02:00
             metricObjectName:       EXABDT
             metricType:             Cumulative
             metricValue:            120 ms
             objectType:             IORM_DATABASE

The collectionTime attribute is the time **at which the metric was collected**.

<span style="text-decoration:underline;">Why does it matter ?</span>

**Based on it, we can compute the delta in second between 2 collections.**

Let's see two use cases.

<span style="text-decoration:underline;">**First use case: **</span>Suppose, you decided to extract real-time metrics from the cumulative ones. To do so, you created a script that takes a snapshot of the cumulative metrics each second (default interval) and computes the delta with the previous snapshot (yes, I am describing my [exadata\_metrics.pl](http://bdrouvot.wordpress.com/exadata_metrics/ "exadata_metrics") script introduced into this [post](http://bdrouvot.wordpress.com/2012/11/27/exadata-real-time-metrics-extracted-from-cumulative-metrics/ "Exadata real-time metrics extracted from cumulative metrics") :-) ).

Then, if the **delta value of the metric is 0**, you need to know why (two explanations are possible as we'll see).

<span style="text-decoration:underline;">Let's see an example: I'll take a snapshot with a 40 seconds interval of 2 IORM cumulative metrics:</span>

    ./exadata_metrics.pl 40 cell=exacell1  name='DB_IO_WT_.*' objectname='EXABDT'
    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    00:19:21   CELL                    NAME                         OBJECTNAME                                                  VALUE
    00:19:21   ----                    ----                         ----------                                                  -----
    00:19:21   exacell1                DB_IO_WT_LG                  EXABDT                                                      0.00 ms
    00:19:21   exacell1                DB_IO_WT_SM                  EXABDT                                                      0.00 ms

Well, as you can see the computed (delta) value is 0.00 ms but:

-   does it mean that **no IO** has been queued by the IORM ?
-   or does it mean that the 2 snaps are **based on the same collectionTime**? (could be the case if the **collection interval is greater** than the interval you are using with my script).

To answer those questions, I modified the script so that it takes care of the collectionTime: **It computes the delta in seconds of the collectionTime recorded into the snapshots**.

<span style="text-decoration:underline;">Let's see it in action:</span>

<span style="text-decoration:underline;">Enable the IORM plan:</span>

    CellCLI> alter iormplan objective=auto;
    IORMPLAN successfully altered

<span style="text-decoration:underline;">and launch the script with a 40 seconds interval:</span>

    ./exadata_metrics.pl 40 cell=exacell1  name='DB_IO_WT_.*' objectname='EXABDT'

    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    61         exacell1                DB_IO_WT_SM                  EXABDT                                                      0.00 ms
    61         exacell1                DB_IO_WT_LG                  EXABDT                                                      1444922.00 ms

    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    60         exacell1                DB_IO_WT_SM                  EXABDT                                                      1.00 ms
    60         exacell1                DB_IO_WT_LG                  EXABDT                                                      2573515.00 ms

    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE
    --------   ----                    ----                         ----------                                                  -----
    0          exacell1                DB_IO_WT_LG                  EXABDT                                                      0.00 ms
    0          exacell1                DB_IO_WT_SM                  EXABDT                                                      0.00 ms

**Look at the DELTA(s) column**: It indicates the delta in seconds for the collectionTime attribute.

<span style="text-decoration:underline;">**So that:**</span>

-   **DELTA(s) &gt; 0**: Means you can check the metric value as the snaps are from 2 **distinct** collectionTime.
-   **DELTA(s) = 0**: Means the snaps come from the **same** collectionTime and then a metric value of 0 is obvious.

<span style="text-decoration:underline;">**Second use case:**</span>

As we now have the DELTA(s) value we can **compute by our own the associated (\_SEC) rate metrics**.

For example, from:

    ./exadata_metrics_orig_new.pl 10 cell=exacell1 name='DB_IO_.*' objectname='EXABDT'
    --------------------------------------
    ----------COLLECTING DATA-------------
    --------------------------------------

    DELTA(s)   CELL                    NAME                         OBJECTNAME                                                  VALUE                
    --------   ----                    ----                         ----------                                                  -----                
    60         exacell1                DB_IO_WT_SM                  EXABDT                                                      0.00 ms        
    60         exacell1                DB_IO_RQ_SM                  EXABDT                                                      153.00 IO requests
    60         exacell1                DB_IO_RQ_LG                  EXABDT                                                      292.00 IO requests
    60         exacell1                DB_IO_WT_LG                  EXABDT                                                      830399.00 ms

<span style="text-decoration:underline;">We can conclude, that:</span>

-   the number of large IO request per second is 292/60=4.87.
-   the number of small IO request per second is 153/60=2.55.

Let's verify those numbers with their associated rate metrics (DB\_IO\_RQ\_LG\_SEC and DB\_IO\_RQ\_SM\_SEC):

    cellcli -e "list metriccurrent attributes name,metrictype,metricobjectname,metricvalue,collectionTime where name like 'DB_IO_.*' and metricobjectname='EXABDT' and metrictype='Rate'"
             DB_IO_RQ_LG_SEC         Rate    EXABDT  4.9 IO/sec              2013-09-13T16:13:40+02:00
             DB_IO_RQ_SM_SEC         Rate    EXABDT  2.6 IO/sec              2013-09-13T16:13:40+02:00
             DB_IO_WT_LG_RQ          Rate    EXABDT  2,844 ms/request        2013-09-13T16:13:40+02:00
             DB_IO_WT_SM_RQ          Rate    EXABDT  0.0 ms/request          2013-09-13T16:13:40+02:00

Great, that's the same numbers.

<span style="text-decoration:underline;">**Conclusion:**</span>

The collectionTime metric attribute can be very useful when you extract real-time metrics from the cumulative ones as:

-   It provides a way to **interpret the results**.
-   it provides a way to **extract the rate metrics (\_SEC)** from their cumulatives ones.

<span style="text-decoration:underline;">Regarding the script:</span>

-   You are able to collect real-time metrics based on cumulative metrics.
-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on name and objectname based on predicates (see the help).
-   You can work on all the cells or a subset thanks to the CELL or the GROUPFILE parameter.
-   You can decide the way to compute the metrics with no aggregation, aggregation on cell, objectname or both.

You can download the exadata\_metrics.pl script from [this repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit).

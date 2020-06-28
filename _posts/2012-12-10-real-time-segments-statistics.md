---
layout: post
title: Real-Time segments statistics
date: 2012-12-10 07:28:48.000000000 +01:00
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
  publicize_twitter_user: BertrandDrouvot
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:4;}s:2:"wp";a:1:{i:0;i:3;}}
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2225791: '1'
  _wpas_done_2077996: '1'
  _edit_last: '40807211'
  _publicize_pending: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/10/real-time-segments-statistics/"
---

The v$segment\_statistics and v$segstat views are a goldmine to extract statistics that are associated with the Oracle segments.

You can see how useful it could be in those posts :

[Kevin Closson's post](http://kevinclosson.wordpress.com/2007/03/13/multiple-buffer-pools-with-oracle/)

[Jonathan Lewis's post](http://jonathanlewis.wordpress.com/2011/09/09/row-lock-waits/) or [this one](http://jonathanlewis.wordpress.com/segment-scans/)

[Arup Nanda's post](http://arup.blogspot.fr/2011/01/more-on-interested-transaction-lists.html)

But those views are cumulatives, so not so helpful to report real-time information on the segments (Imagine your database is generating a lot of I/O right now and you would like to know wich segments are generating those I/O).

To report real-time statistics on the segments I wrote the [segments stats.pl](http://bdrouvot.wordpress.com/segments_stats/ "segments_stats") script (click on the link and then on the view source button to copy/paste the source code) that basically takes a snapshot based on the v$segstat view each second (default interval) and computes the differences with the previous snapshot.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see an example:</span></span>

    ./segments_stats.pl

    Connecting to the Instance...

    07:10:45   INST_NAME    OWNER   OBJECT_NAME STAT_NAME                VALUE     
    07:10:45   BDT1     BDT BDTTAB          physical read requests   6         
    07:10:45   BDT1     BDT BDTTAB          segment scans            6         
    07:10:45   BDT1         BDT     BDTTAB          logical reads            48        
    07:10:45   BDT1         BDT     BDTTAB          physical reads           85        
    07:10:45   BDT1         BDT     BDTTAB          physical reads direct    85   
    --------------------------------------> NEW
    07:10:46   INST_NAME    OWNER   OBJECT_NAME STAT_NAME                VALUE     
    07:10:46   BDT1     BDT BDTTAB          segment scans            19
    07:10:46   BDT1     BDT BDTTAB          physical read requests   28         
    07:10:46   BDT1         BDT     BDTTAB          logical reads            48        
    07:10:46   BDT1         BDT     BDTTAB          physical reads           285        
    07:10:46   BDT1         BDT     BDTTAB          physical reads direct    285

So, as you can see the BDTTAB generated 285 physical reads during the last second.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Let's see the help:</span></span>

    ./segments_stats.pl help

    Usage: ./segments_stats.pl [Interval [Count]] [inst] [top=] [owner=] [statname=] [segment=] [includesys=]

            Default Interval : 1 second.
            Default Count    : Unlimited

      Parameter     Comment                          Default    
      ---------     -------                              -------    
      INST=             ALL - Show all Instance                              ALL        
           CURRENT - Show Current Instance                                         
           INSTANCE_NAME,... - choose Instance(s) to display                       

                           << Instances are only displayed in a RAC DB >>                       

      TOP=          Number of rows to display                            10         
      OWNER=        ALL - Show all OWNER                                 ALL        
      STATNAME=     ALL - Show all Stats                                 ALL        
      SEGMENT=      ALL - Show all SEGMENTS                              ALL        
      INCLUDESYS=           Show SYS OBJECTS                                     N          

    Example : ./segments_stats.pl segment='AQ%' statname='physical%'
    Example : ./segments_stats.pl segment='AQ%' statname='physical writes direct'

<span style="text-decoration:underline;color:#0000ff;">As usual:</span>

-   You can choose the number of snapshots to display and the time to wait between snapshots.
-   You can choose to filter on statname, segment and owner (by default no filter is applied).
-   This script is oracle RAC aware : you can work on all the instances, a subset or the local one.
-   You have to set oraenv on one instance of the database you want to diagnose first.
-   The script has been tested on Linux, Unix and Windows.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Remark for Exadata: </span></span>

As you can filter on the statname, you could choose to filter on the particular '<span style="color:#008000;">optimized physical reads</span>' statistic that way:**  
**

    ./segments_stats.pl statname='%optimized%'

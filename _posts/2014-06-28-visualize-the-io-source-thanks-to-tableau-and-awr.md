---
layout: post
title: Visualize the IO source thanks to Tableau and AWR
date: 2014-06-28 15:00:43.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Tableau
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _wpas_done_5536151: '1'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1446351688951682
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5888652049846935552&type=U&a=YjE-
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/Bwg6QBaWwX8
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/JO81FGBKQu
  _wpas_skip_2077996: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_5536151: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/06/28/visualize-the-io-source-thanks-to-tableau-and-awr/"
---

As you know, the wait event "db file sequential read" records "single block" IO performed **outside** the database buffer cache. But does the IO come from:

-   Filesystem cache (If any and used)
-   Disk Array cache
-   SSD
-   Spindle Disks
-   .....

<span style="text-decoration:underline;">It could be interesting to visualize the distribution of the IO source:</span>

-   Should you migrate from a cached filesystem to ASM (You may need to increase the database cache to put the previous Filesystem cached IOs into the database cache).
-   Should you use [Dynamic Tiering](http://flashdba.com/2014/05/23/understanding-disk-caching-and-tiering/) and want to figure out where the IOs come from (SSD, Spindle Disks..).

To do so, I'll use the AWR data coming from the dba\_hist\_event\_histogram view and [Tableau](http://www.tableausoftware.com/public//). I'll also extract the data coming from dba\_hist\_snapshot (to get the begin\_interval\_date time).

```
alter session set nls_date_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls_timestamp_format='YYYY/MM/DD HH24:MI:SS';

select * from dba_hist_event_histogram where  
snap_id >= (select min(snap_id) from dba_hist_snapshot  
where begin_interval_time >= to_date ('2014/06/01 00:00','YYYY/MM/DD HH24:MI'))  
and event_name='db file sequential read';

select * from dba_hist_snapshot where begin_interval_time >= to_date ('2014/06/01 00:00','YYYY/MM/DD HH24:MI');  
```

As you can see, there is no computation. This is just a simple extraction of the data.

Then I put those data into 2 csv files (awr\_snap\_for\_june.csv and awr\_event\_histogram.csv).

1\) Now, launch Tableau and select the csv files and add an inner join between those files:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-45-19.png" class="aligncenter size-full wp-image-1985" width="640" height="226" alt="Screen Shot 2014-06-28 at 13.45.19" />

2\) Go to the worksheet and put the "begin interval time" dimension into the "column" and change it to an "exact date" (Instead of Year):

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-48-13.png" class="aligncenter size-full wp-image-1986" width="640" height="598" alt="Screen Shot 2014-06-28 at 13.48.13" />

3\) Put the "Wait count" measure into the "Rows" and create a table calculation on it:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-55-33.png" class="aligncenter size-full wp-image-1988" width="640" height="401" alt="Screen Shot 2014-06-28 at 13.55.33" />

Choose "**difference**" as the "WAIT\_COUNT" field is cumulative and we want to see the **delta** between the AWR's snapshots.

4\) My graph now looks like:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-02-38.png" class="aligncenter size-full wp-image-1990" width="640" height="402" alt="Screen Shot 2014-06-28 at 14.02.38" />

The Jun 14 and Jun 20 the database has been re-started and then the difference is &lt; 0.

5\) Let's modify the formula to take care of database restart into the delta computation:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-04-31.png" class="aligncenter size-full wp-image-1991" width="432" height="456" alt="Screen Shot 2014-06-28 at 14.04.31" />

Customize

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-05-19.png" class="aligncenter size-full wp-image-1992" width="583" height="284" alt="Screen Shot 2014-06-28 at 14.05.19" />

Name: "Delta Wait Count" and change ZN(SUM(\[Wait Count\])) - LOOKUP(ZN(SUM(\[Wait Count\])), -1) to max(ZN(SUM(\[Wait Count\])) - LOOKUP(ZN(SUM(\[Wait Count\])), -1),0):

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-07-17.png" class="aligncenter size-full wp-image-1993" width="640" height="252" alt="Screen Shot 2014-06-28 at 14.07.17" />

So that now the graph looks like:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-09-11.png" class="aligncenter size-full wp-image-1994" width="640" height="401" alt="Screen Shot 2014-06-28 at 14.09.11" />

6\) Now we have to "split" those wait count into 2 categories based on the wait\_time\_milli measure coming from dba\_hist\_event\_histogram. Let's say that:

-   "db file sequential read" &lt;= 4 ms are not coming from spindle disks (So from caching, SSD..).
-   "db file sequential read" &gt; 4 ms are coming from spindle disks.

Let's implement this in tableau with a calculated field:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-18-36.png" class="aligncenter size-full wp-image-1997" width="421" height="494" alt="Screen Shot 2014-06-28 at 14.18.36" />

Name: "IO Source" and use this formula:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-21-23.png" class="aligncenter size-full wp-image-1998" width="640" height="250" alt="Screen Shot 2014-06-28 at 14.21.23" />

Feel free to modify this formula according to your environment.

Now take the "IO Source" Dimension and put it into the Color marks:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-23-37.png" class="aligncenter size-full wp-image-1999" width="161" height="277" alt="Screen Shot 2014-06-28 at 14.23.37" />

So that we can now visualize the IO source repartition:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-25-08.png" class="aligncenter size-full wp-image-2000" width="640" height="342" alt="Screen Shot 2014-06-28 at 14.25.08" />

 

<span style="text-decoration:underline;">**Remarks:**</span>

-   Karl Arao presented another example usage of Tableau into [this blog post](http://karlarao.wordpress.com/2012/03/24/fast-analytics-of-awr-top-events/).
-   Should you need to retrieve "db file sequential read" buckets &lt; 1 ms, then you can use [oracle\_trace\_parsing](https://github.com/khailey/oracle_trace_parsing) from Kyle Hailey.

**Update 1**: Example of [oracle\_trace\_parsing](https://github.com/khailey/oracle_trace_parsing) usage into "[Oracle “Physical I/O” ? not always physical](http://www.oraclerealworld.com/oracle-physical-io-not-always-physical/)" blog post.

**Update 2**: Another way to retrieve "db file sequential read" buckets &lt; 1 ms (With external tables this time) into Nikolay Savvinov [blog post](http://savvinov.com/2014/09/08/querying-trace-files/).

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

- Filesystem cache (If any and used)
- Disk Array cache
- SSD
- Spindle Disks
- .....

It could be interesting to visualize&nbsp;the distribution of the IO source:

- Should you migrate from a cached filesystem to ASM (You may need to increase the database cache&nbsp;to put the previous Filesystem cached IOs into the database cache).
- Should you use&nbsp;[Dynamic Tiering](http://flashdba.com/2014/05/23/understanding-disk-caching-and-tiering/)&nbsp;and want to figure out where the&nbsp;IOs come from (SSD, Spindle Disks..).

To do so, I'll use the AWR data coming from the&nbsp;dba\_hist\_event\_histogram view and [Tableau](http://www.tableausoftware.com/public//). I'll also extract the data coming from dba\_hist\_snapshot (to get the begin\_interval\_date time).

[code language="sql"]  
alter session set nls\_date\_format='YYYY/MM/DD HH24:MI:SS';  
alter session set nls\_timestamp\_format='YYYY/MM/DD HH24:MI:SS';

select \* from dba\_hist\_event\_histogram where  
snap\_id \>= (select min(snap\_id) from dba\_hist\_snapshot  
where begin\_interval\_time \>= to\_date ('2014/06/01 00:00','YYYY/MM/DD HH24:MI'))  
and event\_name='db file sequential read';

select \* from dba\_hist\_snapshot where begin\_interval\_time \>= to\_date ('2014/06/01 00:00','YYYY/MM/DD HH24:MI');  
[/code]

As you can see, there is no computation. This is just a simple extraction of the data.

Then I put those data into 2 csv files (awr\_snap\_for\_june.csv and&nbsp;awr\_event\_histogram.csv).

1) Now, launch Tableau and select the csv files and add an inner join between those files:

[![Screen Shot 2014-06-28 at 13.45.19]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-45-19.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-13-45-19.png)

2) Go to the worksheet and put the "begin interval time" dimension into the "column" and change it to an "exact date" (Instead of Year):

[![Screen Shot 2014-06-28 at 13.48.13]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-48-13.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-13-48-13.png)

3) Put the "Wait count" measure into the "Rows" and create a table calculation on it:

[![Screen Shot 2014-06-28 at 13.55.33]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-13-55-33.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-13-55-33.png)

Choose " **difference**" as the "WAIT\_COUNT" field is cumulative and we want to see the **delta** between the AWR's snapshots.

4) My graph now looks like:

[![Screen Shot 2014-06-28 at 14.02.38]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-02-38.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-02-38.png)

The Jun 14 and Jun 20 the database has been re-started and then the difference is \< 0.

5) Let's modify the formula to take care of database restart into the delta computation:

[![Screen Shot 2014-06-28 at 14.04.31]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-04-31.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-04-31.png)

Customize

[![Screen Shot 2014-06-28 at 14.05.19]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-05-19.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-05-19.png)

Name: "Delta Wait Count" and change&nbsp;ZN(SUM([Wait Count])) - LOOKUP(ZN(SUM([Wait Count])), -1) to max(ZN(SUM([Wait Count])) - LOOKUP(ZN(SUM([Wait Count])), -1),0):

[![Screen Shot 2014-06-28 at 14.07.17]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-07-17.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-07-17.png)

So that now the graph looks like:

[![Screen Shot 2014-06-28 at 14.09.11]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-09-11.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-09-11.png)

6) Now we have to "split" those wait count into 2 categories based on the wait\_time\_milli measure coming from dba\_hist\_event\_histogram. Let's say&nbsp;that:

- "db file sequential read" \<= 4 ms are not coming from spindle disks (So from caching, SSD..).
- "db file sequential read" \> 4 ms are coming from spindle disks.

Let's implement this in tableau with a calculated field:

[![Screen Shot 2014-06-28 at 14.18.36]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-18-36.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-18-36.png)

Name: "IO Source" and use this formula:

[![Screen Shot 2014-06-28 at 14.21.23]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-21-23.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-21-23.png)

Feel free to modify this formula according to your environment.

Now take the "IO Source" Dimension and put it into the Color marks:

[![Screen Shot 2014-06-28 at 14.23.37]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-23-37.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-23-37.png)

So that we can now visualize the IO source repartition:

[![Screen Shot 2014-06-28 at 14.25.08]({{ site.baseurl }}/assets/images/screen-shot-2014-06-28-at-14-25-08.png)](https://bdrouvot.files.wordpress.com/2014/06/screen-shot-2014-06-28-at-14-25-08.png)

&nbsp;

**Remarks:**

- Karl Arao presented another example usage of Tableau into [this blog post](http://karlarao.wordpress.com/2012/03/24/fast-analytics-of-awr-top-events/).
- Should you need to retrieve&nbsp;"db file sequential read" buckets \< 1 ms, then you can use [oracle\_trace\_parsing](https://github.com/khailey/oracle_trace_parsing) from Kyle Hailey.

**Update 1** : Example of [oracle\_trace\_parsing](https://github.com/khailey/oracle_trace_parsing) usage into "[Oracle “Physical I/O” ? not always physical](http://www.oraclerealworld.com/oracle-physical-io-not-always-physical/)" blog post.

**Update 2** : Another way to retrieve "db file sequential read" buckets \< 1 ms (With external tables this time) into Nikolay Savvinov [blog post](http://savvinov.com/2014/09/08/querying-trace-files/).


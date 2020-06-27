---
layout: post
title: 'A closer look at ASM rebalance, Part I: Disks have been added'
date: 2014-08-25 19:38:47.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Tableau
tags: []
meta:
  _edit_last: '40807211'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/Pf8zEAM5BQa
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1475549836031867
  _wpas_done_7950430: '1'
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/vSXXLzIKPK
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5909740534192177152&type=U&a=sGNU
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  geo_public: '0'
  _wpas_skip_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/08/25/a-closer-look-at-asm-rebalance-part-i-disks-have-been-added/"
---

This article is the first Part of the "A closer look at ASM rebalance" series:

1.  Part I: Disks have been added.
2.  [Part II: Disks have been dropped.](http://bdrouvot.wordpress.com/2014/09/01/a-closer-look-at-asm-rebalance-part-ii-disks-have-been-dropped/ "A closer look at ASM rebalance, Part II: Disks have been dropped")
3.  [Part III: Disks have been added and dropped (at the same time).](http://bdrouvot.wordpress.com/2014/09/01/a-closer-look-at-asm-rebalance-part-iii-disks-have-been-added-and-dropped-at-the-same-time/ "A closer look at ASM rebalance, Part III: Disks have been added and dropped (at the same time)")

If you are not familiar with ASM rebalance I would suggest first to read those 2 blog posts written by Bane Radulovic:

-   [Rebalancing act](http://asmsupportguy.blogspot.com.au/2011/11/rebalancing-act.html)
-   [When will my rebalance complete](http://asmsupportguy.blogspot.fr/2012/07/when-will-my-rebalance-complete.html)

In this part I want to visualize the rebalance operation (with 3 power values: 2,6 and 11) after disks have been added (no dropped disks yet: It will be for the parts II and III).

To do so, on a 2 nodes Extended Rac Cluster (11.2.0.4), I added 2 disks into the DATA diskgroup (created with an ASM Allocation Unit of 4MB) and launched (connected on +ASM1):

1.  *alter diskgroup DATA rebalance power 2;* (At 11:55 AM).
2.  *alter diskgroup DATA rebalance power 6;* (At 12:05 PM).
3.  *alter diskgroup DATA rebalance power 11;* (At 12:15 PM).

And then I waited until it finished (means *v$asm\_operation* returns no rows for the DATA diskgroup).

Note that 2) and 3) **interrupted** the rebalance in progress and launched a new one with a new power.

During this amount of time I collected the [ASM performance metrics that way](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics") for the **DATA diskgroup only**.

I'll present the results with [Tableau](http://www.tableausoftware.com/public//community) (For each Graph I'll keep the “columns”, “rows” and “marks” shelf into the print screen so that you can reproduce).

<span style="text-decoration:underline;">**Note:**</span> There is no database activity on the Host where the rebalance has been launched.

<span style="text-decoration:underline;">**Here are the results:**</span>

First let's verify that the whole rebalance activity has been done on the +ASM1 instance (As I launched the rebalance operations from it).

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-25-at-18-19-34.png" class="aligncenter size-full wp-image-2215" width="640" height="311" alt="Screen Shot 2014-08-25 at 18.19.34" />

<span style="text-decoration:underline;">We can see:</span>

1.  That all Read and Write rebalance activity has been done on +ASM1 .
2.  That the read throughput is very close to the write throughput on +ASM1.
3.  The impact of the power values (2,6 and 11) on the throughput.

Now I would like to compare the behavior of 2 Sets of Disks: The disks added and the disks that are already part of the DATA diskgroup.

To do so, let's create in Tableau a SET that contains the 2 new disks.

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-27-34.png" class="aligncenter size-full wp-image-2157" width="480" height="514" alt="Screen Shot 2014-08-20 at 21.27.34" />

Let's call it "New Disks"

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-29-42.png" class="aligncenter size-full wp-image-2158" width="482" height="405" alt="Screen Shot 2014-08-20 at 21.29.42" />

So that now we are able to display the ASM metrics **IN** this set (the 2 new disks) and **OUT** this set (the already existing disks means already part of the DATA diskgroup).

I will filter the metrics on ASM1 only (to avoid any "little parasites" coming from ASM2).

<span style="text-decoration:underline;">**Let's visualize the *Reads/s* and *Writes/s* metrics:**</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-25-at-18-26-10.png" class="aligncenter size-full wp-image-2216" width="640" height="312" alt="Screen Shot 2014-08-25 at 18.26.10" />

<span style="text-decoration:underline;">We can see that during the 3 rebalances:</span>

1.  No Reads on the new disks (at least until about 12:40 pm).
2.  Number of *Writes/s* increasing on the new disks depending of the power values.
3.  *Reads/s* and *Writes/s* both increasing on the already existing disks depending of the power values.
4.  As of 12.40 pm, activity on the existing disks while near zero activity on the new ones.
5.  As of 12.40 pm number of *Writes/s &gt;= *Reads/s** on the existing disks (while it was the opposite before).

-   Are 1, 2 and 3 surprising? No.
-   What happened for 4 and 5? I'll answer later on.

<span style="text-decoration:underline;">**Let's visualize the *Kby Read/s* and *Kby Write/s* metrics:**</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-25-at-18-31-59.png" class="aligncenter size-full wp-image-2217" width="640" height="310" alt="Screen Shot 2014-08-25 at 18.31.59" />

<span style="text-decoration:underline;">We can see that during the 3 rebalances:</span>

1.  No *Kby Read/s* on the new disks.
2.  Number of *Kby Write/s* increasing on the new disks depending of the power values.
3.  *Kby Read/s* and *Kby Write/s* both increasing on the existing disks depending of the power values.
4.  As of 12.40 pm, activity on the existing disks while no activity on the new ones.
5.  As of 12.40 pm same amount of *Kby Read/s* and *Kby Write/s* on the existing disks (while it was not the case before).

-   Are 1, 2 and 3 surprising? No.
-   What happened for 4 and 5? I'll answer later on.

<span style="text-decoration:underline;">**Let's visualize the Average By/*Read* and Average B*y/Write *metrics:**</span>

**Important remark regarding the averages computation/display: **The By*/Read* and *By/Write* measures **depend on the number of reads**. So the averages have to be calculated using **Weighted Averages**.

Let’s create the calculated field in Tableau for the *By/Read* Weighted Average:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-56-49.png" class="aligncenter size-full wp-image-2161" width="640" height="229" alt="Screen Shot 2014-08-20 at 21.56.49" />

The same has to be done for the *By/Write* Weighted Average.

Let's see the result:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-25-at-18-38-07.png" class="aligncenter size-full wp-image-2218" width="640" height="309" alt="Screen Shot 2014-08-25 at 18.38.07" />

<span style="text-decoration:underline;">We can see:</span>

1.  The Avg *By/Write* on the new disks is about the same (about 1MB) whatever the power value is (before 12:40 pm).
2.  The Avg *By/Write* tends to increase with the power on the already existing disks.
3.  The Avg *By/Read* on the existing disks is about the same (about 1MB) whatever the power value is.

-   Is 1 surprising? No.
-   Is 2 surprising? Yes (at least for me).
-   Is 3 surprising? No.

Now that we have seen all those metrics, we can ask:

<span style="text-decoration:underline;">**Q1: So what the hell happened at 12:40 pm?**</span>

Let's check the *alert\_+ASM1.log* file at that time:

    Mon Aug 25 12:15:44 2014
    ARB0 started with pid=33, OS id=1187132
    NOTE: assigning ARB0 to group 4/0x1e089b59 (DATA) with 11 parallel I/Os
    Mon Aug 25 12:15:47 2014
    NOTE: Attempting voting file refresh on diskgroup DATA
    NOTE: Refresh completed on diskgroup DATA. No voting file found.
    cellip.ora not found.
    Mon Aug 25 12:39:52 2014
    NOTE: GroupBlock outside rolling migration privileged region
    NOTE: requesting all-instance membership refresh for group=4
    Mon Aug 25 12:40:03 2014
    GMON updating for reconfiguration, group 4 at 372 for pid 35, osid 1225810
    NOTE: group DATA: updated PST location: disk 0014 (PST copy 0)
    NOTE: group DATA: updated PST location: disk 0015 (PST copy 1)
    Mon Aug 25 12:40:03 2014
    NOTE: group 4 PST updated.
    Mon Aug 25 12:40:03 2014
    NOTE: membership refresh pending for group 4/0x1e089b59 (DATA)
    GMON querying group 4 at 373 for pid 18, osid 67864
    SUCCESS: refreshed membership for 4/0x1e089b59 (DATA)
    NOTE: Attempting voting file refresh on diskgroup DATA
    NOTE: Refresh completed on diskgroup DATA. No voting file found.
    Mon Aug 25 12:45:24 2014
    NOTE: F1X0 copy 2 relocating from 18:44668 to 18:20099 for diskgroup 4 (DATA)
    Mon Aug 25 12:53:49 2014
    NOTE: stopping process ARB0
    SUCCESS: rebalance completed for group 4/0x1e089b59 (DATA)

We can see that the ASM rebalance started the compacting phase (See Bane Radulovic's [blog post](http://asmsupportguy.blogspot.fr/2012/07/when-will-my-rebalance-complete.html) for more details about the ASM rebalances phases).

<span style="text-decoration:underline;">**Q2: The ASM Allocation Unit size is 4MB and the Avg By/Read is stucked to 1MB,why?**</span>

I guess this is somehow related to the [max\_sectors\_kb and max\_hw\_sectors\_kb SYSFS parameters](http://martincarstenbach.wordpress.com/2013/07/03/increasing-the-maximum-io-size-in-linux/). It will be the subject of [another post](http://bdrouvot.wordpress.com/2014/09/03/asm-rebalance-why-is-the-avg-byread-equal-to-1mb-while-the-allocation-unit-is-4mb/ "ASM Rebalance: Why is the avg By/Read equal to 1MB while the allocation unit is 4MB?").

<span style="text-decoration:underline;">**Two remarks before to conclude:**</span>

1.  The ASM rebalance activity is not recorded into the *v$asm\_disk\_iostat* view*. *It is recorded into the *v$asm\_disk\_stat* view. So, if you are using the [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm_metrics") utility, you have to change the *asm\_feature\_version *variable to a value &gt; your ASM instance version.
2.  I tested with *compatible.asm* set to 10.1 and 11.2.0.2 and observed the same behavior for all those metrics.

<span style="text-decoration:underline;">**Conclusion of Part I:**</span>

-   We visualized that the compacting phase of the rebalance operation generates much more activity on the existing disks compare to near zero activity on the new disks.
-   We visualized that the compacting phase of the rebalance operation generates the same amount of *Kby Read/s* and *Kby Write/s* on the existing disks (while it was not the case before).
-   We visualized that during the compacting phase the number of *Writes/s &gt;= *Reads/s** on the existing disks (while it was the opposite before).
-   We visualized that the *Avg By/Read* does not exceed 1MB on the existing disks (while the ASM allocation Unit has been set to 4MB on my diskgroup).
-   We visualized that the Avg *By/Write* tends to increase with the power on the already existing disks (quite surprising to me).
-   I'll update this post with ASM 12c results as soon as I can (if something new needs to be told).

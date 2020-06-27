---
layout: post
title: 'A closer look at ASM rebalance, Part III: Disks have been added and dropped
  (at the same time)'
date: 2014-09-01 21:06:38.000000000 +02:00
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
  geo_public: '0'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1480525965534254
  _wpas_done_7950430: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/dqfig3rrNpm
  _wpas_done_8482624: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/Vp77F2Cj3j
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5912299334728196096&type=U&a=ZPwl
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/09/01/a-closer-look-at-asm-rebalance-part-iii-disks-have-been-added-and-dropped-at-the-same-time/"
---
This article is the third&nbsp;Part of the "A closer look at ASM rebalance" series:

1. [Part I: Disks have been added.](http://bdrouvot.wordpress.com/2014/08/25/a-closer-look-at-asm-rebalance-part-i-disks-have-been-added/ "A closer look at ASM rebalance, Part I: Disks have been added")
2. [Part II: Disks have been dropped.](http://bdrouvot.wordpress.com/2014/09/01/a-closer-look-at-asm-rebalance-part-ii-disks-have-been-dropped/ "A closer look at ASM rebalance, Part II: Disks have been dropped")
3. Part III: Disks have been added and dropped (at the same time).

If you are not familiar with ASM rebalance I would suggest first to read those 2 blog posts written by&nbsp;Bane Radulovic:

- [Rebalancing act](http://asmsupportguy.blogspot.com.au/2011/11/rebalancing-act.html)
- [When will my rebalance complete](http://asmsupportguy.blogspot.fr/2012/07/when-will-my-rebalance-complete.html)

In this part III I want to visualize the rebalance operation (with 3 power values: 2,6 and 11) after&nbsp;disks have been added and dropped (at the same time).

To do so, on a&nbsp;2 nodes Extended Rac Cluster (11.2.0.4),&nbsp;I added 2 disks and dropped&nbsp;2 disks (with a **single** command) into the DATA diskgroup (created with an ASM Allocation Unit of 4MB) and launched (connected on +ASM1):

1. _alter diskgroup DATA rebalance power 2;_ (At 02:11&nbsp;PM).
2. _alter diskgroup DATA rebalance power 6;_ (At 02:24&nbsp;PM).
3. _alter diskgroup DATA rebalance power 11;_&nbsp;(At 02:34&nbsp;PM).

And then I waited until it finished (means _v$asm\_operation_ returns no rows for the DATA diskgroup).

Note that 2) and 3) **interrupted** the rebalance in progress and launched a new one with a new power.

During this amount of time I collected the [ASM performance metrics that way](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics")&nbsp;for the **DATA diskgroup only**.

I'll present the results with [Tableau](http://www.tableausoftware.com/public//community) (For each Graph I'll keep the “columns”, “rows” and “marks” shelf into the print screen so that you can reproduce).

**Note:** There is no database activity on the Host where the rebalance has been launched.

**Here are the results:**

First let's verify that the whole rebalance activity has been done on the +ASM1 instance (As I launched the rebalance operations&nbsp;from&nbsp;it).

[![Screen Shot 2014-09-01 at 20.29.42]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-20-29-42.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-20-29-42.png)

We can see:

1. That all Read and Write rebalance activity has been done on +ASM1 .
2. That the read throughput is very close to the write throughput on +ASM1.
3. The impact of the power values&nbsp;(2,6 and 11) on the throughput.

Now I would like to compare the behavior of 3 Sets of Disks: The disks that have been dropped, the disks that have been added and the other existing disks into&nbsp;the DATA diskgroup.

To do so, let's create in Tableau 3&nbsp;groups:

[![Screen Shot 2014-09-01 at 21.00.06]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-21-00-06.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-21-00-06.png)

Let's call it "3 Groups"

[![Screen Shot 2014-09-01 at 20.58.24]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-20-58-24.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-20-58-24.png)

So that now we&nbsp;are&nbsp;able to display the ASM metrics for those 3 sets of disks.

I will filter the metrics on ASM1 only (to avoid any "little parasites" coming from ASM2).

**Let's visualize&nbsp;the _Reads/s_ and _Writes/s_ metrics:**

[![Screen Shot 2014-09-01 at 21.03.04]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-21-03-04.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-21-03-04.png)

We can see that during the 3 rebalances:

1. No writes&nbsp;on the dropped disks.
2. No reads on the new disks.
3. Number of _Reads/s_ increasing on the dropped&nbsp;disks depending of the power values.
4. Number of _Writes/s_ increasing on the new&nbsp;disks depending of the power values.
5. _Reads/s_ and _Writes/s_ both&nbsp;increasing on the other disks depending of the power values.
6. As of 03:06 PM, no activity on the dropped and new disks while there is still activity on the other disks.

- Are 1, 2, 3, 4 and 5 surprising? No.
- What happened for 6? I'll answer later on.

**Let's visualize&nbsp;the _Kby&nbsp;Read/s_ and _Kby&nbsp;Write/s_ metrics:**

[![Screen Shot 2014-09-01 at 21.12.44]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-21-12-44.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-21-12-44.png)

We can see that during the 3 rebalances:

1. No _Kby Write/s_ on the dropped&nbsp;disks.
2. No&nbsp;_Kby Read/s_ on the new disks.
3. Number of _Kby Read/s_ increasing on the dropped&nbsp;disks depending of the power values.
4. Number of _Kby Write/s_ increasing on the new&nbsp;disks depending of the power values.
5. _Kby Read/s_ and _Kby Write/s_ both&nbsp;increasing on the other disks depending of the power values.
6. _Kby Read/s_ and _Kby Write/s_ are&nbsp;very close on the other disks (It was not the case into the [Part I](http://bdrouvot.wordpress.com/2014/08/25/a-closer-look-at-asm-rebalance-part-i-disks-have-been-added/ "A closer look at ASM rebalance, Part I: Disks have been added")).
7. As of 03:06 PM, no activity on the dropped and new disks while there is still activity on the other disks.

- Are 1, 2, 3, 4, 5 and 6 surprising? No.
- What happened for 7? I'll answer later on.

**Let's visualize&nbsp;the Average By/_Read_&nbsp;and Average&nbsp;B_y/Write&nbsp;_metrics:**

**Important remark regarding the&nbsp;averages computation/display:&nbsp;** The By_/Read_ and _By/Write_&nbsp;measures **depend on the number of reads**. So the averages have to be calculated using **Weighted Averages**.

Let’s create the calculated field in Tableau for the _By/Read_ Weighted Average:

[![Screen Shot 2014-08-20 at 21.56.49]({{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-56-49.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-20-at-21-56-49.png)

The same has to be done for the _By/Write_&nbsp;Weighted Average.

Let's see the result:

[![Screen Shot 2014-09-01 at 21.22.10]({{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-21-22-10.png)](https://bdrouvot.files.wordpress.com/2014/09/screen-shot-2014-09-01-at-21-22-10.png)

We can see:

1. The Avg _By/Read_&nbsp;on the dropped&nbsp;disks is about the same (about 1MB) whatever the power value is.
2. The Avg _By/Write_&nbsp;on the new&nbsp;disks is about the same (about 1MB) whatever the power value is.
3. The Avg _By/Read_&nbsp;and _Avg By/Write_ on the other&nbsp;disks is about the same (about 1MB) whatever the power value is.

- Are 1,2 and 3 surprising? No for the behaviour,Yes (at least for me) for the 1MB value as the ASM allocation unit is 4MB.

Now that we have seen all those metrics, we can ask:

**Q1: So what the hell happened at 03:06 pm?**

Let’s check the _alert\_+ASM1.log_ file at that time:

```
Mon Aug 25 **15:06:13** 2014 NOTE: membership refresh pending for group 4/0x1e089b59 (DATA) GMON querying group 4 at 396 for pid 18, osid 67864 GMON querying group 4 at 397 for pid 18, osid 67864 NOTE: Disk DATA\_0006 in mode 0x0 marked for de-assignment NOTE: Disk DATA\_0007 in mode 0x0 marked for de-assignment SUCCESS: refreshed membership for 4/0x1e089b59 (DATA) NOTE: Attempting voting file refresh on diskgroup DATA NOTE: Refresh completed on diskgroup DATA. No voting file found. Mon Aug 25 **15:07:16** 2014 NOTE: stopping process ARB0 SUCCESS: rebalance completed for group 4/0x1e089b59 (DATA)
```

We can see that the ASM rebalance started the compacting&nbsp;phase (See Bane Radulovic’s&nbsp;[blog post](http://asmsupportguy.blogspot.fr/2012/07/when-will-my-rebalance-complete.html) for more details about the ASM rebalances phases).

**Q2: The ASM Allocation Unit size is 4MB and the Avg By/Read is stucked to 1MB,why?**

I don't have the answer yet,&nbsp;it will be the subject of [another post](http://bdrouvot.wordpress.com/2014/09/03/asm-rebalance-why-is-the-avg-byread-equal-to-1mb-while-the-allocation-unit-is-4mb/ "ASM Rebalance: Why is the avg By/Read equal to 1MB while the allocation unit is 4MB?").

**Two remarks before to conclude:**

1. The ASM rebalance activity is not recorded into the&nbsp;_v$asm\_disk\_iostat_ view_.&nbsp;_It is recorded into the _v$asm\_disk\_stat_ view. So, if you are using the [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm\_metrics")&nbsp;utility, you have to change the&nbsp;_asm\_feature\_version&nbsp;_variable to a value \> your ASM instance version.
2. I tested with _compatible.asm_&nbsp;set to 10.1 and 11.2.0.2 and observed the same behaviour for all those metrics.

**Conclusion of Part III:**

- Nothing surprising except (at least for me) that the _Avg By/Read_ is stucked to 1MB (While the allocation unit is 4MB).
- We visualized that the compacting phase of the rebalance operation generates much more activity on the other&nbsp;disks compare to near zero activity on the dropped and new disks.
- I'll update this post with ASM 12c results as soon as I can (if something new needs to be told).

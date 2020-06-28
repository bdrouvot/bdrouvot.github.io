---
layout: post
title: 'A closer look at ASM rebalance, Part II: Disks have been dropped'
date: 2014-09-01 20:45:53.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Tableau
tags: [ASM, oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _wpas_done_7950430: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1480520792201438
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/HSXgpB89row
  _wpas_done_8482624: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/MEC8hrvhjm
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5912294117974511616&type=U&a=D3Fc
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
permalink: "/2014/09/01/a-closer-look-at-asm-rebalance-part-ii-disks-have-been-dropped/"
---

This article is the second Part of the "A closer look at ASM rebalance" series:

1.  [Part I: Disks have been added.](http://bdrouvot.wordpress.com/2014/08/25/a-closer-look-at-asm-rebalance-part-i-disks-have-been-added/ "A closer look at ASM rebalance, Part I: Disks have been added")
2.  Part II: Disks have been dropped.
3.  [Part III: Disks have been added and dropped (at the same time).](http://bdrouvot.wordpress.com/2014/09/01/a-closer-look-at-asm-rebalance-part-iii-disks-have-been-added-and-dropped-at-the-same-time/ "A closer look at ASM rebalance, Part III: Disks have been added and dropped (at the same time)")

If you are not familiar with ASM rebalance I would suggest first to read those 2 blog posts written by Bane Radulovic:

-   [Rebalancing act](http://asmsupportguy.blogspot.com.au/2011/11/rebalancing-act.html)
-   [When will my rebalance complete](http://asmsupportguy.blogspot.fr/2012/07/when-will-my-rebalance-complete.html)

In this part II I want to visualize the rebalance operation (with 3 power values: 2,6 and 11) after disks have been dropped.

To do so, on a 2 nodes Extended Rac Cluster (11.2.0.4), I dropped 2 disks into the DATA diskgroup (created with an ASM Allocation Unit of 4MB) and launched (connected on +ASM1):

1.  *alter diskgroup DATA rebalance power 2;* (At 09:09 AM).
2.  *alter diskgroup DATA rebalance power 6;* (At 09:19 AM).
3.  *alter diskgroup DATA rebalance power 11;* (At 09:29 AM).

And then I waited until it finished (means *v$asm\_operation* returns no rows for the DATA diskgroup).

Note that 2) and 3) **interrupted** the rebalance in progress and launched a new one with a new power.

During this amount of time I collected the [ASM performance metrics that way](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics") for the **DATA diskgroup only**.

I'll present the results with [Tableau](http://www.tableausoftware.com/public//community) (For each Graph I'll keep the “columns”, “rows” and “marks” shelf into the print screen so that you can reproduce).

<span style="text-decoration:underline;">**Note:**</span> There is no database activity on the Host where the rebalance has been launched.

<span style="text-decoration:underline;">**Here are the results:**</span>

First let's verify that the whole rebalance activity has been done on the +ASM1 instance (As I launched the rebalance operations from it).

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-31-at-18-50-48.png" class="aligncenter size-full wp-image-2243" width="640" height="308" alt="Screen Shot 2014-08-31 at 18.50.48" />

<span style="text-decoration:underline;">We can see:</span>

1.  That all Read and Write rebalance activity has been done on +ASM1 .
2.  That the read throughput is very close to the write throughput on +ASM1.
3.  The impact of the power values (2,6 and 11) on the throughput.

Now I would like to compare the behavior of 2 Sets of Disks: The disks that have been dropped and the disks that will remain into the DATA diskgroup.

To do so, let's create in Tableau a SET that contains the 2 dropped disks.

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-27-34.png" class="aligncenter size-full wp-image-2157" width="480" height="514" alt="Screen Shot 2014-08-20 at 21.27.34" />

Let's call it "Dropped Disks"

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-31-at-18-53-04.png" class="aligncenter size-full wp-image-2244" width="472" height="401" alt="Screen Shot 2014-08-31 at 18.53.04" />

So that now we are able to display the ASM metrics **IN** this set (the 2 dropped disks) and **OUT** this set (the disks that will remain into the DATA diskgroup).

I will filter the metrics on ASM1 only (to avoid any "little parasites" coming from ASM2).

<span style="text-decoration:underline;">**Let's visualize the *Reads/s* and *Writes/s* metrics:**</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-31-at-18-57-08.png" class="aligncenter size-full wp-image-2246" width="640" height="310" alt="Screen Shot 2014-08-31 at 18.57.08" />

<span style="text-decoration:underline;">We can see that during the 3 rebalances:</span>

1.  No writes on the dropped disks.
2.  Number of *Reads/s* increasing on the dropped disks depending of the power values.
3.  *Reads/s* and *Writes/s* both increasing on the remaining disks depending of the power values.

-   Are 1, 2 and 3 surprising? No.

<span style="text-decoration:underline;">**Let's visualize the *Kby Read/s* and *Kby Write/s* metrics:**</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-31-at-19-00-33.png" class="aligncenter size-full wp-image-2247" width="640" height="307" alt="Screen Shot 2014-08-31 at 19.00.33" />

<span style="text-decoration:underline;">We can see that during the 3 rebalances:</span>

1.  No *Kby Write/s* on the dropped disks.
2.  Number of *Kby Read/s* increasing on the dropped disks depending of the power values.
3.  *Kby Read/s* and *Kby Write/s* both increasing on the remaining disks depending of the power values.

Are 1, 2 and 3 surprising? No.

<span style="text-decoration:underline;">**Let's visualize the Average By/*Read* and Average B*y/Write *metrics:**</span>

**Important remark regarding the averages computation/display: **The By*/Read* and *By/Write* measures **depend on the number of reads**. So the averages have to be calculated using **Weighted Averages**.

Let’s create the calculated field in Tableau for the *By/Read* Weighted Average:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-20-at-21-56-49.png" class="aligncenter size-full wp-image-2161" width="640" height="229" alt="Screen Shot 2014-08-20 at 21.56.49" />

The same has to be done for the *By/Write* Weighted Average.

Let's see the result:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-09-01-at-21-33-48.png" class="aligncenter size-full wp-image-2262" width="640" height="306" alt="Screen Shot 2014-09-01 at 21.33.48" />

<span style="text-decoration:underline;">We can see:</span>

1.  The Avg *By/Read* on the dropped disks is about the same (about 1MB) whatever the power value is.
2.  The Avg *By/Read* and *Avg By/Write* on the remaining disks is about the same (about 1MB) whatever the power value is.

-   Are 1 and 2 surprising? No for the behaviour, Yes (at least for me) for the 1MB value as the ASM allocation unit is 4MB.

Now that we have seen all those metrics, we can ask:

<span style="text-decoration:underline;">**Q1: The ASM Allocation Unit size is 4MB and the Avg By/Read is stucked to 1MB,why?**</span>

I don't have the answer yet, it will be the subject of [another post](http://bdrouvot.wordpress.com/2014/09/03/asm-rebalance-why-is-the-avg-byread-equal-to-1mb-while-the-allocation-unit-is-4mb/ "ASM Rebalance: Why is the avg By/Read equal to 1MB while the allocation unit is 4MB?").

<span style="text-decoration:underline;">**Two remarks before to conclude:**</span>

1.  The ASM rebalance activity is not recorded into the *v$asm\_disk\_iostat* view*. *It is recorded into the *v$asm\_disk\_stat* view. So, if you are using the [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm_metrics") utility, you have to change the *asm\_feature\_version *variable to a value &gt; your ASM instance version.
2.  I tested with *compatible.asm* set to 10.1 and 11.2.0.2 and observed the same behaviour for all those metrics.

<span style="text-decoration:underline;">**Conclusion of Part II:**</span>

-   Nothing surprising except (at least for me) that the *Avg By/Read* is stucked to 1MB (While the allocation unit is 4MB).
-   I'll update this post with ASM 12c results as soon as I can (if something new needs to be told).

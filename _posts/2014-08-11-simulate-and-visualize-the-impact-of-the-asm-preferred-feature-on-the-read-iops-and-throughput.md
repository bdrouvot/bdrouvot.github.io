---
layout: post
title: Simulate and Visualize the impact of the ASM preferred feature on the read
  IOPS and throughput
date: 2014-08-11 16:27:21.000000000 +02:00
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
  publicize_facebook_url: https://facebook.com/1466080096978841
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5904618923038380032&type=U&a=K0B1
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/gG7JpocFJLU
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/iceNY0cq9c
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/08/11/simulate-and-visualize-the-impact-of-the-asm-preferred-feature-on-the-read-iops-and-throughput/"
---

Suppose that you decided to put the ASM preferred feature in place because you observed that the read latency is **too high on the farthest disk array** (You can find how you can lead to this conclusion with the use case 3 into [this post](http://bdrouvot.wordpress.com/2014/07/12/asm-performance-metrics-visualization-use-cases/ "ASM performance metrics visualization: Use cases")).

<span style="text-decoration:underline;">So, you want to enable the ASM preferred read feature so that:</span>

1.  The +ASM1 instance "prefers" to read from the "WIN" failgroup.
2.  The +ASM2 instance "prefers" to read from the "JMO" failgroup.

<span style="text-decoration:underline;">But doing so **may have an impact on the number of read IOPS and the throughput** repartition per host/disk array because:</span>

1.  The "previous" ASM1 to JMO reads will now be done on the "WIN" array.
2.  The "previous" ASM2 to WIN reads will now be done on the "JMO" array.

Of course, the total number of read operations and throughput will not change, but the **repartition across the failgroup (disk array) may change** with the ASM preferred read feature in place.

<span style="text-decoration:underline;">**Question:**</span>

-   Is the architecture able to deal with this new read repartition?

<span style="text-decoration:underline;">To answer this question I will:</span>

1.  Collect the ASM metrics during a certain amount of time (without the ASM preferred read in place) and produce a csv file as described [here](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics").
2.  Visualize the ASM metrics with [Tableau](http://www.tableausoftware.com/public//community) and **simulate** the impact of the preferred read feature on the read IOPS and the throughput repartition.

Once the csv file is ready (means you collected a representative workload), let's check what the current workload is (**Without** the ASM preferred read in place).

<span style="text-decoration:underline;">For the *Kby Read/s* measure*:*</span>

We can visualize it that way with Tableau (I keep the “columns”, “rows” and “marks” shelf into the print screen so that you can reproduce).

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-10-at-18-45-03.png" class="aligncenter size-full wp-image-2104" width="640" height="362" alt="Screen Shot 2014-08-10 at 18.45.03" />

<span style="text-decoration:underline;">For the *Reads/s* measure*:*</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-07-01.png" class="aligncenter size-full wp-image-2116" width="640" height="338" alt="Screen Shot 2014-08-11 at 11.07.01" />

Now, **what If** we implement the ASM preferred feature? **What would** be the impact on the read IOPS and the throughput repartition?

To simulate and visualize the impact, let's create this "New FG for Read operations" calculated field:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-10-01.png" class="aligncenter size-full wp-image-2118" width="640" height="217" alt="Screen Shot 2014-08-11 at 11.10.01" />

**Basically it simulates the ASM preferred Read in place by assigning the failgroup per ASM instances.**

Now, let's simulate and visualize the impact of the ASM preferred read feature (should it be implemented) using the same csv file and this calculated field as dimension.

<span style="text-decoration:underline;">For the *Kby Read/s* measure*:*</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-12-56.png" class="aligncenter size-full wp-image-2119" width="640" height="340" alt="Screen Shot 2014-08-11 at 11.12.56" />

Note that the throughput repartition would not be the same and that the peak are higher (&gt; 200 Mo/s compare to about 130 Mo/s without the ASM preferred read).

<span style="text-decoration:underline;">For the *Reads/s* measure*:*</span>

<img src="{{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-14-31.png" class="aligncenter size-full wp-image-2120" width="640" height="340" alt="Screen Shot 2014-08-11 at 11.14.31" />

Note that the read IOPS repartition would not be the same and that the peak on the WIN failgroup is higher (about 8000 Reads/s compare to about 5000 Reads/s without the ASM preferred read).

Now you can check (with your Systems and Storage administrators) if your current architecture would be able to deal with this new repartition.

<span style="text-decoration:underline;">**Remarks:**</span>

-   ASM is not performing any reads for the database, it records metrics for the database instances that it is servicing.

<!-- -->

-   I would not suggest to use the [Flex ASM feature](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) with the ASM preferred read because [the preferred read feature is broken with Flex ASM 12c (12.1) in place](https://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/ "Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !").

<span style="text-decoration:underline;">**Conclusion:**</span>

We have been able to simulate and visualize the impact of the ASM preferred read feature on the read IOPS and the throughput repartition without actually implementing it.

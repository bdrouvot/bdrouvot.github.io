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
tags: []
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
Suppose that you decided to put the ASM preferred feature in place because you observed&nbsp;that the read latency is **too high on the&nbsp;farthest disk array** (You can find how you can lead to this conclusion&nbsp;with&nbsp;the use case 3 into [this post](http://bdrouvot.wordpress.com/2014/07/12/asm-performance-metrics-visualization-use-cases/ "ASM performance metrics visualization: Use cases")).

So, you want to enable the ASM preferred read feature so that:

1. The +ASM1 instance "prefers" to read from the "WIN" failgroup.
2. The +ASM2 instance "prefers" to read from the "JMO" failgroup.

But doing so **may have an impact on the number of read IOPS and the throughput** repartition per host/disk array because:

1. The "previous" ASM1 to JMO reads will now be done on the "WIN" array.
2. The "previous" ASM2 to WIN reads will now be done on the "JMO" array.

Of course, the total number of read operations and throughput will not change, but the **repartition across the failgroup (disk array) may change** with the ASM preferred read feature in place.

**Question:**

- Is the architecture able to deal with this new read repartition?

To answer this question I will:

1. Collect the ASM metrics during a certain amount of time (without the ASM preferred read in place) and produce a csv file as described [here](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics").
2. Visualize the ASM metrics with&nbsp;[Tableau](http://www.tableausoftware.com/public//community)&nbsp;and **simulate** &nbsp;the impact of the preferred read feature on the read IOPS and the throughput repartition.

Once the csv file is ready (means you collected a representative workload), let's check what the current workload is ( **Without** the ASM preferred read in place).

For the&nbsp;_Kby Read/s_ measure_:_

We can visualize it that way with Tableau (I keep the “columns”, “rows” and “marks” shelf into the print screen so that you can reproduce).

[![Screen Shot 2014-08-10 at 18.45.03]({{ site.baseurl }}/assets/images/screen-shot-2014-08-10-at-18-45-03.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-10-at-18-45-03.png)

For the&nbsp;_Reads/s_ measure_:_

[![Screen Shot 2014-08-11 at 11.07.01]({{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-07-01.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-11-at-11-07-01.png)We can see the read IOPS and the throughput repartition by failgroup and ASM instances. We can see that the read IOPS and&nbsp;the throughput are&nbsp;equally distributed over the Failgroups (It is the expected behaviour without the ASM preferred read in place).

Now, **what If** we implement the ASM preferred feature? **What would** be the impact on the read IOPS and the throughput repartition?

To simulate and visualize the impact, let's create this "New FG for Read operations" calculated field:

[![Screen Shot 2014-08-11 at 11.10.01]({{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-10-01.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-11-at-11-10-01.png)

**Basically it simulates&nbsp;the ASM preferred Read in place by assigning the failgroup per ASM instances.**

Now, let's simulate and visualize the impact of the ASM preferred read feature (should it be implemented) using the same csv file and this calculated field as dimension.

For the&nbsp;_Kby Read/s_ measure_:_

[![Screen Shot 2014-08-11 at 11.12.56]({{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-12-56.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-11-at-11-12-56.png)

Note that the throughput repartition would not be the same and that the peak are higher (\> 200 Mo/s compare to about 130 Mo/s without the ASM preferred read).

For the&nbsp;_Reads/s_ measure_:_

[![Screen Shot 2014-08-11 at 11.14.31]({{ site.baseurl }}/assets/images/screen-shot-2014-08-11-at-11-14-31.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-11-at-11-14-31.png)

Note that the read IOPS repartition would not be the same and that the peak on the WIN failgroup is&nbsp;higher (about 8000&nbsp;Reads/s compare to about 5000 Reads/s without the ASM preferred read).

Now you can check (with your Systems and Storage administrators) if your current architecture would be able to deal with this new repartition.

**Remarks:**

- ASM is not performing any reads for the database, it records metrics for&nbsp;the database instances that it is servicing.

- I would not suggest to use the&nbsp;[Flex ASM feature](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) with the ASM preferred read because [the preferred read feature is broken with Flex ASM 12c (12.1) in place](https://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/ "Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !").

**Conclusion:**

We have been able to simulate and visualize the impact of the ASM preferred read feature on the read IOPS and the throughput repartition without actually implementing it.


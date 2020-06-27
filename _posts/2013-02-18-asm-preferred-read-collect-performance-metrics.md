---
layout: post
title: 'ASM Preferred Read: Collect performance metrics'
date: 2013-02-18 08:00:34.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
- Perl Scripts
- ToolKit
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/02/18/asm-preferred-read-collect-performance-metrics/"
---

The purpose of this post is not to explain the ASM Preferred Read feature or the way to put it in place (for such purpose you can have a look to this [oracle-base post](http://www.oracle-base.com/articles/11g/asm-enhancements-11gr1.php) or [Christian Bilien's one).](http://christianbilien.wordpress.com/2007/12/24/11g-asm-preferred-reads-for-rac-extended-clusters/)

The purpose is to give a way to see this feature in action and collect related performance metrics. To do this:

-   I set asm\_preferred\_read\_failure\_groups to DATA.WIN on Instance +ASM1
-   I set asm\_preferred\_read\_failure\_groups to DATA.JMO on Instance +ASM2
-   I use [Kevin Closson's SLOB Kit](http://kevinclosson.wordpress.com/2012/02/06/introducing-slob-the-silly-little-oracle-benchmark/) to generate I/O on the database
-   I use my asmiostat utility included into [real\_time.pl](http://bdrouvot.wordpress.com/real_time/ "real_time") (see [this post](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") for more information) with a filter on the DATA Diskgroup (*-dg=data*) and showing metrics at the Instances and Failgroups level (*-show=inst,fg*)

<span style="text-decoration:underline;color:#000080;">First test: </span>

Let's run SLOB to generate IOs Read from a database located on the same Host as the +ASM1 Instance. The result of "*./real\_time.pl -type=asmiostat -show=inst,fg -dg=data*" is the following:

<img src="{{ site.baseurl }}/assets/images/asm_prefer1.png" class="aligncenter size-full wp-image-730" width="620" height="285" alt="asm_prefer1" />

As you can see the Read IOs come from the WIN failgroup (as expected). You also get the performance metrics of the failgroup.

<span style="text-decoration:underline;color:#000080;">Second test:</span>

Let's run SLOB to generate IOs Read from a database located on the same Host as the +ASM2 Instance. The result of "*./real\_time.pl -type=asmiostat -show=inst,fg -dg=data*" is the following:

<img src="{{ site.baseurl }}/assets/images/asm_prefer2.png" class="aligncenter size-full wp-image-731" width="620" height="187" alt="asm_prefer2" />

As you can see the Read IOs come from the JMO failgroup (as expected). You also get the performance metrics of the failgroup.

<span style="text-decoration:underline;"><span style="color:#000000;text-decoration:underline;">**Conclusion:**</span></span>

Thanks to the *-show* option of my asmiostat utility I provided a simple way to collect in real time the performance metrics related to your ASM preferred read configuration. (You can also check if this is working as expected that is to say IOs coming from the right failgroup)

To get the asmiostat utility included into the [real\_time.pl](http://bdrouvot.wordpress.com/real_time/ "real_time") script:  Click on the link, and then on the view source button and then copy/paste the source code. You can also download the script from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit?pli=1) to avoid copy/paste (click on the link)

<span style="text-decoration:underline;color:#000000;">**Updates:**</span>** **

1.  Check how it can be useful for Exadata into this [post](http://bdrouvot.wordpress.com/2013/02/21/exadata-storage-cells-io-performance-metrics-and-io-distribution-with-db-servers/ "Exadata: Storage Cells IO performance metrics and IO distribution with DB servers").
2.  SLOB update 2 has been released since this post. Check how we can use it into this [post](http://bdrouvot.wordpress.com/2013/05/05/asm-preferred-read-collect-performance-metrics-thanks-to-my-amsiostat-utility-and-slob-2/ "ASM Preferred Read: Collect performance metrics thanks to my amsiostat utility and SLOB 2").
3.  The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

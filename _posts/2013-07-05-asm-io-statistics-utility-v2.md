---
layout: post
title: ASM I/O Statistics Utility V2
date: 2013-07-05 13:14:32.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ASM
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
permalink: "/2013/07/05/asm-io-statistics-utility-v2/"
---

Some days ago I wrote about a side effect of Flex ASM 12c (12.1) that I called "[unpreferred read](http://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/ "Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !")". While I was writing this post I thought that the side effect demonstration will be even more clear if my asmiostat utility could display database Instances as well.

<span style="text-decoration:underline;">This is done with my asmiostat utility V2 which provides those new features:</span>

-   Ability to display database instances (as of 11gr1).
-   Ability to sort based on the number of reads.
-   Ability to sort based on the number of writes.

<span style="text-decoration:underline;">The following metrics are still collected:</span>

-   Reads/s: Number of read per second.
-   KbyRead/s: Kbytes read per second.
-   Avg ms/Read: ms per read in average.
-   AvgBy/Read: Average Bytes per read.
-   Same metrics are provided for Write Operations.

<span style="text-decoration:underline;">Of course the old features remain (see [this post](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") for more details about the previous features):</span>

-   Ability to display/aggregate/filter following your needs on ASM instances, diskgroup, failgroup and disks (And now on database instances as well).
-   Ability to display Exadata Cells IPs instead of ASM Failgroup.

<span style="text-decoration:underline;">Let's have a look to 2 examples using the V2 features:</span>

<span style="text-decoration:underline;">**First one: **</span>I want to know which database Instance is generating most of the Read IO requests per ASM instance, and I also want to see the performance metrics.

Fine, let's launch my utility that way:

    ./real_time.pl -type=asmiostat -show=inst,dbinst -sort_field=reads

With the following output:

<img src="{{ site.baseurl }}/assets/images/asmiostatv2_most_reads.png" class="aligncenter size-full wp-image-1189" width="620" height="269" alt="asmiostatv2_most_reads" />

As you can see the BDTO\_2 database instance is generating the most part of the read IO request using +ASM1.

<span style="text-decoration:underline;">**Second one**</span>: I want to see Flex ASM 12c (12.1) "unpreferred" read in action.

Well, I am using the same setup as the one described into the "[unpreferred read](http://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/ "Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !")" post.

I launch Kevin Closson's [SLOB2 Kit](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/) to generate Physical IO locally on the NOPBDT3 database instance. I check the behavior with my asmiostat utility that way:

    ./real_time.pl -type=asmiostat -show=inst,dbinst,dg,fg -dg=DATAP -dbinst='%NOP%'

With the following output:

<img src="{{ site.baseurl }}/assets/images/asmiostatv2_unpreff_reads.png" class="aligncenter size-full wp-image-1190" width="620" height="323" alt="asmiostatv2_unpreff_reads" />

As you can see the NOPBDT3 database instance (located in SITEB) is using the ASM1 instance which prefers to read from SITEA. Then the NOPBDT3 database instance is reading from SITEA which is bad.

<span style="text-decoration:underline;">Remarks and conclusion:</span>

-   My asmiostat utility V2 is helpful to see which database instance is using which ASM instance (and also collect the performance metrics).
-   This will be very useful with [Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) in place but it can also be used with non Flex ASM (See the first example).
-   You can download my asmiostat utility (which is part of the real\_time.pl script) from [this repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit).
-   The utility V2 still works with 10gr2 ASM but the "Database instance" feature is triggered as of 11gr1 (as it is based on the gv$asm\_disk\_iostat view).
-   I did not had the chance to play with pluggable databases yet: This will be the next step around my utility.
-   If you hit this issue:

<!-- -->

    ./real_time.pl 
    : No such file or directory

-   Then launch it that way:

<!-- -->

    perl ./real_time.pl

**UPDATE:** The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

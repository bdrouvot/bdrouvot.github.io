---
layout: post
title: 'ASM Preferred Read: Collect performance metrics thanks to my amsiostat utility
  and SLOB 2'
date: 2013-05-05 15:24:43.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:125;}s:2:"wp";a:1:{i:0;i:30;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  twitter_cards_summary_img_size: a:6:{i:0;i:1176;i:1;i:223;i:2;i:3;i:3;s:25:"width="1176"
    height="223"";s:4:"bits";i:8;s:4:"mime";s:9:"image/png";}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/05/05/asm-preferred-read-collect-performance-metrics-thanks-to-my-amsiostat-utility-and-slob-2/"
---

Some times ago I provided a way to see the ASM Preferred Read feature in action thanks to Kevin Closson's SLOB kit and my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") into this [blog post](http://bdrouvot.wordpress.com/2013/02/18/asm-preferred-read-collect-performance-metrics/ "ASM Preferred Read: Collect performance metrics"). Since that time Kevin released [SLOB 2.](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/)

One of its new feature is that it now supports RAC so that **we don't need to launch SLOB on each node anymore** as it has been done into my previous post.

Let's see how we can show the ASM preferred read feature in action and collect performance metrics thanks to SLOB 2.

<span style="text-decoration:underline;">First, let's create two services SLOB\_BDT1 and SLOB\_BDT2 respectively on BDT\_1 and BDT\_2 instances:</span>

    $ srvctl add service -s SLOB_BDT2 -d BDTO -r BDTO_2
    $ srvctl start service -s SLOB_BDT2 -d BDTO
    $ srvctl add service -s SLOB_BDT1 -d BDTO -r BDTO_1
    $ srvctl start service -s SLOB_BDT1 -d BDTO
    $ srvctl status service -s SLOB_BDT1 -d BDTO
    Service SLOB_BDT1 is running on instance(s) BDTO_1
    $ srvctl status service -s SLOB_BDT2  -d BDTO
    Service SLOB_BDT2 is running on instance(s) BDTO_2

<span style="text-decoration:underline;">Now let's configure the slob.conf so that the SLOB's sessions will be distributed over those 2 services with a "round-robin" manner and no updates triggered:</span>

    $ grep -i sqlnet slob.conf
    ADMIN_SQLNET_SERVICE=slob_bdt
    SQLNET_SERVICE_BASE=slob_bdt
    SQLNET_SERVICE_MAX=2
    $ grep -i UPDATE_PCT slob.conf | head -1
    UPDATE_PCT=0

<span style="text-decoration:underline;">Let's configure the ASM preferred read parameters:</span>

    SQL>  alter system set asm_preferred_read_failure_groups='BDT_ONLY.WIN' sid='+ASM1';

    System altered.

    SQL> alter system set asm_preferred_read_failure_groups='BDT_ONLY.JMO' sid='+ASM2';

    System altered.

As we want to see the ASM preferred read in action, we need to configure our BDT\_1 and BDT\_2 instances in such a way that the SLOB run will generate physical IOs.

<span style="text-decoration:underline;">For this purpose, I use those settings:</span>

    alter system set "_db_block_prefetch_limit"=0 scope=spfile sid='*';
    alter system set "_db_block_prefetch_quota"=0  scope=spfile sid='*';
    alter system set "_db_file_noncontig_mblock_read_count"=0 scope=spfile sid='*';
    alter system set "cpu_count"=1 scope=spfile sid='*';
    alter system set "db_cache_size"=4m scope=spfile sid='*';
    alter system set "shared_pool_size"=500m scope=spfile sid='*';
    alter system set "sga_target"=0 scope=spfile sid='*';

You can find a very good description, on how to use SLOB for PIO testing into this [flashdba's blog post](http://flashdba.com/database/useful-scripts/using-slob-for-pio-testing/).

<span style="text-decoration:underline;">Now we are ready to launch the SLOB 2 run:</span>

    $ ./runit.sh 16

<span style="text-decoration:underline;">and see the preferred read in action as well as its associated performance metrics thanks to my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") that way:</span>

    $ ./real_time.pl -type=asmiostat -dg=BDT_ONLY -show=dg,inst,fg

<span style="text-decoration:underline;">with the following result:</span>

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/asm_preferred_read_slob21.png" class="aligncenter size-full wp-image-979" width="620" height="117" alt="asm_preferred_read_slob2" />](http://bdrouvot.files.wordpress.com/2013/05/asm_preferred_read_slob21.png)

Great, data have been read from their preferred read failure groups ;-) We can also see their respectives performance metrics.

<span style="text-decoration:underline;">**Conclusion:**</span>

Thanks to [Kevin's SLOB 2 kit](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/) and my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") we are now able to check the ASM preferred read performance metrics with a single SLOB run.

**UPDATE:** The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

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

First, let's create two services SLOB\_BDT1 and SLOB\_BDT2 respectively on BDT\_1 and BDT\_2 instances:

```
$ srvctl add service -s SLOB\_BDT2 -d BDTO -r BDTO\_2 $ srvctl start service -s SLOB\_BDT2 -d BDTO $ srvctl add service -s SLOB\_BDT1 -d BDTO -r BDTO\_1 $ srvctl start service -s SLOB\_BDT1 -d BDTO $ srvctl status service -s SLOB\_BDT1 -d BDTO Service SLOB\_BDT1 is running on instance(s) BDTO\_1 $ srvctl status service -s SLOB\_BDT2 -d BDTO Service SLOB\_BDT2 is running on instance(s) BDTO\_2
```

Now let's configure the slob.conf so that the SLOB's sessions will be distributed over those 2 services with a "round-robin" manner and no updates triggered:

```
$ grep -i sqlnet slob.conf ADMIN\_SQLNET\_SERVICE=slob\_bdt SQLNET\_SERVICE\_BASE=slob\_bdt SQLNET\_SERVICE\_MAX=2 $ grep -i UPDATE\_PCT slob.conf | head -1 UPDATE\_PCT=0
```

Let's configure the ASM preferred read parameters:

```
SQL\> alter system set asm\_preferred\_read\_failure\_groups='BDT\_ONLY.WIN' sid='+ASM1'; System altered. SQL\> alter system set asm\_preferred\_read\_failure\_groups='BDT\_ONLY.JMO' sid='+ASM2'; System altered.
```

As we want to see the ASM preferred read in action, we need to configure our BDT\_1 and BDT\_2 instances in such a way that the SLOB run will generate physical IOs.

For this purpose, I use those settings:

```
alter system set "\_db\_block\_prefetch\_limit"=0 scope=spfile sid='\*'; alter system set "\_db\_block\_prefetch\_quota"=0 scope=spfile sid='\*'; alter system set "\_db\_file\_noncontig\_mblock\_read\_count"=0 scope=spfile sid='\*'; alter system set "cpu\_count"=1 scope=spfile sid='\*'; alter system set "db\_cache\_size"=4m scope=spfile sid='\*'; alter system set "shared\_pool\_size"=500m scope=spfile sid='\*'; alter system set "sga\_target"=0 scope=spfile sid='\*';
```

You can find a very good description, on how to use SLOB for PIO testing into this [flashdba's blog post](http://flashdba.com/database/useful-scripts/using-slob-for-pio-testing/).

Now we are ready to launch the SLOB 2 run:

```
$ ./runit.sh 16
```

and see the preferred read in action as well as its associated performance metrics thanks to my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") that way:

```
$ ./real\_time.pl -type=asmiostat -dg=BDT\_ONLY -show=dg,inst,fg
```

with the following result:

[![asm_preferred_read_slob2]({{ site.baseurl }}/assets/images/asm_preferred_read_slob21.png)](http://bdrouvot.files.wordpress.com/2013/05/asm_preferred_read_slob21.png)

Great, data have been read from their preferred read failure groups ;-) We can also see their respectives performance metrics.

**Conclusion:**

Thanks to [Kevin's SLOB 2 kit](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/) and my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") we are now able to check the ASM preferred read performance metrics with a single SLOB run.

**UPDATE:** &nbsp;The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.


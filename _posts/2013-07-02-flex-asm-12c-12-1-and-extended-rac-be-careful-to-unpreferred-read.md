---
layout: post
title: 'Flex ASM 12c (12.1) and Extended Rac: be careful to "unpreferred" read !'
date: 2013-07-02 09:40:24.000000000 +02:00
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
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:192;}s:2:"wp";a:1:{i:0;i:33;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  twitter_cards_summary_img_size: a:6:{i:0;i:844;i:1;i:355;i:2;i:3;i:3;s:24:"width="844"
    height="355"";s:4:"bits";i:8;s:4:"mime";s:9:"image/png";}
  geo_public: '0'
  _wpas_skip_7950430: '1'
  _wpas_skip_5547632: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/gtJKcwKJcTJ
  _wpas_skip_8482624: '1'
  _wpas_done_8482624: '1'
  _publicize_job_id: '5242981235'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/"
---

**Update 2015/03/06:** The following has been recognized as "unpublished BUG 17045279 - ASM\_PREFERRED\_READ DOES NOT WORK WITH FLEX ASM", which is planned to be fixed in the next upcoming release.

**Update 2017/05/20:** As of 12.2, preferred reads are site-aware (extract of  [Markus Michalewicz](https://twitter.com/OracleRACpm) presentation available [here](https://www.slideshare.net/MarkusMichalewicz/oracle-extended-clusters-for-oracle-rac)) so that the issue described into this blog post has been addressed.

<img src="{{ site.baseurl }}/assets/images/pref_site_aware.png" class="aligncenter size-full wp-image-3127" width="640" height="359" />

 

**Introduction**
----------------

As you know Oracle 11g introduced a new feature called "Asm Preferred Read". It is very useful in extended RAC as it allows each node to define a preferred failure group, allowing nodes to access local failure groups in preference to remote ones. This is done thanks to the "*asm\_preferred\_read\_failure\_groups*" parameter.

<span style="text-decoration:underline;">**Fine, but remember:**</span>

1.  This parameter has to be set at the **ASM instance level** (not the database instance one).
2.  This is the **database instance** (or its shadow processes) that is doing the IOs (not the ASM instance).

<span style="text-decoration:underline;">**Why is it important ?** </span>  Because with [Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) in place, database instances are connection load balanced across the set of available ASM instances (that of course are not necessary "local" to the database instance anymore):

<img src="{{ site.baseurl }}/assets/images/flex_asm1.png" class="aligncenter size-full wp-image-1151" width="620" height="260" alt="flex_asm1" />

And then you could hit what I call the "**unpreferred**" read behavior.

<span style="text-decoration:underline;">Let me explain more in depth with an example:</span>

Suppose that you have an extended 3 nodes RAC:

-   racnode1 located in SITE A
-   racnode2 located in SITE A
-   racnode3 located in SITE B

And 2 ASM instances actives:

-   +ASM1 located in SITE A
-   +ASM3 located in SITE B

<!-- -->

    srvctl status asm
    ASM is running on racnode3,racnode1

So you created 2 failgroup SITEA and SITEB and you set the *asm\_preferred\_read\_failure\_groups* parameter that way for the DATAP diskgroup:

    SQL> alter system set asm_preferred_read_failure_groups='DATAP.SITEB' sid='+ASM3';
    System altered.

    SQL> alter system set asm_preferred_read_failure_groups='DATAP.SITEA' sid='+ASM1';
    System altered.

So that ASM3 prefers to read from SITEB and ASM1 from SITEA (which fully makes sense from the ASM point of view).

**<span style="text-decoration:underline;">But what if  a database instance located into SITEB (racnode3) is using ASM1 located in SITEA ?</span>**

    SQL>  select I.INSTANCE_NAME,C.INSTANCE_NAME,C.DB_NAME
      2  from gv$instance I, gv$asm_client C 
      3  where C.INST_ID=I.INST_ID and C.instance_name='NOPBDT3';

    INSTANCE_NAME    INSTANCE_NAME                                                    DB_NAME
    ---------------- ---------------------------------------------------------------- --------
    +ASM1            NOPBDT3                                                          NOPBDT

As you can see the NOPBDT3 database instance is using the +ASM1 instance, while the NOPBDT3 database instance is located on racnode3:

    srvctl status instance -i NOPBDT3 -d NOPBDT
    Instance NOPBDT3 is running on node racnode3

**Which means NOPBDT3 located into SITEB will prefer to request read IO from SITEA which is of course very bad.**

<span style="text-decoration:underline;">Let's check this with my [asmiostat](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") utility and Kevin Closson's [SLOB2 kit](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/):</span>

Let's launch SLOB locally on NOPBDT3 only:

    [oracle@racnode3 SLOB]$ ./runit.sh 3
    NOTIFY: 
    UPDATE_PCT == 0
    RUN_TIME == 300
    WORK_LOOP == 0
    SCALE == 10000
    WORK_UNIT == 256
    ADMIN_SQLNET_SERVICE == ""
    ADMIN_CONNECT_STRING == "/ as sysdba"
    NON_ADMIN_CONNECT_STRING == ""
    SQLNET_SERVICE_MAX == "0"

And check the IO metrics with my asmiostat utility that way (I want to see Instance, Diskgroup and Failgroup):

    ./real_time.pl -type=asmiostat -show=inst,dg,fg -dg=DATAP

With the following output:

<img src="{{ site.baseurl }}/assets/images/flex_asm_pref_read_12c.png" class="aligncenter size-full wp-image-1161" width="620" height="236" alt="flex_asm_pref_read_12c" />

As I am the only one to work on this [Lab](http://bdrouvot.wordpress.com/2013/06/29/build-your-own-flex-asm-12c-lab-using-virtual-box/ "Build your own Flex ASM 12c lab using Virtual Box"), you can see with no doubt that the IO metrics coming from the Instance NOPBDT3 are recorded into the ASM instance +ASM1 and clearly indicates that the read IOs have been done on SITEA.

<span style="text-decoration:underline;">**How can we "fix" this ?**</span>

You can temporary fix this that way (connected on the +ASM1 instance):

    SQL> ALTER SYSTEM RELOCATE CLIENT 'NOPBDT3:NOPBDT';
    System altered.

    SQL> select I.INSTANCE_NAME,C.INSTANCE_NAME,C.DB_NAME
      2  from gv$instance I, gv$asm_client C 
      3   where C.INST_ID=I.INST_ID and C.instance_name='NOPBDT3';

    INSTANCE_NAME    INSTANCE_NAME                                                    DB_NAME
    ---------------- ---------------------------------------------------------------- --------
    +ASM3            NOPBDT3                                                          NOPBDT
    +ASM3            NOPBDT3                                                          NOPBDT

That way the NPPBDT3 database instance will use the +ASM3 instance and then will launch its read IO on SITEB .

But I had to bounce the NOPBDT3 database instance so that it launchs the read IO from SITEB (If not it was still using SITEA, well maybe a subject for another post)

<span style="text-decoration:underline;">**Conclusion:**</span>

[Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) is a great feature but you have to be careful if you want to use it in an extended Rac with preferred read in place.  If not you may hit the "unpreferred" read behavior.

**UPDATES:**

-   Have a look to this [new post](http://bdrouvot.wordpress.com/2013/07/05/asm-io-statistics-utility-v2/ "ASM I/O Statistics Utility V2") describing my ASM I/O Statistics V2 to see the "unpreferred" read more in depth (at the database instance level).
-   The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.
-   Unpreferred read still exists in 12.1.0.2: ASM1/NBDT1 located on SITE1, ASM2/NBDT2 located on SITE2, ASM1 prefers to read from SITE1 and serves NBDT2:

<img src="{{ site.baseurl }}/assets/images/unpref_12102.png" class="aligncenter size-full wp-image-2239" width="640" height="129" alt="unpref_12102" />

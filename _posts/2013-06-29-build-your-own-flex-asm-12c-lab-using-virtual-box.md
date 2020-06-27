---
layout: post
title: Build your own Flex ASM 12c lab using Virtual Box
date: 2013-06-29 18:23:29.000000000 +02:00
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
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:185;}s:2:"wp";a:1:{i:0;i:33;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  geo_public: '0'
  twitter_cards_summary_img_size: a:6:{i:0;i:765;i:1;i:576;i:2;i:3;i:3;s:24:"width="765"
    height="576"";s:4:"bits";i:8;s:4:"mime";s:9:"image/png";}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/06/29/build-your-own-flex-asm-12c-lab-using-virtual-box/"
---

So, Oracle just released database 12c, and one of its new feature is Flex ASM. You can find more details about this new feature into this Bjoern Rost's [post](http://portrix-systems.de/blog/brost/flex-everything-in-oracle-rac-12c/) or this [oracle white paper](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf).[  
](http://portrix-systems.de/blog/author/brost/ "View all posts by Bjoern Rost")

Basically, as Bjoern said: "Flex ASM basically allows a database instance to utilize an ASM instance that is not local to that server node and even fail over to another remote instance if the ASM instance fails".

This short blog post is just to show how you can build your own Flex ASM Lab.

First, I built a 3 nodes RAC. For this purpose you just have to follow one of those great blog posts:

-    Yury Velikanov's one : [Oracle 12c RAC On your laptop, Step by Step Guide](http://www.pythian.com/blog/oracle-12c-rac-on-your-laptop-step-by-step-guide/)
-    Tim Hall's one: [Oracle Database 12c Release 1 (12.1.0.1) RAC On Oracle Linux 6 Using VirtualBox](http://www.oracle-base.com/articles/12c/oracle-db-12cr1-rac-installation-on-oracle-linux-6-using-virtualbox.php)

<span style="text-decoration:underline;">with the following changes:</span>

-   You need to adapt those posts to build a 3 nodes RAC.
-   During the install process of the Grid Infrastructure:

Step 3: Choose "Configure a Standard cluster"

<img src="{{ site.baseurl }}/assets/images/12c_grid_infra_step31.png" class="aligncenter size-full wp-image-1143" width="620" height="466" alt="12c_grid_infra_step3" />

Step 4: Choose "Advanced Installation"

<img src="{{ site.baseurl }}/assets/images/12c_grid_infra_step4.png" class="aligncenter size-full wp-image-1140" width="620" height="463" alt="12c_grid_infra_step4" />

Step 10: Choose "Use Oracle Flex ASM for Storage"

<img src="{{ site.baseurl }}/assets/images/12c_grid_infra_step10.png" class="aligncenter size-full wp-image-1141" width="620" height="463" alt="12c_grid_infra_step10" />

Once this is installed, you can verify that Flex ASM has been set up:

    asmcmd showclustermode
    ASM cluster : Flex mode enabled

You can also check how many ASM instances have been specified for use:

    srvctl config asm | grep -i count
    ASM instance count: 3

Then you can change the configuration to have 2 ASM instances active on this 3 nodes Lab:

    srvctl modify asm -count 2

Once done you can check the status:

    srvctl status asm -detail
    ASM is running on racnode2,racnode1
    ASM is enabled.

<span style="text-decoration:underline;">Remarks:</span>

-   I did not test with a 2 nodes RAC and one active ASM instance: If you do please let me know.
-   If you are curious (as I am) to see how the Flex ASM behaves during physical IO operations, then I suggest to use Kevin Closson's [SLOB2 Kit](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/) and my [asmiostat utility](http://bdrouvot.wordpress.com/2013/02/15/asm-io-statistics-utility/ "ASM I/O Statistics Utility") (I'll use them for sure and share my findings/remarks if any ;-) )
-   Steve Karam is building a huge list of  Oracle 12c articles from the community into this [post ](http://www.oraclealchemist.com/news/install-oracle-12c-12-1/) (Do not hesitate to have a look at it)

<span style="text-decoration:underline;">Findings reported thanks to this Lab:</span>

-   Finding 1: [Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !](http://bdrouvot.wordpress.com/2013/07/02/flex-asm-12c-12-1-and-extended-rac-be-careful-to-unpreferred-read/ "Flex ASM 12c (12.1) and Extended Rac: be careful to “unpreferred” read !")
-   Finding 2: [Flex ASM 12c (12.1): be careful to “invisible” I/O !](http://bdrouvot.wordpress.com/2013/07/16/flex-asm-12c-12-1-be-careful-to-invisible-io/ "Flex ASM 12c (12.1): be careful to “invisible” I/O !")

 

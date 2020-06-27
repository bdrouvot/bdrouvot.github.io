---
layout: post
title: Issue installing the 11gR2 database software after the 12.1.0.2 clusterware
  setup
date: 2014-09-17 19:39:00.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rac
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/Eeqj3SRV7SG
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/Ekh6Bofifw
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5918075472683503616&type=U&a=QtOJ
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
permalink: "/2014/09/17/issue-installing-11gr2-database-software-after-the-12-1-0-2-clusterware-setup/"
---

If you don't plan to install the 11gR2 database software after a 12.1.0.2 clusterware installation, I guess there is no need for you to read this post.

I just want to share the issue I got and the way you could workaround it. The purpose of this post is just to save your time, in case of.

So, after a 12.1.0.2 clusterware installation (on a 2 nodes RAC cluster), I decided to install the 11.2.0.4 database software. I launched the runInstaller, followed the install process until the Step 4:

[<img src="{{ site.baseurl }}/assets/images/install11g_db.png" class="aligncenter size-full wp-image-2324" width="640" height="496" alt="install11g_db" />](http://bdrouvot.files.wordpress.com/2014/09/install11g_db.png)

As you can see the nodes that are part of the cluster **have not been proposed and clicking next would produce "CRS is not installed on any of the nodes". **

<span style="text-decoration:underline;">The first question you may ask is</span>: Is the 11gR2 database compatible with CRS 12.1? Yes it is, as you can see [here](http://docs.oracle.com/database/121/CWADD/intro.htm#CWADD90955):

[<img src="{{ site.baseurl }}/assets/images/crs_supported_version1.png" class="aligncenter size-full wp-image-2328" width="640" height="414" alt="crs_supported_version" />](http://bdrouvot.files.wordpress.com/2014/09/crs_supported_version1.png)

As I am curious, I canceled the 11gR2 database software installation and gave a try with the 12.1.0.2 database software. The Step 4 produced:

[<img src="{{ site.baseurl }}/assets/images/install12c_db.png" class="aligncenter size-full wp-image-2325" width="640" height="494" alt="install12c_db" />](http://bdrouvot.files.wordpress.com/2014/09/install12c_db.png)

As you can see the nodes that are part of the cluster **have been proposed**.

I canceled the 12.1.0.2 database software installation and I had a look to the Inventory. Then I discovered that **the CRS="true"** flag was not set for the 12.1.0.2 GI HOME:

    $ cat /etc/oraInst.loc | grep inventory_loc
    inventory_loc=/ec/poc/server/oracle/olrpoc1/u000/oraInventory

    $ cat /ec/poc/server/oracle/olrpoc1/u000/oraInventory/ContentsXML/inventory.xml
    ..
    HOME NAME="OraGI12Home1" LOC="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" TYPE="O" IDX="1"
    ..

Then I added the **the CRS="true" flag** that way:

    $GI_HOME/oui/bin/runInstaller -updateNodeList ORACLE_HOME="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" CRS=true

So that:

    $ cat /ec/poc/server/oracle/olrpoc1/u000/oraInventory/ContentsXML/inventory.xml
    ..
    HOME NAME="OraGI12Home1" LOC="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" TYPE="O" IDX="1" CRS="true"
    ..

Then I relaunched the 11.2.0.4 database software installation, and the Step 4 produced:

[<img src="{{ site.baseurl }}/assets/images/install11g_db_fixed.png" class="aligncenter size-full wp-image-2326" width="640" height="496" alt="install11g_db_fixed" />](http://bdrouvot.files.wordpress.com/2014/09/install11g_db_fixed.png)

As you can see the **nodes have been proposed so that** I have been able to complete the installation successfully.

<span style="text-decoration:underline;">**Remarks:**</span>

1.  MOS 1053393.1 provides more details about the CRS="true" flag.
2.  I guess the "issue" will be the same for database software &gt;=10.1 and &lt; 12.1 (But I did not test it).
3.  On another RAC, I **upgraded** the GI from 12.1.0.1 to 12.1.0.2 and then the CRS="true" flag **has been set automatically** for the 12.1.0.2 GI.

<span style="text-decoration:underline;">**Conclusion:**</span> **After a 12.1.0.2 CRS installation,**

1.  You may **need** to put the **CRS="true"** flag to avoid this issue during a 11gR2 database software installation.
2.  You **don't need** to put the **CRS="true"** flag during a 12.1.0.2 database software installation as the issue does not appear.
3.  It looks like you won't hit the issue if the GI has been upgraded to 12.1.0.2 (See third remark).

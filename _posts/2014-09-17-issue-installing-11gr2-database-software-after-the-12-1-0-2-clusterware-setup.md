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
If you don't plan to install the 11gR2 database software after a 12.1.0.2 clusterware&nbsp;installation, I guess there is no need for you to read this post.

I just want to share the issue I got and the way you could workaround it. The purpose of this post is just to save your time, in case of.

So, after a 12.1.0.2 clusterware&nbsp;installation (on a 2 nodes RAC cluster), I decided to install the 11.2.0.4 database software. I launched the runInstaller, followed the install process until the Step 4:

[![install11g_db]({{ site.baseurl }}/assets/images/install11g_db.png)](http://bdrouvot.files.wordpress.com/2014/09/install11g_db.png)

As you can see the nodes that are part of the cluster&nbsp; **have not been proposed and&nbsp;clicking next would produce "CRS is not installed on any of the nodes".&nbsp;**

The first question you may ask is: Is the 11gR2 database compatible with CRS 12.1? Yes it is, as you can see [here](http://docs.oracle.com/database/121/CWADD/intro.htm#CWADD90955):

[![crs_supported_version]({{ site.baseurl }}/assets/images/crs_supported_version1.png)](http://bdrouvot.files.wordpress.com/2014/09/crs_supported_version1.png)

As I am curious, I canceled the 11gR2 database software installation and gave a try with the 12.1.0.2 database software. The Step 4 produced:

[![install12c_db]({{ site.baseurl }}/assets/images/install12c_db.png)](http://bdrouvot.files.wordpress.com/2014/09/install12c_db.png)

As you can see the nodes that are part of the cluster&nbsp; **have been proposed**.

I canceled the 12.1.0.2 database software installation and I had a look to the Inventory. Then I&nbsp;discovered that **the CRS="true"** flag was not set for the 12.1.0.2 GI HOME:

```
$ cat /etc/oraInst.loc | grep inventory\_loc inventory\_loc=/ec/poc/server/oracle/olrpoc1/u000/oraInventory $ cat /ec/poc/server/oracle/olrpoc1/u000/oraInventory/ContentsXML/inventory.xml .. HOME NAME="OraGI12Home1" LOC="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" TYPE="O" IDX="1" ..
```

Then I added the&nbsp; **the CRS="true" flag** that way:

```
$GI\_HOME/oui/bin/runInstaller -updateNodeList ORACLE\_HOME="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" CRS=true
```

So that:

```
$ cat /ec/poc/server/oracle/olrpoc1/u000/oraInventory/ContentsXML/inventory.xml .. HOME NAME="OraGI12Home1" LOC="/ec/poc/server/oracle/olrpoc1/u000/product/GRID.12.1.0.2" TYPE="O" IDX="1" CRS="true" ..
```

Then I relaunched the 11.2.0.4 database software installation, and the Step 4 produced:

[![install11g_db_fixed]({{ site.baseurl }}/assets/images/install11g_db_fixed.png)](http://bdrouvot.files.wordpress.com/2014/09/install11g_db_fixed.png)

As you can see the **nodes have been proposed so that** &nbsp;I have been able to complete the installation&nbsp;successfully.

**Remarks:**

1. MOS&nbsp;1053393.1 provides more details about the CRS="true" flag.
2. I guess the "issue" will be the same for database software \>=10.1 and \< 12.1 (But I did not&nbsp;test it).
3. On another RAC, I **upgraded** the GI from 12.1.0.1 to 12.1.0.2 and then the CRS="true" flag **has been set automatically** for the 12.1.0.2 GI.

**Conclusion:**  **After a 12.1.0.2 CRS installation,**

1. You may&nbsp; **need** to put the **CRS="true"** flag to avoid this issue during&nbsp;a&nbsp;11gR2 database software installation.
2. You **don't need** to put the **CRS="true"** flag during&nbsp;a&nbsp;12.1.0.2&nbsp;database software installation as the issue does not appear.
3. It looks like you won't hit the issue if the GI has been upgraded to 12.1.0.2 (See third remark).

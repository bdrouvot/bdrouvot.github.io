---
layout: post
title: Modify the Private Network Information in Oracle Clusterware with HAIP in place
date: 2014-01-27 12:23:36.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rac
tags: [Oracle RAC, oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/9VULZ95YYuU
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5833529520376135680&type=U&a=D-Km
  publicize_facebook_url: https://facebook.com/
  _wpas_done_5547632: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"101126738655139704850";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/aHPVpaUmhq
  _wpas_done_2225791: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_5536151: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/01/27/modify-the-private-network-information-in-oracle-clusterware-with-haip-in-place/"
---

The MOS note "How to Modify Private Network Information in Oracle Clusterware (Doc ID 283684.1)" explains how to change the private Network information with the interconnect made of a single interface (then producing **downtime**).

So what's about changing the interconnect configuration **with Highly Available IP (HAIP) in place** ?

Let's remember what HAIP is (from [Oracle® Database High Availability Best Practices](http://docs.oracle.com/cd/E11882_01/server.112/e10803/config_cw.htm#HABPT4849)):

<img src="{{ site.baseurl }}/assets/images/haip.png" class="aligncenter size-full wp-image-1592" width="620" height="157" alt="haip" />

As HAIP provides redundant interconnect, we should be able to change the interconnect configuration of **one private interface**  without any downtime, right ?

<span style="text-decoration:underline;">First let's check the current interconnect configuration:</span>

    oifcfg getif       
    eth4  172.16.0.128  global  cluster_interconnect
    eth6  172.16.1.0  global  cluster_interconnect

<span style="text-decoration:underline;">and the associated Virtual IP:</span>

    oifcfg iflist -p -n
    eth4  172.16.0.128  PRIVATE  255.255.255.128
    eth4  169.254.0.0  UNKNOWN  255.255.128.0
    eth6  172.16.1.0  PRIVATE  255.255.255.128
    eth6  169.254.128.0  UNKNOWN  255.255.128.0

I removed the public network from the output. As you can see each private interface is hosting a Virtual IP (169.xxx.x.x).

Now your sysadmin do the change (for example he will change the subnet and the VLAN) on one of the private interface (Let's say eth4 for example), so that the **ohasd.log** log file reports something like:

    2014-01-27 11:16:17.154: [GIPCHGEN][1837012736]gipchaInterfaceFail: marking interface failing 0x7f4c0c1b5f00 { host '', haName 'CLSFRAME_olrdev1', local (nil), ip '172.16.0.129:28029', subnet '172.16.0.128', mask '255.255.255.128', mac 'e8-39-35-12-77-7e', ifname 'eth4', numRef 0, numFail 0, idxBoot 0, flags 0x184d }
    2014-01-27 11:16:17.334: [GIPCHGEN][1856595712]gipchaInterfaceDisable: disabling interface 0x7f4c0c1b5f00 { host '', haName 'CLSFRAME_olrdev1', local (nil), ip '172.16.0.129:28029', subnet '172.16.0.128', mask '255.255.255.128', mac 'e8-39-35-12-77-7e', ifname 'eth4', numRef 0, numFail 0, idxBoot 0, flags 0x19cd }
    2014-01-27 11:16:17.339: [GIPCHDEM][1856595712]gipchaWorkerCleanInterface: performing cleanup of disabled interface 0x7f4c0c1b5f00 { host '', haName 'CLSFRAME_olrdev1', local (nil), ip '172.16.0.129:28029', subnet '172.16.0.128', mask '255.255.255.128', mac 'e8-39-35-12-77-7e', ifname 'eth4', numRef 0, numFail 0, idxBoot 0, flags 0x19ed }

<span style="text-decoration:underline;">So now let's check the virtual IP and the available interfaces and subnet again:</span>

    oifcfg iflist -p -n
    eth4  172.17.3.0  PRIVATE  255.255.255.128
    eth6  172.16.1.0  PRIVATE  255.255.255.128
    eth6  169.254.128.0  UNKNOWN  255.255.128.0
    eth6  169.254.0.0  UNKNOWN  255.255.128.0
    bond0  158.168.4.0  UNKNOWN  255.255.252.0

<span style="text-decoration:underline;">Well, we can see 2 things:</span>

-   The first one, is that eth6 is now "hosting" **both** Virtual IPs (169.xxxx).
-   The second one, is the **new** "available" subnet for eth4 (172.17.3.0).

<span style="text-decoration:underline;">So that, we just have to remove the previous eth4 configuration</span>

    oifcfg delif -global eth4/172.16.0.128

<span style="text-decoration:underline;">and put the new one that way:</span>

    oifcfg setif -global eth4/172.17.3.0:cluster_interconnect

<span style="text-decoration:underline;">Now, check again the Virtual IPs:</span>

    oifcfg iflist -p -n
    eth4  172.17.3.0  PRIVATE  255.255.255.128
    eth4  169.254.128.0  UNKNOWN  255.255.128.0
    eth6  172.16.1.0  PRIVATE  255.255.255.128
    eth6  169.254.0.0  UNKNOWN  255.255.128.0

Perfect, now each private Interface hosts a Virtual IP (We are back to a "normal" configuration).

<span style="text-decoration:underline;">Remarks:</span>

No downtime will occur as long as:

-   You don't change both interfaces configuration at the same time ;-)
-   You don't remove by mistake the configuration of the interface that hosts both Virtual IPs.
-   The interconnect hosting both VIPs doesn't fail before you put back the updated interface.

There is nothing new with this post. I just had to do the exercise so that I share it ;-)

<span style="text-decoration:underline;">Conclusion:</span>

Thanks to HAIP, we have been able to change the Interconnect Network configuration of one interface without any downtime.

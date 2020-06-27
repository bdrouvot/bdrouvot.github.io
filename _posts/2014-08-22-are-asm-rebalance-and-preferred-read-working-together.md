---
layout: post
title: Are ASM rebalance and preferred read working together?
date: 2014-08-22 10:27:03.000000000 +02:00
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
  publicize_facebook_url: https://facebook.com/1471304506456400
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/Ead84FcBii9
  _publicize_pending: '1'
  _edit_last: '40807211'
  geo_public: '0'
  _wpas_done_7950430: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/3ZAT5ivv3P
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5908514502608977920&type=U&a=mRPW
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_5547632: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2014/08/22/are-asm-rebalance-and-preferred-read-working-together/"
---
<p><span style="text-decoration:underline;"><strong>Introduction:</strong></span></p>
<p>If I add disks into a diskgroup, then during the rebalance operation, ASM needs to read the data coming from the disks already part of the diskgroup to rebalance them on all the disks (including the new ones).</p>
<p><span style="text-decoration:underline;"><strong>Question:</strong></span></p>
<p>If I add 2 disks (one into each failgroup) is the preferred feature took into account <strong>for</strong> the rebalance process? ("for" means "for the reads generated <strong>by</strong> the rebalance operation").</p>
<p><span style="text-decoration:underline;"><strong>Let's see:</strong></span></p>
<ul>
<li>Set the preferred read on +ASM1 (so that +ASM1 "prefers" to read from the "HOST31" failgroup):</li>
</ul>
<pre style="padding-left:30px;">SQL&gt; alter system set asm_preferred_read_failure_groups='DATA.HOST31';

System altered.</pre>
<ul>
<li>Add 2 disks (one into each failgroup) into the DATA diskgroup (connected on +ASM1):</li>
</ul>
<pre style="padding-left:30px;">SQL&gt; alter diskgroup DATA add failgroup HOST31 disk '/dev/san/HOST31CA8D0D' failgroup HOST32 disk '/dev/san/HOST32CA8D0D';

Diskgroup altered.</pre>
<ul>
<li>Check that the ASM compatibility is high enough (&gt;=11.1) to use the preferred read feature:</li>
</ul>
<pre style="padding-left:30px;">SQL&gt; select COMPATIBILITY from v$asm_diskgroup where NAME='DATA';

COMPATIBILITY
------------------------------------------------------------
11.2.0.2.0

- Launch the rebalance:

```
SQL\> alter diskgroup DATA rebalance power 2; Diskgroup altered.
```

Now, let's collect the [ASM metrics that way and visualize the result with Tableau](http://bdrouvot.wordpress.com/2014/07/08/graphing-asm-performance-metrics/ "Graphing ASM performance metrics").

Note: During the rebalance near zero database activity occurred so that near 100%&nbsp;of the activity is coming from the rebalance process.

**Result:**

[![Screen Shot 2014-08-23 at 18.08.39]({{ site.baseurl }}/assets/images/screen-shot-2014-08-23-at-18-08-39.png)](https://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-23-at-18-08-39.png)

As you can see:

1. The +ASM1 instance reads from the HOST31 and the HOST32 failgroups: It did **not take into account** the preferred read.
2. I changed the power during the rebalance just for the fun ;-)

**Remark:**

It has been tested on a 11.2.0.4 extended RAC (Still need to test on 12c).

**Conclusion:**

- The ASM preferred read feature **is not took** into account **for** the rebalance process.
- I guess it is still took into account for the **reads coming from the databases during** the rebalance process.

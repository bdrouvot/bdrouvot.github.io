---
layout: post
title: db_io*metrics family
date: 2014-05-09 12:33:55.000000000 +02:00
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
  publicize_facebook_url: https://facebook.com/1431232367130281
  _wpas_done_5536151: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/3FqcqaqeLWr
  _wpas_done_5547632: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/dNj1Z5eRes
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5870495690811011072&type=U&a=cEZf
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
permalink: "/2014/05/09/db_iometrics-family/"
---
Into the last three posts I introduced three utilities to report database physical IO metrics in real time.

[![Screen Shot 2014-05-09 at 08.40.54]({{ site.baseurl }}/assets/images/screen-shot-2014-05-09-at-08-40-54.png?w=300)](http://bdrouvot.files.wordpress.com/2014/05/screen-shot-2014-05-09-at-08-40-54.png)I just want to introduce again those three utilities into a single post as:

- You may have missed one of them.
- They are all members of the same family.

The three utilities are:

1. [db\_io\_metrics](http://bdrouvot.wordpress.com/db_io_metrics_script/ "db\_io\_metrics")&nbsp;for reads and writes metrics related to data files and temp files.
2. [db\_io\_type\_metrics](http://bdrouvot.wordpress.com/db_io_type_metrics_script/ "db\_io\_type\_metrics")&nbsp;for reads, writes, small, large and synchronous metrics related to data files, temp files,control files, log files, archive logs, and so on.
3. [db\_io\_function\_metrics](http://bdrouvot.wordpress.com/db_io_function_metrics_script/ "db\_io\_function\_metrics")&nbsp;for reads, writes, small and large metrics related to database functions (LGWR, DBWR, Smart Scan and so on).

If a new member join this family, I will let you know.

**Remarks:**

- One member of the family looks like&nbsp;[SLOB](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/)'s logo (see below its "official" logo) designed by [flashdba](http://flashdba.com/)'s childs, don't you think? ;-)

[![Screen Shot 2014-05-09 at 11.54.47]({{ site.baseurl }}/assets/images/screen-shot-2014-05-09-at-11-54-47.png?w=213)](http://bdrouvot.files.wordpress.com/2014/05/screen-shot-2014-05-09-at-11-54-47.png)

- &nbsp;If you are interested in IO metrics related to ASM, you can have a look to the [asm\_metrics](http://bdrouvot.wordpress.com/asm_metrics_script/ "asm\_metrics") utility.

---
layout: post
title: 'ASM Rebalance: Why is the avg By/Read equal to 1MB while the allocation unit
  is 4MB?'
date: 2014-09-03 21:50:06.000000000 +02:00
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
  geo_public: '0'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/1481539125432938
  _wpas_done_7950430: '1'
  _publicize_done_external: a:1:{s:8:"facebook";a:1:{i:100007305933776;b:1;}}
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/hh7WZqBizqm
  _wpas_done_8482624: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/d6azzQVMVk
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5913035041474121728&type=U&a=pelK
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
permalink: "/2014/09/03/asm-rebalance-why-is-the-avg-byread-equal-to-1mb-while-the-allocation-unit-is-4mb/"
---

During the "[A closer look at ASM rebalance series](http://bdrouvot.wordpress.com/2014/08/25/a-closer-look-at-asm-rebalance-part-i-disks-have-been-added/ "A closer look at ASM rebalance, Part I: Disks have been added")" I observed that the average *By/Read* extracted from the *v$asm\_disk\_stat* view during a rebalance is equal to 1MB even if the ASM allocation unit is set to 4MB.

<span style="text-decoration:underline;">Example:</span>

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/screen-shot-2014-08-25-at-18-38-07.png" class="aligncenter size-full wp-image-2218" width="640" height="309" alt="Screen Shot 2014-08-25 at 18.38.07" />](http://bdrouvot.files.wordpress.com/2014/08/screen-shot-2014-08-25-at-18-38-07.png)

<span style="text-decoration:underline;">**The question is: why?**</span>

To try to answer this question I decided to use strace (the Linux system call trace utility) on the ASM arb process (this process rebalances data extents within an ASM disk group) during the rebalance of the DATA diskgroup (AU size is **4MB**).

<span style="text-decoration:underline;">**From strace I can see:**</span>

-   That a very large majority (more than 80% in my case) of the IOs are submitted that way:

<!-- -->

    io_submit(140691479359488, 4, \{\{0x7ff547f6c210, 0, 0, 0, 261\}, \{0x7ff547f6d0b0, 0, 0, 0, 261\}, \{0x7ff547f6c960, 0, 0, 0, 261\}, \{0x7ff547f6cbd0, 0, 0, 0, 261\}\}) = 4

We can see that **4** IOs have been submitted **at the same time** (with a single io\_submit call).

-   And that about 100% of all the IOs submitted are 1MB (look at the IO size which is 1048576 bytes):

<!-- -->

    io_getevents(140691479359488, 1, 128, \{\{0x7ff547f6c960, 0x7ff547f6c960, 1048576, 0\}, \{0x7ff547f6cbd0, 0x7ff547f6cbd0, 1048576, 0\}, \{0x7ff547f6d0b0, 0x7ff547f6d0b0, 1048576, 0\}, \{0x7ff547f6c210, 0x7ff547f6c210, 1048576, 0\}\}, \{600, 0\}) = 4

I also straced arb during the rebalance of the DATA1M diskgroup (Allocation unit of **1MB**) and observed:

-   That about 80% of the IOs are submitted that way:

<!-- -->

    io_submit(139928633700352, 1, \{\{0x7f43aadbd210, 0, 1, 0, 262\}\}) = 1

So **1** IO is submitted per io\_submit call.

-   And that about 100% of all the IOs submitted are 1MB:

<!-- -->

    io_getevents(139928633700352, 3, 128, \{\{0x7f43aadbd6f0, 0x7f43aadbd6f0, 1048576, 0\}\}, \{0, 0\}) = 1

<span style="text-decoration:underline;">**So that it makes sense to conclude that:**</span>

1.  The arb process **always request IOs of 1MB** (whatever the allocation unit is).
2.  The arb IOs requests are grouped and submitted at the same time depending of the diskgroup allocation unit (**4** IOs are submitted at the same time with allocation unit set to **4MB**, **1** IO submitted by io\_submit call with allocation unit set to **1MB**).

Based on this, I think it makes sense that the *v$asm\_disk\_stat* view reports *Avg By/Read* of 1MB during the rebalance process (whatever the allocation unit is).

<span style="text-decoration:underline;">**But now one more question:**</span> What is the impact of the [Linux Maximum IO size](http://martincarstenbach.wordpress.com/2013/07/03/increasing-the-maximum-io-size-in-linux/)?

For this, I launched the rebalance 2 times (on both diskgroups: DATA with AU=4MB and DATA1M with AU=1MB):  One time with max\_sectors\_kb set to 512 and one time with max\_sectors\_kb set to 4096.

<span style="text-decoration:underline;">**And observed that:**</span>

-   Oracle still does 1MB IOs only and groups the IOs depending of the allocation unit (**whatever** the max\_sectors\_kb is).

<!-- -->

-   With max\_sectors\_kb set to **512**: The IOs issued on the devices linked to the ASM disks are **limited** to 512 KB (even if oracle requested 1MB IOs). This can be seen with the iostat output (looking at the avgrq-sz field):

<!-- -->

    Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sdak              0.00     0.00    0.00   13.65     0.00 13919.15  1019.72     0.04    3.11   3.11   4.25
    sdbd              0.00     0.00    0.10    0.00   102.05     0.00  1020.50     0.00    2.00   2.00   0.02
    sddh              0.00     0.00    0.15    0.00   153.25     0.00  1021.67     0.00    4.33   4.33   0.07
    sdej              0.00     0.00    0.00   12.35     0.00 12640.10  1023.49     0.04    3.01   3.01   3.72
    sdcr              0.00     0.00    0.00   11.80     0.00 12083.20  1024.00     0.03    2.71   2.71   3.20
    sdfi              0.00     0.00    0.00   12.35     0.00 12646.40  1024.00     0.07    5.55   5.55   6.85
    sdg               0.00     0.00    0.00   11.70     0.00 11980.80  1024.00     0.06    5.35   5.34   6.25

The avgrq-sz is the average size (in sectors) of the requests that were issued to the device. The sector size I am using is 512 bytes ([not 4K](http://flashdba.com/4k-sector-size/deep-dive-oracle-with-4k-sectors/)) so that the IOs are about 1024\*512 bytes = 512 KB (which is our max\_sectors\_kb size).

-   With max\_sectors\_kb set to **4096**: The IOs issued on the devices linked to the ASM disks can be greater than 1MB (even if oracle requested 1MB IOs):

<!-- -->

    Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
    sdi               0.00     0.00    3.75    2.05 20173.15  2253.60  3866.68     0.17   29.72  28.79  16.70
    sdas              0.00     0.00    3.90    1.95 20992.35  1638.80  3868.57     0.18   31.08  29.91  17.50
    sdgj              0.00     0.00    3.75    2.00 20173.85  2253.20  3900.36     0.20   34.71  33.74  19.40

about 2MB in my case (3900 \* 512 bytes).

<span style="text-decoration:underline;">**Remarks:**</span>

-   It has been tested with ASM 11.2.0.4 on Linux x86-64 (without asmlib).
-   I’ll update this post with ASM 12c results as soon as I can (if something new needs to be told).

<span style="text-decoration:underline;">**Conclusion:**</span>

-   The arb process always request IOs of 1MB (whatever the allocation unit is).
-   The arb process always request IOs of 1MB (whatever the max\_sectors\_kb is): Then it looks like arb doesn't probe the IO capabilities of the associated device.
-   The arb IOs requests are grouped and submitted at the same time depending of the diskgroup allocation unit.
-   The kernel splits the arb IOs requests if max\_sectors\_kb is &lt; 1MB.
-   The kernel try to merge the arb IOs requests if max\_sectors\_kb is &gt; 1MB.

Thanks to Frits Hoogland for his "[Extra huge database IOs series](https://fritshoogland.wordpress.com/2013/07/14/extra-huge-database-ios-part-3/)" and for the time he spent answering the questions I asked.

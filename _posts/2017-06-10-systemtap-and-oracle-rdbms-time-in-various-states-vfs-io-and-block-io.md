---
layout: post
title: 'SystemTap and Oracle RDBMS: Time in Various States, VFS I/O and Block I/O'
date: 2017-06-10 17:41:22.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- SystemTap
tags: [oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6279346573159866368&type=U&a=bBSW
  _publicize_job_id: '5975340918'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/873580884565393410";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
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
permalink: "/2017/06/10/systemtap-and-oracle-rdbms-time-in-various-states-vfs-io-and-block-io/"
---

Introduction
------------

Now that I am able to [aggregate SytemTap probes by Oracle database](https://bdrouvot.wordpress.com/2017/06/05/systemtap-aggregate-by-database/), it's time to create several scripts in a toolkit. The toolkit is available in this [github repository](https://github.com/bdrouvot/SystemTap).

Let's describe 3 new members of the toolkit:

-   schedtimes\_per\_db.stp: To track time databases spend in various states
-   vfsio\_per\_db.stp: To track I/O by database through the Virtual File System ([vfs)](https://en.wikipedia.org/wiki/Virtual_file_system) layer
-   blkio\_per\_db.stp: To track I/O by database through the [block IO layer](https://en.wikipedia.org/wiki/I/O_scheduling)

schedtimes\_per\_db
-------------------

This script tracks the time databases spend in various states. It also reports the time spend by non oracle database.

### Usage:

    $> stap -g ./schedtimes_per_db.stp <oracle uid> <refresh time ms>

### Output example:

    $> stap -g ./schedtimes_per_db.stp 54321 10000

    ------------------------------------------------------------------
    DBNAME    :    run(us)  sleep(us) iowait(us) queued(us)  total(us)
    ------------------------------------------------------------------
    NOT_A_DB  :     447327  200561911       1328     517522  201528088
    BDT       :      42277  316189082          0      69355  316300714
    VBDT      :      74426  326694570          0      77489  326846485

vfsio\_per\_db
--------------

This script tracks the database I/O through the VFS layer. It also reports the I/O in this layer for non oracle database.

### Usage:

    $> stap -g ./vfsio_per_db.stp <oracle uid> <refresh time ms>

### Output example:

    $> stap -g ./vfsio_per_db.stp 54321 10000

    ------------------------------------------------------------------------
    DBNAME      : NB_READ   READ_KB   NB_WRITE  WRITE_KB  NB_TOTAL  TOTAL_KB
    ------------------------------------------------------------------------
    BDTS        : 110       347       6         96        116       443
    NOT_A_DB    : 89        11        2         0         91        11

blkio\_per\_db
--------------

This script tracks the database I/O through the block IO layer. It also reports the I/O in this layer for non oracle database.

### Usage:

    $> stap -g ./blkio_per_db.stp <oracle uid> <refresh time ms>

### Output example:

    $> stap -g ./blkio_per_db.stp 54321 10000

    ------------------------------------------------------------------------
    DBNAME      : NB_READ   READ_KB   NB_WRITE  WRITE_KB  NB_TOTAL  TOTAL_KB
    ------------------------------------------------------------------------
    BDTS        : 9690      110768    18        192       9708      110960
    NOT_A_DB    : 0         0         6         560       6         560

Remarks
-------

-   The schedtimes\_per\_db script is mainly inspired by this [one](https://sourceware.org/systemtap/examples/process/schedtimes.stp) (full credit goes to the authors).
-   Why is it  interesting to look at the vfs layer? Answers are in this awesome File System Latency series (see parts [1](http://dtrace.org/blogs/brendan/2011/05/11/file-system-latency-part-1/), [2](http://dtrace.org/blogs/brendan/2011/05/13/file-system-latency-part-2/), [3](http://dtrace.org/blogs/brendan/2011/05/18/file-system-latency-part-3/), [4](http://dtrace.org/blogs/brendan/2011/05/24/file-system-latency-part-4/) and [5](http://dtrace.org/blogs/brendan/2011/06/03/file-system-latency-part-5/)) from Brendan Gregg.
-   In this post the word database stands for "all the foreground and background processes linked to an oracle database".
-   In a consolidated environment, having a view per database can be very useful.

Conclusion
----------

The toolkit has been created and 3 new members are part of it. Expect from it to grow a lot.

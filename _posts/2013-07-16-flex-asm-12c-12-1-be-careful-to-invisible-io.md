---
layout: post
title: 'Flex ASM 12c (12.1): be careful to “invisible” I/O !'
date: 2013-07-16 16:34:34.000000000 +02:00
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
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:213;}s:2:"wp";a:1:{i:0;i:36;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/07/16/flex-asm-12c-12-1-be-careful-to-invisible-io/"
---

The starting point of this blog post is a talk that I had with my twitter friend Martin Berger ([@martinberx](https://twitter.com/martinberx)): He suggested me to test the [Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf) behavior with different ASM disks path per machine.

<span style="text-decoration:underline;">As the documentation states:</span>

[<img src="{{ site.baseurl }}/assets/images/asm_diskstring_121.png" class="aligncenter size-full wp-image-1219" width="620" height="272" alt="asm_diskstring_121" />](http://bdrouvot.files.wordpress.com/2013/07/asm_diskstring_121.png)

Different nodes might see the same disks under different names, however each instance must be able to use its ASM\_DISKSTRING to discover the same physical media as the other nodes in the cluster.

<span style="text-decoration:underline;">Not saying this is a good practice but as everything not forbidden is allowed let's give it a try that way:</span>

-   My [Flex ASM lab](http://bdrouvot.wordpress.com/2013/06/29/build-your-own-flex-asm-12c-lab-using-virtual-box/ "Build your own Flex ASM 12c lab using Virtual Box") is a 3 nodes RAC.
-   The ASM\_DISKSTRING is set to **/dev/asm\*** on the ASM instances.
-   I'll add a new disk with [udev rules](http://www.oracle-base.com/articles/linux/udev-scsi-rules-configuration-in-oracle-linux-5-and-6.php) in place on the 3 machines so that the new disk will be identified as:

<!-- -->

-   /dev/asm**1**-disk10 on racnode**1**
-   /dev/asm**2**-disk10 on racnode**2**
-   /dev/asm**3**-disk10 on racnode**3**

As you can see, the ASM\_DISKSTRING (/dev/asm\*) **is able to discover this new disk on the three nodes**. Please note this **is the same shared disk**, it is just identified by different path on each nodes.

<span style="text-decoration:underline;">On my Flex ASM lab, 2 ASM instances are running:</span>

    srvctl status asm
    ASM is running on racnode2,racnode1

<span style="text-decoration:underline;">Let's create a diskgroup IOPS on this new disk (From the ASM1 instance):</span>

    . oraenv
    ORACLE_SID = [+ASM1] ? +ASM1
    The Oracle base remains unchanged with value /u01/app/oracle
    [oracle@racnode1 ~]$ sqlplus / as sysasm

    SQL*Plus: Release 12.1.0.1.0 Production on Tue Jul 16 12:03:59 2013

    Copyright (c) 1982, 2013, Oracle.  All rights reserved.

    Connected to:
    Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
    With the Real Application Clusters and Automatic Storage Management options

    SQL> create diskgroup IOPS external redundancy disk '/dev/asm1-disk10';

    Diskgroup created.

    SQL> exit
    Disconnected from Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
    With the Real Application Clusters and Automatic Storage Management options

    [oracle@racnode1 ~]$ srvctl start diskgroup -g IOPS

    [oracle@racnode1 ~]$ srvctl status diskgroup -g IOPS
    Disk Group IOPS is running on racnode2,racnode1

<span style="text-decoration:underline;">So everything went fine. Let's check the disk from the ASM point of view:</span>

    SQL> l
      1  select
      2  i.instance_name,g.name,d.path
      3  from
      4  gv$instance i,gv$asm_diskgroup g, gv$asm_disk d
      5  where
      6  i.inst_id=g.inst_id
      7  and g.inst_id=d.inst_id
      8  and g.group_number=d.group_number
      9  and g.name='IOPS'
     10*
    SQL> /

    INSTANCE_NAME    NAME                           PATH
    ---------------- ------------------------------ --------------------
    +ASM1            IOPS                           /dev/asm1-disk10
    +ASM2            IOPS                           /dev/asm2-disk10

As you can see +ASM**1** discovered /dev/asm**1**-disk10 and +ASM**2** discovered /dev/asm**2**-disk10. This is expected and everything is ok so far.

Now, go on the third node racnode3, where there is no ASM instance.

Remember that on racnode**3** the new disk is /dev/asm**3**-disk10. Let's connect to the NOPBDT3 database instance and create a tablespace IOPS on the IOPS diskgroup.

    SQL> create tablespace IOPS datafile '+IOPS' size 1g;

    Tablespace created.

Perfect, everything is ok.  Now check v$asm\_disk from the NOPBDT3 database instance:

    SQL> select path from v$asm_disk where path like '%10';

    PATH
    --------------------------------------------------------------------------------
    /dev/asm2-disk10

As you can see the NOPBDT3 database instance is linked to the +ASM**2** instance (as it reports /dev/asm**2**-disk10)

But the NOPBDT**3** database instance located on racnode**3** access /dev/asm**3**-disk10.

    SQL> select instance_name from v$instance;

    INSTANCE_NAME
    ----------------
    NOPBDT3

    SQL> !ls -l /dev/asm2-disk10
    ls: cannot access /dev/asm2-disk10: No such file or directory

    SQL> !ls -l /dev/asm3-disk10
    brw-rw----. 1 oracle dba 8, 193 Jul 16 15:35 /dev/asm3-disk10

Ooooh wait !  The NOPBDT3 database instance access the disk /dev/asm**3**-disk10 which is **not recorded **into gv$asm\_disk.

<span style="text-decoration:underline;">So what if I launch [SLOB](http://kevinclosson.wordpress.com/2013/05/02/slob-2-a-significant-update-links-are-here/) locally on the NOPBDT3 database instance, are the metrics recorded ?</span>

First, let's setup SLOB on the IOPS tablespace:

    [oracle@racnode3 SLOB]$ ./setup.sh IOPS 3

Now, launch SLOB and check the I/O metrics thanks to my [asmiostat utility](http://bdrouvot.wordpress.com/2013/07/05/asm-io-statistics-utility-v2/ "ASM I/O Statistics Utility V2") that way:

     ./real_time.pl -type=asmiostat -show=inst,dbinst -dg=IOPS

With the following output:

[<img src="{{ site.baseurl }}/assets/images/metrics_not_recorded.png" class="aligncenter size-full wp-image-1230" width="620" height="128" alt="metrics_not_recorded" />](http://bdrouvot.files.wordpress.com/2013/07/metrics_not_recorded.png)

As you can see the metrics have not been recorded, while the IOPs have been done (/dev/asm3-disk10 is /dev/sdm):

    egrep -i "sdm|device" iostat.out | tail -4
    Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
    sdm               0.00     0.00  101.64    0.00     0.79     0.00    16.00     1.80   17.74   6.05  61.52
    Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
    sdm               0.00     0.00  109.34    0.00     0.85     0.00    16.00     1.82   16.66   5.70  62.28

Of course the same test launched from the NOPBDT2 database instance linked to the +ASM2 instance,would produce the following output:

[<img src="{{ site.baseurl }}/assets/images/metrics_recorded.png" class="aligncenter size-full wp-image-1233" width="620" height="127" alt="metrics_recorded" />](http://bdrouvot.files.wordpress.com/2013/07/metrics_recorded.png)

So the metrics are recorded as the database Instance is doing the IOPS on devices that the ASM instance is aware of (NOPBDT2 linked to +ASM1 would produce the "invisible" metrics).

<span style="text-decoration:underline;">**Important remark:**</span>

The metrics are not visible from the gv$asm\_disk view (from the ASM or the database instance), but there is a place where the **metrics are recorded: The gv$asm\_disk\_iostat view from the database instance** (Not the ASM one).

<span style="text-decoration:underline;">**Conclusion:**</span>

<span style="text-decoration:underline;">The "invisible" I/O (metrics not recorded into the **gv$asm\_disk** view) issue occurs if:</span>

1.  you set different disks path per machine (so per ASM instance) in a [Flex ASM](http://www.oracle.com/technetwork/products/cloud-storage/oracle-12c-asm-overview-1965430.pdf/) (12.1) configuration.
2.  and the database instance is attached to a remote ASM instance (then using different path).

So I would suggest to use the same path per machine for the ASM disks in a Flex ASM (12.1) configuration to avoid this issue.

**Update:** The asmiostat utility is not part of the real\_time.pl script anymore. A new utility called **asm\_metrics.pl** has been created. See "[ASM metrics are a gold mine. Welcome to asm\_metrics.pl, a new utility to extract and to manipulate them in real time](http://bdrouvot.wordpress.com/2013/10/04/asm-metrics-are-a-gold-mine-welcome-to-asm_metrics-pl-a-new-utility-to-extract-and-to-manipulate-them-in-real-time/ "ASM metrics are a gold mine. Welcome to asm_metrics.pl, a new utility to extract and to manipulate them in real time")" for more information.

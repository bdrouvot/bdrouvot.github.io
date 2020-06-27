---
layout: post
title: Measure the impact of DRBD on your PostgreSQL database thanks to pgio (the
  SLOB method for PostgreSQL)
date: 2018-06-02 10:18:26.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Postgresql
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _oembed_207f2a1ccab183a904e2d13a3edbf7c4: "{{unknown}}"
  _oembed_f76f86bef2ef8237aecfabb8fa612d99: "{{unknown}}"
  _oembed_d70477bc11e1d09ddc3a320e4f8e9cd3: "{{unknown}}"
  _oembed_186d1a647dd33dd1a86c4387f326cdf8: "{{unknown}}"
  _oembed_5a470f1afd9c3aa3f6f99f37a37783b9: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_5a470f1afd9c3aa3f6f99f37a37783b9: '1527918439'
  _oembed_df8ba5ac18e7dbeecb342b52752b8eb7: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_df8ba5ac18e7dbeecb342b52752b8eb7: '1527918487'
  _oembed_9176f4ebdab246631674eb797bca4e3e: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_9176f4ebdab246631674eb797bca4e3e: '1527918508'
  _oembed_a694b1883b7fc70b7209d889b062428d: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_a694b1883b7fc70b7209d889b062428d: '1527918529'
  _oembed_ce7f4fec0941ef624ae8bbc08bce2bb7: "{{unknown}}"
  _oembed_216ff34cdba35a4ab758abc8443a9174: "{{unknown}}"
  _oembed_bdd0d5873d6647e9846da4b48dba2ac8: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_bdd0d5873d6647e9846da4b48dba2ac8: '1527918640'
  _oembed_c1f5de1c247b891216c8bc33646bdb56: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_4683dfe325e03a2f3948bcfd7f6a25e2: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_c1f5de1c247b891216c8bc33646bdb56: '1527918640'
  _oembed_889706b5e09cc4c36d084c00a79eb106: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_4683dfe325e03a2f3948bcfd7f6a25e2: '1527918640'
  _oembed_3e258330db21c218fa151587c74dd366: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_889706b5e09cc4c36d084c00a79eb106: '1527918640'
  _oembed_53f1e5b99dd201d3a1100ae3f6b3a575: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_3e258330db21c218fa151587c74dd366: '1527918640'
  _oembed_time_53f1e5b99dd201d3a1100ae3f6b3a575: '1527918640'
  _oembed_cd289c704dfbb4011577895f7d497074: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_cd289c704dfbb4011577895f7d497074: '1527918641'
  _oembed_7fa4f7b4d911d2e3eb14c41db38d7904: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_7fa4f7b4d911d2e3eb14c41db38d7904: '1527918642'
  _oembed_22c8ac323b22a3ec5a6384f2e1ac4e82: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6408607570881179648&type=U&a=--DF
  _oembed_1cb6ffce18f63ce7e8d38bbfbc58fb1b: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_1cb6ffce18f63ce7e8d38bbfbc58fb1b: '1527931109'
  _oembed_d7723e34f1b498c1b8863539db6df0af: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_d7723e34f1b498c1b8863539db6df0af: '1527931109'
  timeline_notification: '1527931109'
  _oembed_833559f410d917608dd7ebf93ad014f5: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_833559f410d917608dd7ebf93ad014f5: '1527931109'
  _publicize_job_id: '18522695812'
  _oembed_75a20ada9e0b90dbe05dc982e5560d8b: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_75a20ada9e0b90dbe05dc982e5560d8b: '1527931110'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1002841882735636480";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _oembed_time_22c8ac323b22a3ec5a6384f2e1ac4e82: '1527931112'
  _oembed_460883d4bde2765a3001880a49b948f1: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_460883d4bde2765a3001880a49b948f1: '1527931113'
  _oembed_ed80e489937b830a6ea3939d2c0ce9b6: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_ed80e489937b830a6ea3939d2c0ce9b6: '1527931113'
  _oembed_02a408729659f6dc7da6db28d6ebfb07: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_02a408729659f6dc7da6db28d6ebfb07: '1527931113'
  _oembed_32d1aa31d1beb8e6cfa322b9528d6017: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_32d1aa31d1beb8e6cfa322b9528d6017: '1527931113'
  _oembed_d070099e0d4830a57a6edb41735515f4: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_d070099e0d4830a57a6edb41735515f4: '1527931113'
  _oembed_a4fcef9a8c9f58c74848ae36133b8454: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_a4fcef9a8c9f58c74848ae36133b8454: '1527931114'
  _oembed_8204a31224b15f3b8f62143fe3a2f024: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_8204a31224b15f3b8f62143fe3a2f024: '1527931114'
  _oembed_490cedbd4518b57fb701717b3a7269ea: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_490cedbd4518b57fb701717b3a7269ea: '1531575605'
  _oembed_1408bb6c1a1853f32c062d368305d7fc: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_1408bb6c1a1853f32c062d368305d7fc: '1531575605'
  _oembed_07d95bd281bf01ae0f19f258181c1103: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_07d95bd281bf01ae0f19f258181c1103: '1531575606'
  _oembed_ad815054ad547adb78d16b68f85206b1: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_time_ad815054ad547adb78d16b68f85206b1: '1531575606'
  _oembed_9d7b2e9b0f794628bec966585305ec94: '<a href="https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part IV: How To Reduce The Amount
    of Memory In The Linux Page Cache For Testing&nbsp;Purposes.</a>'
  _oembed_b8aebb39b30d0cb22e46c2ce94bd10d9: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part I: The Beta pgio README&nbsp;File.</a>'
  _oembed_time_b8aebb39b30d0cb22e46c2ce94bd10d9: '1578137198'
  _oembed_b870392fe216f838c215062901218261: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL) Part II: Bulk Data&nbsp;Loading.</a>'
  _oembed_time_b870392fe216f838c215062901218261: '1578137198'
  _oembed_665181a71f11405bf5257f04c4748209: '<a href="https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/">Sneak
    Preview of pgio (The SLOB Method for PostgreSQL)  Part III: Link To The Full README
    file for Beta pgio&nbsp;v0.9.</a>'
  _oembed_time_665181a71f11405bf5257f04c4748209: '1578137198'
  _oembed_time_9d7b2e9b0f794628bec966585305ec94: '1578137199'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2018/06/02/measure-the-impact-of-drbd-on-your-postgresql-database-thanks-to-pgio-the-slob-method-for-postgresql/"
---

You might want to replicate a PostgreSQL database thanks to [DRBD](https://en.wikipedia.org/wiki/Distributed_Replicated_Block_Device). In this case you should measure the impact of your DRBD setup, especially if you plan to use DRBD in sync mode. As I am a lucky beta tester of [pgio](https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/) (the SLOB method for PostgreSQL), let's use it to measure the impact.

### DRBD configuration

The purpose of this post is not to explain how to set up DRBD. The DRBD configuration that will be used in this post is the following:

-   Primary host: ubdrbd1
-   Secondary host: ubdrbd2

Configuration:

    root@ubdrbd1:/etc# cat drbd.conf
    global {
    usage-count no;
    }
    common {
    protocol C;
    }
    resource postgresql {
    on ubdrbd1 {
    device /dev/drbd1;
    disk /dev/sdb1;
    address 172.16.170.153:7789;
    meta-disk internal;
    }
    on ubdrbd2 {
    device /dev/drbd1;
    disk /dev/sdb1;
    address 172.16.170.155:7789;
    meta-disk internal;
    }
    }

The [C protocol](https://docs.linbit.com/doc/users-guide-84/s-replication-protocols/) is the synchronous one, so that local write operations on the primary node are considered completed only after both the local and the remote disk write (on /dev/sdb1) have been confirmed.

To ensure that PostgreSQL writes in this replicated device, let's create a filesystem on it:

    root@ubdrbd1:/etc# mkfs.ext4 /dev/drbd1
    mke2fs 1.44.1 (24-Mar-2018)
    Creating filesystem with 5242455 4k blocks and 1310720 inodes
    Filesystem UUID: a7cc203b-39fa-46b3-a707-f330f72ca5b1
    Superblock backups stored on blocks:
    32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
    4096000

    Allocating group tables: done
    Writing inode tables: done
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done

And mount it, in sync mode, on the postgresql default data location:

    root@ubdrbd1:~# mkdir /var/lib/postgresql
    root@ubdrbd1:~# mount -o sync /dev/drbd1 /var/lib/postgresql

### pgio setup

Once PostgreSQL is up and running, we can setup pgio, this is as easy as that:

    postgres@ubdrbd1:~/$ tar -xvf pgio-0.9.tar
    postgres@ubdrbd1:~/$ cd pgio

then, edit the configuration file to suit your needs. In our current case, the configuration file being used is the following:

    postgres@ubdrbd1:~/pgio$ cat pgio.conf
    UPDATE_PCT=10
    RUN_TIME=120
    NUM_SCHEMAS=1
    NUM_THREADS=1
    WORK_UNIT=255
    UPDATE_WORK_UNIT=8
    SCALE=200M
    DBNAME=pgio
    CONNECT_STRING="pgio"
    CREATE_BASE_TABLE=TRUE

As you can see:

-   I want 1 schema and 1 thread by schema
-   I have set UPDATE\_PCT so that 10% of calls will do an UPDATE (to test writes, and so DRBD replication impact)
-   I kept the default work unit to read 255 blocks and, for those 10% updates, update 8 blocks only

Now, create the pgio tablespace and pgio database:

    postgres=# create tablespace pgio location '/var/lib/postgresql/pgdata';
    CREATE TABLESPACE
    postgres=# create database pgio tablespace pgio;
    CREATE DATABASE

and run setup.sh to load the data:

    postgres@ubdrbd1:~/pgio$ ./setup.sh

    Job info:      Loading 200M scale into 1 schemas as per pgio.conf->NUM_SCHEMAS.
    Batching info: Loading 1 schemas per batch as per pgio.conf->NUM_THREADS.
    Base table loading time: 3 seconds.
    Waiting for batch. Global schema count: 1. Elapsed: 0 seconds.
    Waiting for batch. Global schema count: 1. Elapsed: 13 seconds.

    Group data loading phase complete.         Elapsed: 13 seconds.

check the content:

    postgres@ubdrbd1:~/pgio$ echo '\d+' | psql pgio
                          List of relations
     Schema |   Name    | Type  |  Owner   |  Size  | Description
    --------+-----------+-------+----------+--------+-------------
     public | pgio1     | table | postgres | 200 MB |
     public | pgio_base | table | postgres | 29 MB  |
    (2 rows)

Now, let's run 2 times pgio:

-   One time with DRBD not active
-   One time with DRBD active

For each test, the pgio database will be recreated and the setup will be launched. Also the same pgio run duration will be applied (to compare the exact same things).

### First run: DRBD not active

The result is the following:

    postgres@ubdrbd1:~/pgio$ ./runit.sh
    Date: Sat Jun 2 05:09:46 UTC 2018
    Database connect string: "pgio".
    Shared buffers: 128MB.
    Testing 1 schemas with 1 thread(s) accessing 200M (25600 blocks) of each schema.
    Running iostat, vmstat and mpstat on current host--in background.
    Launching sessions. 1 schema(s) will be accessed by 1 thread(s) each.
    pg_stat_database stats:
    datname| blks_hit| blks_read|tup_returned|tup_fetched|tup_updated
    BEFORE: pgio | 98053 | 34640 | 34309 | 6780 | 8
    AFTER: pgio | 19467796 | 11956992 | 30807460 | 29665212 | 115478
    DBNAME: pgio. 1 schemas, 1 threads(each). Run time: 120 seconds. RIOPS >99352< CACHE_HITS/s >161414<

As you can see, 115470 tuples have been updated during this 120 seconds run **without** DRBD in place.

### Second run: DRBD active

For the purpose of this post, let's **simulate** "slowness" on the DRBD replication. To do so, we can apply some throttling on the DRBD target device thanks to the [blkio cgroup](https://www.kernel.org/doc/Documentation/cgroup-v1/blkio-controller.txt). I would suggest to read [Frits Hoogland post](https://fritshoogland.wordpress.com/2012/12/15/throttling-io-with-linux/) to see how it can be implemented.

My cgroup configuration on the DRBD secondary server is the following:

    root@ubdrbd2:~# ls -l /dev/sdb
    brw-rw---- 1 root disk 8, 16 Jun 1 15:16 /dev/sdb

    root@ubdrbd2:~# cat /etc/cgconfig.conf
    mount {
    blkio = /cgroup/blkio;
    }

    group iothrottle {
    blkio {
    blkio.throttle.write_iops_device="8:16 500";
    }
    }

    root@ubdrbd2:~# cat /etc/cgrules.conf
    * blkio /iothrottle

Means: we want to ensure that the number of writes per second on /dev/sdb will be limited to 500 for all the processes.

Let's check that DRDB is active:

    root@ubdrbd1:~# drbdsetup status --verbose --statistics
    postgresql role:Primary suspended:no
        write-ordering:flush
      volume:0 minor:1 disk:UpToDate
          size:20969820 read:453661 written:6472288 al-writes:215 bm-writes:0 upper-pending:4 lower-pending:0 al-suspended:no blocked:no
      peer connection:Connected role:Secondary congested:no
        volume:0 replication:Established peer-disk:UpToDate resync-suspended:no
            received:0 sent:5161968 out-of-sync:0 pending:4 unacked:0

    root@ubdrbd2:~# drbdsetup status --verbose --statistics
    postgresql role:Secondary suspended:no
        write-ordering:flush
      volume:0 minor:1 disk:UpToDate
          size:20969820 read:0 written:5296960 al-writes:0 bm-writes:0 upper-pending:0 lower-pending:2 al-suspended:no blocked:no
      peer connection:Connected role:Primary congested:no
        volume:0 replication:Established peer-disk:UpToDate resync-suspended:no
            received:5296968 sent:0 out-of-sync:0 pending:0 unacked:2

With this configuration in place, let's run pgio on the source machine:

    postgres@ubdrbd1:~/pgio$ ./runit.sh
    Date: Sat Jun 2 05:37:03 UTC 2018
    Database connect string: "pgio".
    Shared buffers: 128MB.
    Testing 1 schemas with 1 thread(s) accessing 200M (25600 blocks) of each schema.
    Running iostat, vmstat and mpstat on current host--in background.
    Launching sessions. 1 schema(s) will be accessed by 1 thread(s) each.
    pg_stat_database stats:
    datname| blks_hit| blks_read|tup_returned|tup_fetched|tup_updated
    BEFORE: pgio | 111461 | 44099 | 35967 | 5640 | 6
    AFTER: pgio | 3879414 | 2132201 | 5799491 | 5766888 | 22425
    DBNAME: pgio. 1 schemas, 1 threads(each). Run time: 120 seconds. RIOPS >17400< CACHE_HITS/s >31399<

As you can see, 22419 tuples have been updated during this 120 seconds run **with** DRBD and blkio throttling in place (this is far less than the 115470 observed in the first test).

pgio also provides snapshots of iostat, vmstat and mpstat, so that it is easy to check that the throttling was in place (writes per second &lt; 500 on the source due to the throttling on the target):

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sdb              0.00  425.42      0.00      3.20     0.00    84.41   0.00  16.56    0.00    0.18   0.08     0.00     7.71   0.16   6.64
    sdb              0.00  415.18      0.00      3.16     0.00    74.92   0.00  15.29    0.00    0.20   0.08     0.00     7.80   0.17   7.26
    .
    .

compares to:

    Device            r/s     w/s     rMB/s     wMB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
    sdb              0.00 1311.07      0.00     10.14     0.00    62.98   0.00   4.58    0.00    0.10   0.14     0.00     7.92   0.10  13.70
    sdb              0.00 1252.05      0.00      9.67     0.00    62.33   0.00   4.74    0.00    0.11   0.13     0.00     7.91   0.10  13.01
    .
    .

with DRBD and throttling, both not in place.

### So?

Thanks to pgio, we have been able to measure the impact of our DRBD replication setup on PostgreSQL by executing the exact same workload during both tests. We simulated a poor performing DRBD replication by using blkio throttling on the secondary node.

### Want to read more about pgio?

https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgressql-part-i-the-beta-pgio-readme-file/

https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-ii-bulk-data-loading/

https://kevinclosson.net/2018/05/22/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iii-link-to-the-full-readme-file-for-beta-pgio-v0-9/

https://kevinclosson.net/2018/05/23/sneak-preview-of-pgio-the-slob-method-for-postgresql-part-iv-how-to-reduce-the-amount-of-memory-in-the-linux-page-cache-for-testing-purposes/

<https://blog.dbi-services.com/postgres-the-fsync-issue-and-pgio-the-slob-method-for-postgresql/>

<https://blog.dbi-services.com/which-bitnami-service-to-choose-in-the-oracle-cloud-infrastructure/>

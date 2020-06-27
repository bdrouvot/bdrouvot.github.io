---
layout: post
title: Reduce resource consumption and clone in seconds your oracle virtual environment
  on your laptop using linux containers and btrfs
date: 2014-04-25 15:56:23.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Other
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_facebook_url: https://facebook.com/
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/cP2nmASG9Eg
  _wpas_done_5547632: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"101126738655139704850";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/6cGtJ7gv2n
  _wpas_done_2225791: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5865473208408379393&type=U&a=t67o
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
permalink: "/2014/04/25/reduce-resource-consumption-and-clone-in-seconds-your-oracle-virtual-environment-on-your-laptop-using-linux-containers-and-btrfs/"
---

Last week I wanted to create a new oracle virtual machine on my **Laptop** and I discovered that the disk space on my SSD device was more or less exhausted. Then I looked for a solution **to minimize the disk usage** of my oracle virtual machines.

<span style="text-decoration:underline;">**I find a way to achieve my need based on those technologies:**</span>

-   [Linux Containers](http://linuxcontainers.org/): a **lightweight** virtualization solution for Linux.
-   [btrfs file system](http://www.oracle.com/technetwork/articles/servers-storage-admin/gettingstarted-btrfs-1695246.html): a file system that allows to **create snapshots almost instantly** and consume virtually **no additional disk space** as a snapshot and the original it was taken from **initially share all** of the same data blocks.

In this post I will show how I create an oracle environment on my laptop with those technologies and how we can clone a container, an ORACLE\_HOME and a database in a few seconds with initially **no additional disk space**.

**<span style="text-decoration:underline;">Note:</span> ** The goal is to create a "test" environment on your laptop. I would not suggest to follow this installation process on a "real" system ;-)

<span style="color:#008000;">**PREPARATION PHASE**</span>

**Step 1**: let's create a [OEL 6.5 virtual machine](http://bdrouvot.wordpress.com/create-a-simple-oracle-oel-vm-with-virtualbox/ "Create a simple Oracle OEL VM with virtualbox") (named lxc) using virtualbox. This virtual machine will host our Linux containers, oracle software and databases.

**Step 2**: Install lxc and btrfs into the virtual machine created into step 1.

    [root@lxc ~]# yum install btrfs-progs
    [root@lxc ~]# yum install lxc
    [root@lxc ~]# service cgconfig start
    [root@lxc ~]# chkconfig cgconfig on
    [root@lxc ~]# service libvirtd start
    [root@lxc ~]# chkconfig libvirtd on

**Step 3**: Add a btrfs file system into the virtual machine (This file system will receive the oracle software and databases). To do so, [add a disk to your virtualbox machine](http://bdrouvot.wordpress.com/add-a-disk-to-your-virtualbox-machine/ "Add a disk to your virtualbox machine") created in step 1, start the machine and launch the fs creation:

    [root@lxc ~]# mkfs.btrfs /dev/sdb
    [root@lxc ~]# mkdir /btrfs
    [root@lxc ~]# mount /dev/sdb /btrfs
    [root@lxc ~]# chown oracle:dba /btrfs
    [root@lxc ~]# blkid /dev/sdb
    /dev/sdb: UUID="3f6f7b51-7662-4d81-9a29-195e167e54ff" UUID_SUB="1d79e0d0-933d-4c65-9939-9614375da5e1" TYPE="btrfs"
    Retrieve the UUID and put it into the fstab
    [root@lxc ~]# cat >> /etc/fstab << EOF
    UUID=3f6f7b51-7662-4d81-9a29-195e167e54ff /btrfs btrfs    defaults   0 0
    EOF

**Step 4**: Add a btrfs file system into the virtual machine (This file system will receive the linux containers). To do so, [add a disk to your virtualbox machine](http://bdrouvot.wordpress.com/add-a-disk-to-your-virtualbox-machine/ "Add a disk to your virtualbox machine") created in step 1, start the machine and launch the fs creation:

    [root@lxc ~]# mkfs.btrfs /dev/sdc
    [root@lxc ~]# mkdir /container
    [root@lxc ~]# mount /dev/sdc /container
    [root@lxc ~]# blkid /dev/sdc
    /dev/sdc: UUID="8a565bfd-2deb-4d02-bd91-a81c4cc9eb54" UUID_SUB="44cb0a14-afc5-48eb-bc60-4c24b9b02ab1" TYPE="btrfs"
    Retrieve the UUID and put it into the fstab
    [root@lxc ~]# cat  >> /etc/fstab << EOF
    UUID=8a565bfd-2deb-4d02-bd91-a81c4cc9eb54 /container btrfs    defaults   0 0
    EOF

**Step 5**: Create btrfs subvolume for the database software and databases.

    [root@lxc ~]# btrfs subvolume create /btrfs/u01
    Create subvolume '/btrfs/u01'
    [root@lxc ~]# btrfs subvolume create /btrfs/databases
    Create subvolume '/btrfs/databases'
    [root@lxc ~]# chown oracle:dba /btrfs/u01
    [root@lxc ~]# chown oracle:dba /btrfs/databases

**Step 6**: add the hostname into */etc/hosts*

    [root@lxc btrfs]# cat /etc/hosts
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 lxc
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

and [Install the 12cR1 database software](http://bdrouvot.wordpress.com/install-12cr1-database-software/ "Install 12cR1 database software") with:

*Oracle Base: /btrfs/u01*  
*Software location: /btrfs/u01/product/12.1.0/dbhome\_1*  
*Inventory directory: /btrfs/u01/oraInventory*  
*oraInventory Group Name: dba*

**Step 7**: [Create a simple database](http://bdrouvot.wordpress.com/create-a-simple-test-database-into-a-single-filesystem/ "Create a simple test database into a single filesystem") with datafiles, redologs and controlfile located into the */btrfs/databases* folder.

**Step 8**: Create a linux container (using oracle template) that will be the source of all our new containers.

    lxc-create --name cont_source -B btrfs --template oracle -- --url http://public-yum.oracle.com -R 6.latest -r "perl sudo oracle-rdbms-server-12cR1-preinstall"

**Here we are: we are now ready to clone all of this into a new linux container in seconds without any additional disk usage.**

<span style="color:#008000;">**CLONING PHASE**</span>

First, let's take a picture of the current disk usage:

    [root@lxc ~]# df -h 
    Filesystem                  Size  Used Avail Use% Mounted on
    /dev/mapper/vg_lxc-lv_root   45G  3.1G   40G   8% /
    tmpfs                       2.0G     0  2.0G   0% /dev/shm
    /dev/sda1                   477M   55M  398M  13% /boot
    /dev/sdb                     50G  6.4G   42G  14% /btrfs
    /dev/sdc                     50G  1.1G   48G   3% /container

**Clone step 1**:  Add into  */etc/security/limits.conf*  (If not, you won't be able to *su - oracle* into the linux containers)

    *   soft   nofile    1024
    *   hard   nofile    65536
    *   soft   nproc    2047
    *   hard   nproc    16384
    *   soft   stack    10240
    *   hard   stack    32768

and reboot the virtual machine created into step 1.

**Clone step 2**:  clone the linux container created during step 8 to a new one named for example *dbhost1*.

    [root@lxc oradata]# time lxc-clone -s -t btrfs -o cont_source -n dbhost1
    Tweaking configuration
    Copying rootfs...
    Create a snapshot of '/container/cont_source/rootfs' in '/container/dbhost1/rootfs'
    Updating rootfs...
    'dbhost1' created

    real    0m0.716s
    user    0m0.023s
    sys     0m0.029s

**Clone step 3**: clone the database software.

    [root@lxc oradata]# time btrfs su snapshot /btrfs/u01 /btrfs/u01_dbhost1
    Create a snapshot of '/btrfs/u01' in '/btrfs/u01_dbhost1'

    real    0m0.038s
    user    0m0.000s
    sys     0m0.006s

**Clone step 4**: clone the database (shutdown immediate before)

    [root@lxc oradata]# time btrfs su snapshot /btrfs/databases /btrfs/databases_dbhost1
    Create a snapshot of '/btrfs/databases' in '/btrfs/databases_dbhost1'

    real    0m0.041s
    user    0m0.002s
    sys     0m0.006s

**Clone step 5**: Link the new container to this database software and database clones. Edit */container/dbhost1/config* and put:

    lxc.mount.entry=/btrfs/u01_dbhost1 /container/dbhost1/rootfs/btrfs/u01 none rw,bind 0 0
    lxc.mount.entry=/btrfs/databases_dbhost1 /container/dbhost1/rootfs/btrfs/databases none rw,bind 0 0

**Clone step 6**: Copy dbhome, oraenv and coraenv and start the new container *dbhost1*

    [root@lxc ~]# cp -p /usr/local/bin/coraenv /usr/local/bin/dbhome /usr/local/bin/oraenv /container/dbhost1/rootfs/usr/local/bin
    [root@lxc oradata]# mkdir -p /container/dbhost1/rootfs/btrfs/u01
    [root@lxc oradata]# mkdir -p /container/dbhost1/rootfs/btrfs/databases
    [root@lxc oradata]# lxc-start -n dbhost1

**Clone step 7**: connect to the **new container** (default password for root is root), create the oratab, and start the database.

    [root@lxc ~]# lxc-console -n dbhost1
    [root@dbhost1 ~]# su  - oracle
    [oracle@dbhost1 dbs]$ . oraenv
    ORACLE_SID = [BDTDB] ? 
    The Oracle base remains unchanged with value /btrfs/u01
    [oracle@dbhost1 dbs]$ echo "startup" | sqlplus / as sysdba

    SQL*Plus: Release 12.1.0.1.0 Production on Fri Apr 25 08:01:52 2014

    Copyright (c) 1982, 2013, Oracle.  All rights reserved.

    Connected to an idle instance.

    SQL> ORACLE instance started.

    Total System Global Area 1570009088 bytes
    Fixed Size                  2288776 bytes
    Variable Size             905970552 bytes
    Database Buffers          654311424 bytes
    Redo Buffers                7438336 bytes
    Database mounted.
    Database opened.

Check the disk usage on our virtual machine created into step 1:

    [root@lxc ~]# df -h
    Filesystem                  Size  Used Avail Use% Mounted on
    /dev/mapper/vg_lxc-lv_root   45G  3.1G   40G   8% /
    tmpfs                       2.0G     0  2.0G   0% /dev/shm
    /dev/sda1                   477M   55M  398M  13% /boot
    /dev/sdb                     50G  6.4G   42G  14% /btrfs
    /dev/sdc                     50G  1.1G   48G   3% /container

Et voila ;-) we created a new linux container, a new database home and a new database with initially **no additional disk space**.

<span style="text-decoration:underline;">**Remarks:**</span>

-   Jason Arneil did a demo on Linux containers [here](http://blog.e-dba.com/blog/2014/01/06/linux-containers-demo/).
-   Ludovico Caldara also shows how to save disk space when building a RAC with [virtualbox linked clones](http://www.ludovicocaldara.net/dba/multinode-rac12c-virtualbox-cloning/).
-   If you are interested in how oracle databases/rac and lxc can work together, I suggest to read Alvaro Miranda's stuff [here](http://kikitux.net/) (I mainly took my inspiration from his blog).
-   Ofir Manor describes another Linux container use case related to Hadoop cluster [here](http://ofirm.wordpress.com/2014/01/05/creating-a-virtualized-fully-distributed-hadoop-cluster-using-linux-containers/).
-   Cloning the database software as I did here is not the "right" way (See MOS Doc ID 1154613.1). But this suits me fine for my laptop environment.

<span style="text-decoration:underline;">**Conclusion:**</span>

We can create a new container, a new ORACLE\_HOME and a new database, **reducing resource consumption** (specially disk) on our laptop in **a few seconds**.

**Again:** I would not suggest to use all of this on a "real" system. But for a test environment on a laptop it sound goods to me.

I hope you will save some disk space on your laptop thanks to this ;-).

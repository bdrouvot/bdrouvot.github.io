---
layout: post
title: ASM free and usable space in case of Imbalanced Diskgroup
date: 2014-02-18 11:59:26.000000000 +01:00
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
  _publicize_pending: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5841495973154947072&type=U&a=5l48
  publicize_facebook_url: https://facebook.com/
  publicize_google_plus_url: https://plus.google.com/101126738655139704850/posts/VtPWXCn7ojJ
  _wpas_done_5547632: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"101126738655139704850";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/axPGNgQbub
  _wpas_done_2225791: '1'
  _wpas_done_2077996: '1'
  geo_public: '0'
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
permalink: "/2014/02/18/asm-free-and-usable-space-in-case-of-imbalanced-diskgroup/"
---

In case of "normal" situation (I mean no Imbalanced Disks) you should be able to easily know how much space is available into the ASM Diskgroup.

Harald van Breederode already explained it in detail into [this blog post](http://prutser.wordpress.com/2013/01/03/demystifying-asm-required_mirror_free_mb-and-usable_file_mb/).

<span style="text-decoration:underline;">But now suppose the Diskgroup has **imbalanced** Disks: It can occurs for several reasons:</span>

-   A rebalance operation has been aborted/halted.
-   A rebalance operation is waiting.
-   Disks are not of the same size.

<span style="text-decoration:underline;">Let's simulate an Imbalanced situation and compute the usable free space:</span>

First I created a "balanced" diskgroup with normal redundancy.

    SQL> !kfod
    --------------------------------------------------------------------------------
     Disk          Size Path                                     User     Group   
    ================================================================================
       1:     225280 Mb /dev/san/JMO7C43D06                      oracle   dba  
       2:      56320 Mb /dev/san/JMO7C52D06                      oracle   dba  
       3:      56320 Mb /dev/san/JMO7C61D06                      oracle   dba  
       4:     225280 Mb /dev/san/WIN7C43D06                      oracle   dba  
       5:      56320 Mb /dev/san/WIN7C52D06                      oracle   dba  
       6:      56320 Mb /dev/san/WIN7C61D06                      oracle   dba  
    --------------------------------------------------------------------------------

    SQL> create diskgroup BDT normal redundancy failgroup F1 disk '/dev/san/JMO7C52D06' failgroup F2 disk '/dev/san/WIN7C52D06';

    Diskgroup created.

Let's check its free space with **asmcmd** and with **asm\_free\_usable\_imbalance.sql** (I'll share the code later into this post) built to report the free/usable space of Imbalanced Diskgroup:

    SQL> !asmcmd lsdg BDT
    State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
    MOUNTED  NORMAL  N         512   4096  1048576    112640   112538                0           56269              0             N  BDT/

    SQL> @asm_free_usable_imbalance.sql

    NAME                              FREE_MB
    ------------------------------ ----------
    BDT                                 56269

Well, we can see that from asmcmd (112538 / 2) 56269 Mb is free/usable and that the asm\_free\_usable\_imbalance.sql reports the same value.

So that the asm\_free\_usable\_imbalance.sql script **works at least **with "Balanced" diskgroup ;-).

<span style="text-decoration:underline;">Now let's produce an Imbalanced situation that way:</span>

-   Add datafiles into the Diskgroup.
-   Add Disks into the failgroups without trigerring a rebalance operation (asm\_power\_limit = 0).

<!-- -->

    SQL> create tablespace BDT datafile '+BDT' size 30g;

    Tablespace created.

    SQL> alter tablespace BDT add datafile '+BDT' size 10g;

    Tablespace altered.

    SQL> alter tablespace BDT add datafile '+BDT' size 20g;
    alter tablespace BDT add datafile '+BDT' size 20g
    *
    ERROR at line 1:
    ORA-01119: error in creating database file '+BDT'
    ORA-17502: ksfdcre:4 Failed to create file +BDT
    ORA-15041: diskgroup "BDT" space exhausted

So, we can't add a 20 GB datafile anymore (sounds obvious as the free space was about 55 GB and we already added 40 GB into the diskgroup).

Now add disks of different size without trigerring a rebalance operation:

    SQL> !kfod
    --------------------------------------------------------------------------------
     Disk          Size Path                                     User     Group   
    ================================================================================
       1:     225280 Mb /dev/san/JMO7C43D06                      oracle   dba  
       2:      56320 Mb /dev/san/JMO7C61D06                      oracle   dba  
       3:     225280 Mb /dev/san/WIN7C43D06                      oracle   dba  
       4:      56320 Mb /dev/san/WIN7C61D06                      oracle   dba  
    --------------------------------------------------------------------------------

    SQL> alter diskgroup BDT add failgroup F1 disk '/dev/san/JMO7C43D06','/dev/san/JMO7C61D06' failgroup F2 disk '/dev/san/WIN7C43D06','/dev/san/WIN7C61D06';

    Diskgroup altered.

Verify that the rebalance is waiting:

    SQL> select * from v$asm_operation where group_number = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT');

    GROUP_NUMBER OPERA STAT      POWER     ACTUAL      SOFAR   EST_WORK   EST_RATE EST_MINUTES ERROR_CODE
    ------------ ----- ---- ---------- ---------- ---------- ---------- ---------- ----------- --------------------------------------------
               8 REBAL WAIT          0

and the disks usage:

    SQL> select failgroup,total_mb,FREE_MB from v$asm_disk where group_number = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT');

    FAILGROUP                        TOTAL_MB    FREE_MB
    ------------------------------ ---------- ----------
    F1                                  56320      56318
    F1                                 225280     225277
    F1                                  56320      15304
    F2                                 225280     225277
    F2                                  56320      15304
    F2                                  56320      56318

<span style="text-decoration:underline;">Here we are: **How much usable/free space is "really" available into this "Imbalanced" diskgroup ?**</span>

Let's see what asmcmd and asm\_free\_usable\_imbalance.sql report:

    SQL> !asmcmd lsdg BDT
    State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
    MOUNTED  NORMAL  Y         512   4096  1048576    675840   593798           225280          184259              0             N  BDT/

    SQL> @asm_free_usable_imbalance.sql

    NAME                              FREE_MB
    ------------------------------ ----------
    BDT                                 91824

Now we have to remember that for **primary** extents, ASM will allocate new extents in such way as to distribute each **file equally and evenly across all disks and to fill all disks evenly**. Thus every disk is maintained at the **same percentage full, regardless of the size of the disk. **

So it will write 4 times more **primary** extents into the 225280 MB disks than into the 56320 MB disks (It will not necessary be **the case for mirrored extents** as you'll see later on into this post).

So, **asmcmd** reports **593798/2 MB** of free space: This space is "just" the sum of the free space of the disks that belongs to the diskgroup. So this is not fully "usable" due to the Imbalanced Diskgroup and the rule explained above.

My **utility** reports **91824** MB of free/usable space (less than 90 GB).

<span style="text-decoration:underline;">Let's verify how much space is "usable":</span>

    SQL> alter tablespace BDT add datafile '+BDT' size 30g;

    Tablespace altered.

    SQL> alter tablespace BDT add datafile '+BDT' size 30g;

    Tablespace altered.

    SQL> alter tablespace BDT add datafile '+BDT' size 30g;
    alter tablespace BDT add datafile '+BDT' size 30g
    *
    ERROR at line 1:
    ORA-01119: error in creating database file '+BDT'
    ORA-17502: ksfdcre:4 Failed to create file +BDT
    ORA-15041: diskgroup "BDT" space exhausted

    SQL> alter tablespace BDT add datafile '+BDT' size 29g;

    Tablespace altered.

So, it was not possible to add 90 GB while it has been possible to add 89 GB: My utility reported the right "free/usable" value.

<span style="text-decoration:underline;">Let's see the asm\_free\_usable\_imbalance.sql:</span>

```
SQL> !cat asm_free_usable_imbalance.sql  
select /* EXTERNAL REDUNDANCY */  
g.name,  
sum(d.TOTAL_MB) * min(d.FREE_MB / d.total_mb) /  
decode(g.type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) "USABLE_FREE_MB"  
from v$asm_disk d, v$asm_diskgroup g  
where d.group_number = g.group_number  
and g.type = 'EXTERN'  
group by g.name, g.type  
union  
select /* NON EXTERNAL REDUNDANCY WITH SYMMETRIC FG */  
g.name,  
sum(d.TOTAL_MB) * min(d.FREE_MB / d.total_mb) /  
decode(g.type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) "USABLE_FREE_MB"  
from v$asm_disk d, v$asm_diskgroup g  
where d.group_number = g.group_number  
and g.group_number not in /* KEEP SYMMETRIC*/  
(select distinct (group_number)  
from (select group_number,  
failgroup,  
TOTAL_MB,  
count_dsk,  
greatest(lag(count_dsk, 1, 0)  
over(partition by TOTAL_MB,  
group_number order by TOTAL_MB,  
FAILGROUP),  
lead(count_dsk, 1, 0)  
over(partition by TOTAL_MB,  
group_number order by TOTAL_MB,  
FAILGROUP)) as max_lag_lead,  
count(distinct(failgroup)) over(partition by group_number, TOTAL_MB) as nb_fg_per_size,  
count_fg  
from (select group_number,  
failgroup,  
TOTAL_MB,  
count(*) over(partition by group_number, failgroup, TOTAL_MB) as count_dsk,  
count(distinct(failgroup)) over(partition by group_number) as count_fg  
from v$asm_disk))  
where count_dsk &lt;> max_lag_lead  
or nb_fg_per_size &lt;> count_fg)  
and g.type &lt;> 'EXTERNAL'  
group by g.name, g.type  
union  
select /* NON EXTERNAL REDUNDANCY WITH NON SYMMETRIC FG  
AND DOES EXIST AT LEAST ONE DISK WITH PARTNERS OF DIFFERENT SIZE*/  
name,  
min(free) / decode(type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) "USABLE_FREE_MB"  
from (select name,  
disk_number,  
free_mb / (factor / sum(factor) over(partition by name)) as free,  
type  
from (select name,  
disk_number,  
avg(free_mb) as free_mb,  
avg(total_mb) as total_mb,  
sum(factor_disk + factor_partner) as factor,  
type  
from (SELECT g.name,  
g.type,  
d.group_number as group_number,  
d.disk_number disk_number,  
d.total_mb as total_mb,  
d.free_mb as free_mb,  
p.number_kfdpartner "Partner disk#",  
f.factor as factor_disk,  
fp.factor as factor_partner  
FROM x$kfdpartner p,  
v$asm_disk d,  
v$asm_diskgroup g,  
(select disk_number,  
group_number,  
TOTAL_MB / min(total_mb) over(partition by group_number) as factor  
from v$asm_disk  
where state = 'NORMAL'  
and mount_status = 'CACHED') f,  
(select disk_number,  
group_number,  
TOTAL_MB / min(total_mb) over(partition by group_number) as factor  
from v$asm_disk  
where state = 'NORMAL'  
and mount_status = 'CACHED') fp  
WHERE p.disk = d.disk_number  
and p.grp = d.group_number  
and f.disk_number = d.disk_number  
and f.group_number = d.group_number  
and fp.disk_number = p.number_kfdpartner  
and fp.group_number = p.grp  
and d.group_number = g.group_number  
and g.type &lt;> 'EXTERN'  
and g.group_number in /* KEEP NON SYMMETRIC */  
(select distinct (group_number)  
from (select group_number,  
failgroup,  
TOTAL_MB,  
count_dsk,  
greatest(lag(count_dsk, 1, 0)  
over(partition by  
TOTAL_MB,  
group_number order by  
TOTAL_MB,  
FAILGROUP),  
lead(count_dsk, 1, 0)  
over(partition by  
TOTAL_MB,  
group_number order by  
TOTAL_MB,  
FAILGROUP)) as max_lag_lead,  
count(distinct(failgroup)) over(partition by group_number, TOTAL_MB) as nb_fg_per_size,  
count_fg  
from (select group_number,  
failgroup,  
TOTAL_MB,  
count(*) over(partition by group_number, failgroup, TOTAL_MB) as count_dsk,  
count(distinct(failgroup)) over(partition by group_number) as count_fg  
from v$asm_disk))  
where count_dsk &lt;> max_lag_lead  
or nb_fg_per_size &lt;> count_fg)  
and d.group_number not in /* KEEP DG THAT DOES NOT CONTAIN AT LEAST ONE DISK HAVING PARTNERS OF DIFFERENT SIZE*/  
(select distinct (group_number)  
from (select d.group_number as group_number,  
d.disk_number disk_number,  
p.number_kfdpartner "Partner disk#",  
f.factor as factor_disk,  
fp.factor as factor_partner,  
greatest(lag(fp.factor, 1, 0)  
over(partition by  
d.group_number,  
d.disk_number order by  
d.group_number,  
d.disk_number),  
lead(fp.factor, 1, 0)  
over(partition by  
d.group_number,  
d.disk_number order by  
d.group_number,  
d.disk_number)) as max_lag_lead,  
count(p.number_kfdpartner) over(partition by d.group_number, d.disk_number) as nb_partner  
FROM x$kfdpartner p,  
v$asm_disk d,  
v$asm_diskgroup g,  
(select disk_number,  
group_number,  
TOTAL_MB / min(total_mb) over(partition by group_number) as factor  
from v$asm_disk  
where state = 'NORMAL'  
and mount_status = 'CACHED') f,  
(select disk_number,  
group_number,  
TOTAL_MB / min(total_mb) over(partition by group_number) as factor  
from v$asm_disk  
where state = 'NORMAL'  
and mount_status = 'CACHED') fp  
WHERE p.disk = d.disk_number  
and p.grp = d.group_number  
and f.disk_number = d.disk_number  
and f.group_number = d.group_number  
and fp.disk_number =  
p.number_kfdpartner  
and fp.group_number = p.grp  
and d.group_number = g.group_number  
and g.type &lt;> 'EXTERN')  
where factor_partner &lt;> max_lag_lead  
and nb_partner > 1))  
group by name, disk_number, type))  
group by name, type;  
```

<span style="text-decoration:underline;">Don't be afraid ;-): The SQL is composed of 3 cases:</span>

1.  **Diskgroup with external redundancy**: In that case the calculation is based on the smallest free space for a disk within the diskgroup.
2.  **Diskgoup with "non external" redundancy and symmetric failgroups** (Same number of disks grouped by size across failgroups): In that case the calculation is based on the smallest free space for a disk within the diskgroup.
3.  **Diskgoup with "non external" redundancy and non symmetric failgroups: **In that case the calculation is based on a weighting factor depending of the size of the disks and their partners.

<span style="text-decoration:underline;">**<span style="color:#ff0000;text-decoration:underline;">Warning</span>**:</span> The SQL does not <span style="color:#008000;">**yet**</span> cover the case (it will not return any result) of "non external" redundancy with non symmetric failgroups and if **it exists at least one disk that has partners of different sizes.**

This is due to the following remark.

<span style="text-decoration:underline;">**Remark:**</span>

I said that the ASM allocator ensures balanced disk utilization for **primary extents and not necessary for mirrored exents.**

<span style="text-decoration:underline;">Proof:</span>

Create an external redundancy diskgroup with 4 failgroups of disks not of the same sizes.

    SQL> !kfod
    --------------------------------------------------------------------------------
     Disk          Size Path                                     User     Group   
    ================================================================================
       1:     225280 Mb /dev/san/JMO7C43D06                      oracle   dba  
       2:      56320 Mb /dev/san/JMO7C52D06                      oracle   dba  
       3:      56320 Mb /dev/san/JMO7C61D06                      oracle   dba  
       4:     225280 Mb /dev/san/WIN7C43D06                      oracle   dba  
       5:      56320 Mb /dev/san/WIN7C52D06                      oracle   dba  
       6:      56320 Mb /dev/san/WIN7C61D06                      oracle   dba  
    --------------------------------------------------------------------------------

    SQL> create diskgroup BDT normal redundancy failgroup F1 disk '/dev/san/JMO7C43D06' failgroup F2 disk '/dev/san/JMO7C52D06' failgroup F3 disk '/dev/san/JMO7C61D06' failgroup F4 disk '/dev/san/WIN7C43D06';

    Diskgroup created.

Now create a 30 GB tablespace into the database:

    SQL> create tablespace BDT datafile '+BDT' size 30g;

    Tablespace created.

And check the primary and mirrored extents distribution within the disks into ASM:

    SQL> select d.group_number,d.disk_number,d.total_mb,d.free_mb,d.failgroup,decode(LXN_KFFXP,0,'P',1,'M','MM') ,count(*)
    from  X$KFFXP, v$asm_disk d
    where 
    GROUP_KFFXP = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT')
    and d.group_number=GROUP_KFFXP
    and d.disk_number=DISK_KFFXP
    group by d.group_number,d.disk_number,d.total_mb,d.free_mb,d.failgroup,LXN_KFFXP
    order by d.failgroup,d.total_mb; 

    GROUP_NUMBER DISK_NUMBER   TOTAL_MB    FREE_MB FAILGROUP                      DE   COUNT(*)
    ------------ ----------- ---------- ---------- ------------------------------ -- ----------
               8           0     225280     202020 F1                             P       12310
               8           0     225280     202020 F1                             M       10945
               8           0     225280     202020 F1                             MM          2
               8           1      56320      48778 F2                             P        3076
               8           1      56320      48778 F2                             M        4442
               8           1      56320      48778 F2                             MM         22
               8           2      56320      48779 F3                             P        3077
               8           2      56320      48779 F3                             M        4443
               8           2      56320      48779 F3                             MM         19
               8           3     225280     202018 F4                             P       12309
               8           3     225280     202018 F4                             M       10942
               8           3     225280     202018 F4                             MM          8

P stands for primary extents and M for mirrored "2 way" extents.

So as you can see ASM allocator writes 4 times more **primary extents **into the 225280 MB disks than into the 56320 MB disks (12310/3076) while **this is not the case for the mirrored extents** (about 10945/4442= 2.5 times).

This is the main reason why a case is still not covered into asm\_free\_space\_imbalanced.sql. If you have any clues about the mirrored extents distribution please let me know ;-)

<span style="text-decoration:underline;">**Conclusion:**</span>

-   The sql provides a way to calculate the free space in case of **Imbalanced diskgroup.**
-   The calculation is based on the smallest free space for a disk within the diskgroup and a weighting factor that depends of the disks size and their partners.
-   The sql provides also right results for **Balanced diskgroup ;-)**
-   The sql does not <span style="color:#008000;">**yet** </span>cover the case of "non external" redundancy and if it exists at least one disk that has partners of different sizes.

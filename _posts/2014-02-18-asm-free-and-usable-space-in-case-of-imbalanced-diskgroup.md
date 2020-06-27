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
<p>In case of "normal" situation (I mean no Imbalanced Disks) you should be able to easily know how much space is available into the ASM Diskgroup.</p>
<p>Harald van Breederode already explained it in detail into <a href="http://prutser.wordpress.com/2013/01/03/demystifying-asm-required_mirror_free_mb-and-usable_file_mb/" target="_blank">this blog post</a>.</p>
<p><span style="text-decoration:underline;">But now suppose the Diskgroup has <strong>imbalanced</strong> Disks: It can occurs for several reasons:</span></p>
<ul>
<li>A rebalance operation has been aborted/halted.</li>
<li>A rebalance operation is waiting.</li>
<li>Disks are not of the same size.</li>
</ul>
<p><span style="text-decoration:underline;">Let's simulate an Imbalanced situation and compute the usable free space:</span></p>
<p>First I created a "balanced" diskgroup with normal redundancy.</p>
<pre style="padding-left:30px;">SQL&gt; !kfod
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

SQL&gt; create diskgroup BDT normal redundancy failgroup F1 disk '/dev/san/JMO7C52D06' failgroup F2 disk '/dev/san/WIN7C52D06';

Diskgroup created.</pre>
<p>Let's check its free space with <strong>asmcmd</strong> and with <strong>asm_free_usable_imbalance.sql</strong> (I'll share the code later into this post) built to report the free/usable space of Imbalanced Diskgroup:</p>
<pre style="padding-left:30px;">SQL&gt; !asmcmd lsdg BDT
State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  NORMAL  N         512   4096  1048576    112640   112538                0           56269              0             N  BDT/

SQL&gt; @asm_free_usable_imbalance.sql

NAME                              FREE_MB
------------------------------ ----------
BDT                                 56269</pre>
<p>Well, we can see that from asmcmd (112538 / 2) 56269 Mb is free/usable and that the asm_free_usable_imbalance.sql reports the same value.</p>
<p>So that the asm_free_usable_imbalance.sql script <strong>works at least </strong>with "Balanced" diskgroup ;-).</p>
<p><span style="text-decoration:underline;">Now let's produce an Imbalanced situation that way:</span></p>
<ul>
<li>Add datafiles into the Diskgroup.</li>
<li>Add Disks into the failgroups without trigerring a rebalance operation (asm_power_limit = 0).</li>
</ul>
<pre style="padding-left:30px;">SQL&gt; create tablespace BDT datafile '+BDT' size 30g;

Tablespace created.

SQL&gt; alter tablespace BDT add datafile '+BDT' size 10g;

Tablespace altered.

SQL&gt; alter tablespace BDT add datafile '+BDT' size 20g;
alter tablespace BDT add datafile '+BDT' size 20g
*
ERROR at line 1:
ORA-01119: error in creating database file '+BDT'
ORA-17502: ksfdcre:4 Failed to create file +BDT
ORA-15041: diskgroup "BDT" space exhausted</pre>
<p>So, we can't add a 20 GB datafile anymore (sounds obvious as the free space was about 55 GB and we already added 40 GB into the diskgroup).</p>
<p>Now add disks of different size without trigerring a rebalance operation:</p>
<pre style="padding-left:30px;">SQL&gt; !kfod
--------------------------------------------------------------------------------
 Disk          Size Path                                     User     Group   
================================================================================
   1:     225280 Mb /dev/san/JMO7C43D06                      oracle   dba  
   2:      56320 Mb /dev/san/JMO7C61D06                      oracle   dba  
   3:     225280 Mb /dev/san/WIN7C43D06                      oracle   dba  
   4:      56320 Mb /dev/san/WIN7C61D06                      oracle   dba  
--------------------------------------------------------------------------------

SQL&gt; alter diskgroup BDT add failgroup F1 disk '/dev/san/JMO7C43D06','/dev/san/JMO7C61D06' failgroup F2 disk '/dev/san/WIN7C43D06','/dev/san/WIN7C61D06';

Diskgroup altered.</pre>
<p>Verify that the rebalance is waiting:</p>
<pre style="padding-left:30px;">SQL&gt; select * from v$asm_operation where group_number = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT');

GROUP_NUMBER OPERA STAT      POWER     ACTUAL      SOFAR   EST_WORK   EST_RATE EST_MINUTES ERROR_CODE
------------ ----- ---- ---------- ---------- ---------- ---------- ---------- ----------- --------------------------------------------
           8 REBAL WAIT          0</pre>
<p>and the disks usage:</p>
<pre style="padding-left:30px;">SQL&gt; select failgroup,total_mb,FREE_MB from v$asm_disk where group_number = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT');

FAILGROUP                        TOTAL_MB    FREE_MB
------------------------------ ---------- ----------
F1                                  56320      56318
F1                                 225280     225277
F1                                  56320      15304
F2                                 225280     225277
F2                                  56320      15304
F2                                  56320      56318</pre>
<p><span style="text-decoration:underline;">Here we are: <strong>How much usable/free space is "really" available into this "Imbalanced" diskgroup ?</strong></span></p>
<p>Let's see what asmcmd and asm_free_usable_imbalance.sql report:</p>
<pre style="padding-left:30px;">SQL&gt; !asmcmd lsdg BDT
State    Type    Rebal  Sector  Block       AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  NORMAL  Y         512   4096  1048576    675840   593798           225280          184259              0             N  BDT/

SQL&gt; @asm_free_usable_imbalance.sql

NAME                              FREE_MB
------------------------------ ----------
BDT                                 91824</pre>
<p>Now we have to remember that for <strong>primary</strong> extents, ASM will allocate new extents in such way as to distribute each <strong>file equally and evenly across all disks and to fill all disks evenly</strong>. Thus every disk is maintained at the <strong>same percentage full, regardless of the size of the disk. </strong></p>
<p>So it will write 4 times more <strong>primary</strong> extents into the 225280 MB disks than into the 56320 MB disks (It will not necessary be <strong>the case for mirrored extents</strong> as you'll see later on into this post).</p>
<p>So, <strong>asmcmd</strong> reports <strong>593798/2 MB</strong> of free space: This space is "just" the sum of the free space of the disks that belongs to the diskgroup. So this is not fully "usable" due to the Imbalanced Diskgroup and the rule explained above.</p>
<p>My <strong>utility</strong> reports <strong>91824</strong> MB of free/usable space (less than 90 GB).</p>
<p><span style="text-decoration:underline;">Let's verify how much space is "usable":</span></p>
<pre style="padding-left:30px;">SQL&gt; alter tablespace BDT add datafile '+BDT' size 30g;

Tablespace altered.

SQL&gt; alter tablespace BDT add datafile '+BDT' size 30g;

Tablespace altered.

SQL&gt; alter tablespace BDT add datafile '+BDT' size 30g;
alter tablespace BDT add datafile '+BDT' size 30g
*
ERROR at line 1:
ORA-01119: error in creating database file '+BDT'
ORA-17502: ksfdcre:4 Failed to create file +BDT
ORA-15041: diskgroup "BDT" space exhausted

SQL&gt; alter tablespace BDT add datafile '+BDT' size 29g;

Tablespace altered.</pre>
<p>So, it was not possible to add 90 GB while it has been possible to add 89 GB: My utility reported the right "free/usable" value.</p>
<p><span style="text-decoration:underline;">Let's see the asm_free_usable_imbalance.sql:</span></p>
<p>[code language="sql"]<br />
SQL&gt; !cat asm_free_usable_imbalance.sql<br />
select /* EXTERNAL REDUNDANCY */<br />
g.name,<br />
sum(d.TOTAL_MB) * min(d.FREE_MB / d.total_mb) /<br />
decode(g.type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) &quot;USABLE_FREE_MB&quot;<br />
from v$asm_disk d, v$asm_diskgroup g<br />
where d.group_number = g.group_number<br />
and g.type = 'EXTERN'<br />
group by g.name, g.type<br />
union<br />
select /* NON EXTERNAL REDUNDANCY WITH SYMMETRIC FG */<br />
g.name,<br />
sum(d.TOTAL_MB) * min(d.FREE_MB / d.total_mb) /<br />
decode(g.type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) &quot;USABLE_FREE_MB&quot;<br />
from v$asm_disk d, v$asm_diskgroup g<br />
where d.group_number = g.group_number<br />
and g.group_number not in /* KEEP SYMMETRIC*/<br />
(select distinct (group_number)<br />
from (select group_number,<br />
failgroup,<br />
TOTAL_MB,<br />
count_dsk,<br />
greatest(lag(count_dsk, 1, 0)<br />
over(partition by TOTAL_MB,<br />
group_number order by TOTAL_MB,<br />
FAILGROUP),<br />
lead(count_dsk, 1, 0)<br />
over(partition by TOTAL_MB,<br />
group_number order by TOTAL_MB,<br />
FAILGROUP)) as max_lag_lead,<br />
count(distinct(failgroup)) over(partition by group_number, TOTAL_MB) as nb_fg_per_size,<br />
count_fg<br />
from (select group_number,<br />
failgroup,<br />
TOTAL_MB,<br />
count(*) over(partition by group_number, failgroup, TOTAL_MB) as count_dsk,<br />
count(distinct(failgroup)) over(partition by group_number) as count_fg<br />
from v$asm_disk))<br />
where count_dsk &lt;&gt; max_lag_lead<br />
or nb_fg_per_size &lt;&gt; count_fg)<br />
and g.type &lt;&gt; 'EXTERNAL'<br />
group by g.name, g.type<br />
union<br />
select /* NON EXTERNAL REDUNDANCY WITH NON SYMMETRIC FG<br />
AND DOES EXIST AT LEAST ONE DISK WITH PARTNERS OF DIFFERENT SIZE*/<br />
name,<br />
min(free) / decode(type, 'EXTERN', 1, 'NORMAL', 2, 'HIGH', 3, 1) &quot;USABLE_FREE_MB&quot;<br />
from (select name,<br />
disk_number,<br />
free_mb / (factor / sum(factor) over(partition by name)) as free,<br />
type<br />
from (select name,<br />
disk_number,<br />
avg(free_mb) as free_mb,<br />
avg(total_mb) as total_mb,<br />
sum(factor_disk + factor_partner) as factor,<br />
type<br />
from (SELECT g.name,<br />
g.type,<br />
d.group_number as group_number,<br />
d.disk_number disk_number,<br />
d.total_mb as total_mb,<br />
d.free_mb as free_mb,<br />
p.number_kfdpartner &quot;Partner disk#&quot;,<br />
f.factor as factor_disk,<br />
fp.factor as factor_partner<br />
FROM x$kfdpartner p,<br />
v$asm_disk d,<br />
v$asm_diskgroup g,<br />
(select disk_number,<br />
group_number,<br />
TOTAL_MB / min(total_mb) over(partition by group_number) as factor<br />
from v$asm_disk<br />
where state = 'NORMAL'<br />
and mount_status = 'CACHED') f,<br />
(select disk_number,<br />
group_number,<br />
TOTAL_MB / min(total_mb) over(partition by group_number) as factor<br />
from v$asm_disk<br />
where state = 'NORMAL'<br />
and mount_status = 'CACHED') fp<br />
WHERE p.disk = d.disk_number<br />
and p.grp = d.group_number<br />
and f.disk_number = d.disk_number<br />
and f.group_number = d.group_number<br />
and fp.disk_number = p.number_kfdpartner<br />
and fp.group_number = p.grp<br />
and d.group_number = g.group_number<br />
and g.type &lt;&gt; 'EXTERN'<br />
and g.group_number in /* KEEP NON SYMMETRIC */<br />
(select distinct (group_number)<br />
from (select group_number,<br />
failgroup,<br />
TOTAL_MB,<br />
count_dsk,<br />
greatest(lag(count_dsk, 1, 0)<br />
over(partition by<br />
TOTAL_MB,<br />
group_number order by<br />
TOTAL_MB,<br />
FAILGROUP),<br />
lead(count_dsk, 1, 0)<br />
over(partition by<br />
TOTAL_MB,<br />
group_number order by<br />
TOTAL_MB,<br />
FAILGROUP)) as max_lag_lead,<br />
count(distinct(failgroup)) over(partition by group_number, TOTAL_MB) as nb_fg_per_size,<br />
count_fg<br />
from (select group_number,<br />
failgroup,<br />
TOTAL_MB,<br />
count(*) over(partition by group_number, failgroup, TOTAL_MB) as count_dsk,<br />
count(distinct(failgroup)) over(partition by group_number) as count_fg<br />
from v$asm_disk))<br />
where count_dsk &lt;&gt; max_lag_lead<br />
or nb_fg_per_size &lt;&gt; count_fg)<br />
and d.group_number not in /* KEEP DG THAT DOES NOT CONTAIN AT LEAST ONE DISK HAVING PARTNERS OF DIFFERENT SIZE*/<br />
(select distinct (group_number)<br />
from (select d.group_number as group_number,<br />
d.disk_number disk_number,<br />
p.number_kfdpartner &quot;Partner disk#&quot;,<br />
f.factor as factor_disk,<br />
fp.factor as factor_partner,<br />
greatest(lag(fp.factor, 1, 0)<br />
over(partition by<br />
d.group_number,<br />
d.disk_number order by<br />
d.group_number,<br />
d.disk_number),<br />
lead(fp.factor, 1, 0)<br />
over(partition by<br />
d.group_number,<br />
d.disk_number order by<br />
d.group_number,<br />
d.disk_number)) as max_lag_lead,<br />
count(p.number_kfdpartner) over(partition by d.group_number, d.disk_number) as nb_partner<br />
FROM x$kfdpartner p,<br />
v$asm_disk d,<br />
v$asm_diskgroup g,<br />
(select disk_number,<br />
group_number,<br />
TOTAL_MB / min(total_mb) over(partition by group_number) as factor<br />
from v$asm_disk<br />
where state = 'NORMAL'<br />
and mount_status = 'CACHED') f,<br />
(select disk_number,<br />
group_number,<br />
TOTAL_MB / min(total_mb) over(partition by group_number) as factor<br />
from v$asm_disk<br />
where state = 'NORMAL'<br />
and mount_status = 'CACHED') fp<br />
WHERE p.disk = d.disk_number<br />
and p.grp = d.group_number<br />
and f.disk_number = d.disk_number<br />
and f.group_number = d.group_number<br />
and fp.disk_number =<br />
p.number_kfdpartner<br />
and fp.group_number = p.grp<br />
and d.group_number = g.group_number<br />
and g.type &lt;&gt; 'EXTERN')<br />
where factor_partner &lt;&gt; max_lag_lead<br />
and nb_partner &gt; 1))<br />
group by name, disk_number, type))<br />
group by name, type;<br />
[/code]</p>
<p><span style="text-decoration:underline;">Don't be afraid ;-): The SQL is composed of 3 cases:</span></p>
<ol>
<li><strong>Diskgroup with external redundancy</strong>: In that case the calculation is based on the smallest free space for a disk within the diskgroup.</li>
<li><strong>Diskgoup with "non external" redundancy and symmetric failgroups</strong> (Same number of disks grouped by size across failgroups): In that case the calculation is based on the smallest free space for a disk within the diskgroup.</li>
<li><strong>Diskgoup with "non external" redundancy and non symmetric failgroups: </strong>In that case the calculation is based on a weighting factor depending of the size of the disks and their partners.</li>
</ol>
<p><span style="text-decoration:underline;"><strong><span style="color:#ff0000;text-decoration:underline;">Warning</span></strong>:</span> The SQL does not <span style="color:#008000;"><strong>yet</strong></span> cover the case (it will not return any result) of "non external" redundancy with non symmetric failgroups and if <strong>it exists at least one disk that has partners of different sizes.</strong></p>
<p>This is due to the following remark.</p>
<p><span style="text-decoration:underline;"><strong>Remark:</strong></span></p>
<p>I said that the ASM allocator ensures balanced disk utilization for <strong>primary extents and not necessary for mirrored exents.</strong></p>
<p><span style="text-decoration:underline;">Proof:</span></p>
<p>Create an external redundancy diskgroup with 4 failgroups of disks not of the same sizes.</p>
<pre>SQL&gt; !kfod
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

SQL&gt; create diskgroup BDT normal redundancy failgroup F1 disk '/dev/san/JMO7C43D06' failgroup F2 disk '/dev/san/JMO7C52D06' failgroup F3 disk '/dev/san/JMO7C61D06' failgroup F4 disk '/dev/san/WIN7C43D06';

Diskgroup created.</pre>
<p>Now create a 30 GB tablespace into the database:</p>
<pre style="padding-left:30px;">SQL&gt; create tablespace BDT datafile '+BDT' size 30g;

Tablespace created.</pre>
<p>And check the primary and mirrored extents distribution within the disks into ASM:</p>
<pre style="padding-left:30px;">SQL&gt; select d.group_number,d.disk_number,d.total_mb,d.free_mb,d.failgroup,decode(LXN_KFFXP,0,'P',1,'M','MM') ,count(*)
from  X$KFFXP, v$asm_disk d
where 
GROUP_KFFXP = (select GROUP_NUMBER from v$asm_diskgroup where NAME='BDT')
and d.group_number=GROUP_KFFXP
and d.disk_number=DISK_KFFXP
group by d.group_number,d.disk_number,d.total_mb,d.free_mb,d.failgroup,LXN_KFFXP
order by d.failgroup,d.total_mb; 

GROUP_NUMBER DISK_NUMBER   TOTAL_MB    FREE_MB FAILGROUP                      DE   COUNT(*)
------------ ----------- ---------- ---------- ------------------------------ -- ----------
8 0 225280 202020 F1 P 12310 8 0 225280 202020 F1 M 10945 8 0 225280 202020 F1 MM 2 8 1 56320 48778 F2 P 3076 8 1 56320 48778 F2 M 4442 8 1 56320 48778 F2 MM 22 8 2 56320 48779 F3 P 3077 8 2 56320 48779 F3 M 4443 8 2 56320 48779 F3 MM 19 8 3 225280 202018 F4 P 12309 8 3 225280 202018 F4 M 10942 8 3 225280 202018 F4 MM 8

P stands for primary extents and M for mirrored "2 way" extents.

So as you can see ASM allocator writes 4 times more **primary extents&nbsp;** into the&nbsp;225280 MB disks than into the&nbsp;56320 MB disks (12310/3076)&nbsp;while **this is not the case for the mirrored extents** (about&nbsp;10945/4442= 2.5 times).

This is the main reason why a case is still not covered into&nbsp;asm\_free\_space\_imbalanced.sql. If you have any clues about the mirrored extents distribution please let me know ;-)

**Conclusion:**

- The sql provides a way to calculate the free space in case of **Imbalanced diskgroup.**
- The calculation is based on the smallest free space for a disk within the diskgroup and a&nbsp;weighting factor that depends of the disks size and their partners.
- The sql provides also right results for **Balanced diskgroup ;-)**
- The sql does not **yet** cover the case of&nbsp;"non external" redundancy and if it exists at least one disk that has partners of different sizes.

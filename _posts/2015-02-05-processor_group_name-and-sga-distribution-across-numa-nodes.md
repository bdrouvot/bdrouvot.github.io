---
layout: post
title: processor_group_name and SGA distribution across NUMA nodes
date: 2015-02-05 18:57:32.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/PRSVJrhixki
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/a2kzOi90sa
  _wpas_done_2225791: '1'
  publicize_linkedin_url: ''
  _wpas_done_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/02/05/processor_group_name-and-sga-distribution-across-numa-nodes/"
---

<span style="text-decoration:underline;">**Introduction**</span>

On a NUMA enabled system, a 12c database Instance allocates the SGA **evenly** across all the NUMA nodes by default (unless you change the "*\_enable\_NUMA\_interleave*” hidden parameter to *FALSE*). You can find more details about this behaviour in [Yves's post](https://ycolin.wordpress.com/2015/01/18/numa-interleave-memory).

Fine, but what if I am setting the *processor\_group\_name* parameter (to instruct the database instance to run itself within a specified operating system processor group) **to a cgroup that contains more than one NUMA node**?

I may need to link the *processor\_group\_name* parameter to more than one NUMA node because:

-   the database needs more memory that one NUMA node could offer.
-   the database needs more cpus that one NUMA node could offer.

In that case, **is the SGA still evenly allocated across the NUMA nodes defined in the cgroup?**

<span style="text-decoration:underline;">**Time to test**</span>

My NUMA configuration is the following:

    > lscpu | grep NUMA
    NUMA node(s):          8
    NUMA node0 CPU(s):     1-10,41-50
    NUMA node1 CPU(s):     11-20,51-60
    NUMA node2 CPU(s):     21-30,61-70
    NUMA node3 CPU(s):     31-40,71-80
    NUMA node4 CPU(s):     0,81-89,120-129
    NUMA node5 CPU(s):     90-99,130-139
    NUMA node6 CPU(s):     100-109,140-149
    NUMA node7 CPU(s):     110-119,150-159

with this memory allocation (no Instances up):

     > sh ./numa_memory.sh
                    MemTotal         MemFree         MemUsed           Shmem      HugePages_Total       HugePages_Free       HugePages_Surp
    Node_0      136052736_kB     49000636_kB     87052100_kB        77728_kB                39300                39044                    0
    Node_1      136052736_kB     50503228_kB     85549508_kB        77764_kB                39300                39044                    0
    Node_2      136052736_kB     18163112_kB    117889624_kB        78236_kB                39300                39045                    0
    Node_3      136052736_kB     47859560_kB     88193176_kB        77744_kB                39300                39045                    0
    Node_4      136024568_kB     43286800_kB     92737768_kB        77780_kB                39300                39045                    0
    Node_5      136052736_kB     50348004_kB     85704732_kB        77792_kB                39300                39045                    0
    Node_6      136052736_kB     31591648_kB    104461088_kB        77976_kB                39299                39044                    0
    Node_7      136052736_kB     48524064_kB     87528672_kB        77780_kB                39299                39044                    0

A cgroup (named oracle) has been created with those properties:

       group oracle {
       perm {
         task {
           uid = oracle;
           gid = dc_dba;
         }
         admin {
           uid = oracle;
           gid = dc_dba;
         }
       }
       cpuset {
         cpuset.mems=2,3;
         cpuset.cpus="21-30,61-70,31-40,71-80";
       }

The 12.1.0.2 database:

-   has been started using this cgroup thanks to the *processor\_group\_name* parameter (so that the database is linked to **2 NUMA nodes:  Nodes 2 and 3)**.
-   uses *use\_large\_pages* set to ***ONLY***.
-   uses "\_enable\_NUMA\_interleave" set to its **default** value: *TRUE*.
-   *uses "\_enable\_NUMA\_support"* set to its **default** value: *FALSE.  
    *

<span style="text-decoration:underline;">**First test: The Instance has been started with a SGA that could fully fit into one NUMA node.**</span>

Let's check the memory distribution:

    > sh ./numa_memory.sh
                    MemTotal         MemFree         MemUsed           Shmem      HugePages_Total       HugePages_Free       HugePages_Surp
    Node_0      136052736_kB     48999132_kB     87053604_kB        77720_kB                39300                39044                    0
    Node_1      136052736_kB     50504752_kB     85547984_kB        77748_kB                39300                39044                    0
    Node_2      136052736_kB     17903752_kB    118148984_kB        78224_kB                39300                23427                    0
    Node_3      136052736_kB     47511596_kB     88541140_kB        77736_kB                39300                39045                    0
    Node_4      136024568_kB     43282316_kB     92742252_kB        77796_kB                39300                39045                    0
    Node_5      136052736_kB     50345492_kB     85707244_kB        77804_kB                39300                39045                    0
    Node_6      136052736_kB     31581004_kB    104471732_kB        77988_kB                39299                39044                    0
    Node_7      136052736_kB     48516964_kB     87535772_kB        77784_kB                39299                39044                    0

As you can see, 39300-23427 large pages have been allocated from **node 2 only**. This is the amount of large pages needed for the SGA. So we can conclude that **the memory is not evenly distributed across NUMA nodes 2 et 3**.

<span style="text-decoration:underline;">**Second test: The Instance has been started with a SGA that can not fully fit into one NUMA node.**</span>

Let's check the memory distribution:

    > sh ./numa_memory.sh
                    MemTotal         MemFree         MemUsed           Shmem      HugePages_Total       HugePages_Free       HugePages_Surp
    Node_0      136052736_kB     51332440_kB     84720296_kB        77832_kB                39300                39044                    0
    Node_1      136052736_kB     52532564_kB     83520172_kB        77788_kB                39300                39044                    0
    Node_2      136052736_kB     51669192_kB     84383544_kB        77892_kB                39300                    0                    0
    Node_3      136052736_kB     52089448_kB     83963288_kB        77860_kB                39300                25864                    0
    Node_4      136024568_kB     51992248_kB     84032320_kB        77876_kB                39300                39045                    0
    Node_5      136052736_kB     52571468_kB     83481268_kB        77856_kB                39300                39045                    0
    Node_6      136052736_kB     52131912_kB     83920824_kB        77844_kB                39299                39044                    0
    Node_7      136052736_kB     52268200_kB     83784536_kB        77832_kB                39299                39044                    0

As you can see, the large pages have been allocated from nodes 2 and 3 **but not evenly (as there is no more free large pages on node 2 while there is still about 25000 free on node 3).**

<span style="text-decoration:underline;">**Remarks**</span>

-   I observed the same with or without ASMM.

<!-- -->

-   I also tried with [*cpuset.memory\_spread\_page* and *cpuset.memory\_spread\_slab*](https://access.redhat.com/documentation/fr-FR/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpuset.html) set to 1: Same results.

<!-- -->

-   With **no large pages** (*use\_large\_pages*=*FALSE*) and no ASMM, I can observe this memory distribution:

<!-- -->

    > sh ./numa_memory.sh
                    MemTotal         MemFree         MemUsed           Shmem      HugePages_Total       HugePages_Free       HugePages_Surp
    Node_0      136052736_kB     48994024_kB     87058712_kB        77712_kB                39300                39044                    0
    Node_1      136052736_kB     50497216_kB     85555520_kB        77752_kB                39300                39044                    0
    Node_2      136052736_kB      9225964_kB    126826772_kB      8727192_kB                39300                39045                    0
    Node_3      136052736_kB     38986380_kB     97066356_kB      8710796_kB                39300                39045                    0
    Node_4      136024568_kB     43279124_kB     92745444_kB        77792_kB                39300                39045                    0
    Node_5      136052736_kB     50341284_kB     85711452_kB        77796_kB                39300                39045                    0
    Node_6      136052736_kB     31570200_kB    104482536_kB        78000_kB                39299                39044                    0
    Node_7      136052736_kB     48505716_kB     87547020_kB        77776_kB                39299                39044                    0

As you can see the SGA **has been evenly distributed** across NUMA nodes 2 et 3 (see the *Shmem* column). I did not do more tests (means with ASMM or with AMM or...) with **non** large pages as not using large pages is not an option for me.

-   With large page and **no** *processor\_group\_name* set, the memory allocation looks like:

<!-- -->

    > sh ./numa_memory.sh
                    MemTotal         MemFree         MemUsed           Shmem      HugePages_Total       HugePages_Free       HugePages_Surp
    Node_0      136052736_kB     51234984_kB     84817752_kB        77824_kB                39300                37094                    0
    Node_1      136052736_kB     52417252_kB     83635484_kB        77772_kB                39300                37095                    0
    Node_2      136052736_kB     51178872_kB     84873864_kB        77904_kB                39300                37096                    0
    Node_3      136052736_kB     52263300_kB     83789436_kB        77872_kB                39300                37096                    0
    Node_4      136024568_kB     51870848_kB     84153720_kB        77864_kB                39300                37097                    0
    Node_5      136052736_kB     52304932_kB     83747804_kB        77852_kB                39300                37097                    0
    Node_6      136052736_kB     51837040_kB     84215696_kB        77840_kB                39299                37096                    0
    Node_7      136052736_kB     52213888_kB     83838848_kB        77860_kB                39299                37096                    0

As you can see the large pages have **been evenly allocated** from all the NUMA nodes (this is what has been told in the introduction).

-   If you like it, you can get the simple *num\_memory.sh* script from [here](https://github.com/bdrouvot/numa_scripts).

<span style="text-decoration:underline;">**Summary**</span>

1.  **With large pages** in place and ***processor\_group\_name* linked** to more than one NUMA node, then the **SGA is not evenly distributed** across the NUMA nodes defined in the cgroup.
2.  **With large pages** in place and ***processor\_group\_name* not set**,  then the **SGA is evenly distributed** across the NUMA nodes.

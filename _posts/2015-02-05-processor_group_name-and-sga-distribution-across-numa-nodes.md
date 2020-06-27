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
 **Introduction**

On a NUMA enabled system, a 12c database Instance allocates the SGA **evenly** across all the NUMA nodes by default (unless you change the "_\_enable\_NUMA\_interleave_” hidden parameter to _FALSE_). You can find more details about this behaviour in [Yves's post](https://ycolin.wordpress.com/2015/01/18/numa-interleave-memory).

Fine, but what if I am setting the _processor\_group\_name_ parameter (to instruct the database instance to run itself within a specified operating system processor group) **to a cgroup that contains more than one NUMA node**?

I may need to link the _processor\_group\_name_ parameter to more than one NUMA node because:

- the database needs more memory that one NUMA node could offer.
- the database needs more cpus that one NUMA node could offer.

In that case, **is the SGA still evenly allocated across the NUMA nodes defined in the cgroup?**

**Time to test**

My NUMA configuration is the following:

```
\> lscpu | grep NUMA NUMA node(s): 8 NUMA node0 CPU(s): 1-10,41-50 NUMA node1 CPU(s): 11-20,51-60 NUMA node2 CPU(s): 21-30,61-70 NUMA node3 CPU(s): 31-40,71-80 NUMA node4 CPU(s): 0,81-89,120-129 NUMA node5 CPU(s): 90-99,130-139 NUMA node6 CPU(s): 100-109,140-149 NUMA node7 CPU(s): 110-119,150-159
```

with this memory allocation (no Instances up):

```
\> sh ./numa\_memory.sh MemTotal MemFree MemUsed Shmem HugePages\_Total HugePages\_Free HugePages\_Surp Node\_0 136052736\_kB 49000636\_kB 87052100\_kB 77728\_kB 39300 39044 0 Node\_1 136052736\_kB 50503228\_kB 85549508\_kB 77764\_kB 39300 39044 0 Node\_2 136052736\_kB 18163112\_kB 117889624\_kB 78236\_kB 39300 39045 0 Node\_3 136052736\_kB 47859560\_kB 88193176\_kB 77744\_kB 39300 39045 0 Node\_4 136024568\_kB 43286800\_kB 92737768\_kB 77780\_kB 39300 39045 0 Node\_5 136052736\_kB 50348004\_kB 85704732\_kB 77792\_kB 39300 39045 0 Node\_6 136052736\_kB 31591648\_kB 104461088\_kB 77976\_kB 39299 39044 0 Node\_7 136052736\_kB 48524064\_kB 87528672\_kB 77780\_kB 39299 39044 0
```

A cgroup (named oracle) has been created with those properties:

```
group oracle { &nbsp;&nbsp; perm { &nbsp;&nbsp;&nbsp;&nbsp; task { &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; uid = oracle; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; gid = dc\_dba; &nbsp;&nbsp;&nbsp;&nbsp; } &nbsp;&nbsp;&nbsp;&nbsp; admin { &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; uid = oracle; &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; gid = dc\_dba; &nbsp;&nbsp;&nbsp;&nbsp; } &nbsp;&nbsp; } &nbsp;&nbsp; cpuset { &nbsp;&nbsp;&nbsp;&nbsp; **cpuset.mems=2,3** ; &nbsp;&nbsp;&nbsp;&nbsp; **cpuset.cpus="21-30,61-70,31-40,71-80"** ; &nbsp;&nbsp; }
```

The 12.1.0.2 database:

- has been started using this cgroup thanks to the _processor\_group\_name_ parameter (so that the database is linked to **2 NUMA nodes:&nbsp; Nodes 2 and 3)**.
- uses _use\_large\_pages_ set to **_ONLY_**.
- uses "\_enable\_NUMA\_interleave" set to its **default** value: _TRUE_.
- _uses "\_enable\_NUMA\_support"_ set to its **default** value: _FALSE._  

**First test: The Instance has been started with a SGA that could fully fit into one NUMA node.**

Let's check the memory distribution:

```
\> sh ./numa\_memory.sh MemTotal MemFree MemUsed Shmem HugePages\_Total HugePages\_Free HugePages\_Surp Node\_0 136052736\_kB 48999132\_kB 87053604\_kB 77720\_kB 39300 39044 0 Node\_1 136052736\_kB 50504752\_kB 85547984\_kB 77748\_kB 39300 39044 0 **Node\_2 136052736\_kB 17903752\_kB 118148984\_kB 78224\_kB 39300 23427** 0 Node\_3 136052736\_kB 47511596\_kB 88541140\_kB 77736\_kB 39300 39045 0 Node\_4 136024568\_kB 43282316\_kB 92742252\_kB 77796\_kB 39300 39045 0 Node\_5 136052736\_kB 50345492\_kB 85707244\_kB 77804\_kB 39300 39045 0 Node\_6 136052736\_kB 31581004\_kB 104471732\_kB 77988\_kB 39299 39044 0 Node\_7 136052736\_kB 48516964\_kB 87535772\_kB 77784\_kB 39299 39044 0
```

As you can see, 39300-23427 large pages have been allocated from **node 2 only**. This is the amount of large pages needed for the SGA. So we can conclude that **the memory is not evenly distributed across NUMA nodes 2 et 3**.

**Second test: The Instance has been started with a SGA that can not fully fit into one NUMA node.**

Let's check the memory distribution:

```
\> sh ./numa\_memory.sh MemTotal MemFree MemUsed Shmem HugePages\_Total HugePages\_Free HugePages\_Surp Node\_0 136052736\_kB 51332440\_kB 84720296\_kB 77832\_kB 39300 39044 0 Node\_1 136052736\_kB 52532564\_kB 83520172\_kB 77788\_kB 39300 39044 0 **Node\_2 136052736\_kB 51669192\_kB 84383544\_kB 77892\_kB 39300 0 0 Node\_3 136052736\_kB 52089448\_kB 83963288\_kB 77860\_kB 39300 25864 0** Node\_4 136024568\_kB 51992248\_kB 84032320\_kB 77876\_kB 39300 39045 0 Node\_5 136052736\_kB 52571468\_kB 83481268\_kB 77856\_kB 39300 39045 0 Node\_6 136052736\_kB 52131912\_kB 83920824\_kB 77844\_kB 39299 39044 0 Node\_7 136052736\_kB 52268200\_kB 83784536\_kB 77832\_kB 39299 39044 0
```

As you can see, the large pages have been allocated from nodes 2 and 3 **but not evenly (as there is no more free large pages on node 2 while there is still about 25000 free on node 3).**

**Remarks**

- I observed the same with or without ASMM.

- I also tried with [_cpuset.memory\_spread\_page_ and _cpuset.memory\_spread\_slab_](https://access.redhat.com/documentation/fr-FR/Red_Hat_Enterprise_Linux/6/html/Resource_Management_Guide/sec-cpuset.html) set to 1: Same results.

- With **no large pages** (_use\_large\_pages_=_FALSE_) and no ASMM, I can observe this memory distribution:

```
\> sh ./numa\_memory.sh MemTotal MemFree MemUsed Shmem HugePages\_Total HugePages\_Free HugePages\_Surp Node\_0 136052736\_kB 48994024\_kB 87058712\_kB 77712\_kB 39300 39044 0 Node\_1 136052736\_kB 50497216\_kB 85555520\_kB 77752\_kB 39300 39044 0 **Node\_2 136052736\_kB 9225964\_kB 126826772\_kB 8727192\_kB 39300 39045 0 Node\_3 136052736\_kB 38986380\_kB 97066356\_kB 8710796\_kB 39300 39045 0** Node\_4 136024568\_kB 43279124\_kB 92745444\_kB 77792\_kB 39300 39045 0 Node\_5 136052736\_kB 50341284\_kB 85711452\_kB 77796\_kB 39300 39045 0 Node\_6 136052736\_kB 31570200\_kB 104482536\_kB 78000\_kB 39299 39044 0 Node\_7 136052736\_kB 48505716\_kB 87547020\_kB 77776\_kB 39299 39044 0
```

As you can see the SGA **has been evenly distributed** across NUMA nodes 2 et 3 (see the _Shmem_ column). I did not do more tests (means with ASMM or with AMM or...) with **non** large pages as not using large pages is not an option for me.

- With large page and **no** _processor\_group\_name_ set, the memory allocation looks like:

```
\> sh ./numa\_memory.sh MemTotal MemFree MemUsed Shmem HugePages\_Total HugePages\_Free HugePages\_Surp Node\_0 136052736\_kB 51234984\_kB 84817752\_kB 77824\_kB 39300 37094 0 Node\_1 136052736\_kB 52417252\_kB 83635484\_kB 77772\_kB 39300 37095 0 Node\_2 136052736\_kB 51178872\_kB 84873864\_kB 77904\_kB 39300 37096 0 Node\_3 136052736\_kB 52263300\_kB 83789436\_kB 77872\_kB 39300 37096 0 Node\_4 136024568\_kB 51870848\_kB 84153720\_kB 77864\_kB 39300 37097 0 Node\_5 136052736\_kB 52304932\_kB 83747804\_kB 77852\_kB 39300 37097 0 Node\_6 136052736\_kB 51837040\_kB 84215696\_kB 77840\_kB 39299 37096 0 Node\_7 136052736\_kB 52213888\_kB 83838848\_kB 77860\_kB 39299 37096 0
```

As you can see the large pages have **been evenly allocated** from all the NUMA nodes (this is what has been told in the introduction).

- If you like it, you can get the simple _num\_memory.sh_ script from [here](https://github.com/bdrouvot/numa_scripts).

**Summary**

1. **With large pages** in place and **_processor\_group\_name_ linked** to more than one NUMA node, then the **SGA is not evenly distributed** across the NUMA nodes defined in the cgroup.
2. **With large pages** in place and **_processor\_group\_name_ not set** ,&nbsp; then the **SGA is evenly distributed** across the NUMA nodes.

---
layout: post
title: 'binding (with processor_group_name) versus caging: Facts, Observations and
  Customer cases.'
date: 2015-03-18 19:33:31.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Consolidation
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/PytbBytHvWS
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/ic6CkXklbT
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5984028685731135488&type=U&a=NsB5
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
permalink: "/2015/03/18/binding-with-processor_group_name-versus-caging-facts-observations-and-customer-cases/"
---

**INTRODUCTION**
================

This blog post came to my mind following an interesting conversation I have had on twitter with [Philippe Fierens](http://pfierens.blogspot.be/) and [Karl Arao](https://karlarao.wordpress.com/).

<span style="text-decoration:underline;">Let's go for it:</span>

Nowadays it is very common to consolidate multiple databases on the same server.

One could want to limit the CPU usage of databases and/or ensure guarantee CPU for some databases. I would like to compare two methods to achieve this:

-   Instance Caging (available since 11.2).
-   *processor\_group\_name* (available since 12.1).

This blog post is composed of 4 mains parts:

1.  **Facts**: Describing the features and a quick overview on how to implement them.
2.  **Observations**: What I observed on my lab server. You should observe the same on your environment for most of them.
3.  **OEM and AWR views** for Instance Caging and *processor\_group\_name* with CPU pressure.
4.  **Customer cases**: I cover all the cases I faced so far.

**FACTS**
=========

**Instance caging:** Purpose is to Limit or “cage” the amount of CPU that a database instance can use at any time. It can be enabled **online** (no need to restart the database) in 2 steps:

-    Set “*cpu\_count*” parameter to the maximum number of CPUs the database should be able to use (Oracle advises to set *cpu\_count* to at least 2)

<!-- -->

    SQL> alter system set cpu_count = 2;

-   Set “*resource\_manager\_plan*” parameter to enable CPU Resource Manager

<!-- -->

    SQL> alter system set resource_manager_plan = ‘default_plan’;

Graphical view with multiple databases limited as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/instance_caging_graphical_view1.png" class="aligncenter size-full wp-image-2659" width="621" height="497" alt="instance_caging_graphical_view" />](https://bdrouvot.files.wordpress.com/2015/03/instance_caging_graphical_view1.png)**processor\_group\_name**: Bind a database instance to specific CPUs / NUMA nodes. It is enabled in 4 steps on my RedHat 6.5 machine:

-    Specify the CPUs or NUMA nodes by creating a “processor group”:

Example: Snippet of */etc/cgconfig.conf*:

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
          cpuset.mems=0;
          cpuset.cpus="1-9";
        }
     }

-    Start the cgconfig service:

<!-- -->

    service cgconfig start

-    Set the Oracle parameter “*processor\_group\_name*” to the name of this processor group:

<!-- -->

    SQL> alter system set processor_group_name='oracle' scope=spfile;

-    Restart the Database Instance.

Graphical view with multiple databases bound as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/cpu_binding_graphical_view1.png" class="aligncenter size-full wp-image-2660" width="640" height="175" alt="cpu_binding_graphical_view" />](https://bdrouvot.files.wordpress.com/2015/03/cpu_binding_graphical_view1.png)This graphical view represents the **ideal** configuration, where a database is bound to specific CPUs and NUMA **local** memory (If you need more details, see the observation number 1 later on this post).

<span style="text-decoration:underline;">Remark regarding the Database Availability:</span>

-   The Instance caging can be enabled online.
-   For cgroups/*processor\_group\_name* the database needs to be stopped and started.

**OBSERVATIONS**
================

For brevity, "Caging" is used for "Instance Caging" and "Binding" for "cgroups/*processor\_group\_name*" usages.

Also "CPU" is used when CPU **threads** are referred to, which is the actual **granularity** for caging and binding.

The observations have been made on a 12.1.0.2 database using large pages (*use\_large\_pages=only*). I will not cover the “non large pages” case as not using large pages is not an option for me.

**1.** When you define the cgroup for the binding, you should pay attention to **the NUMA memory and CPUs locality** as it has an impact on the LIO performance (See this [blog post](https://bdrouvot.wordpress.com/2015/01/07/measure-the-impact-of-remote-versus-local-numa-node-access-thanks-to-processor_group_name/ "Measure the impact of remote versus local NUMA node access thanks to processor_group_name")). As you can see (on my specific configuration), remote NUMA nodes access has been slower by about 2 times compare to local access.

**   2.Without binding**, the SGA is **interleaved** across all the NUMA nodes (unless you set the oracle hidden parameter *\_enable\_NUMA\_interleave* to FALSE):

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/numa_mem_alloc_db_no_bind2.png" class="aligncenter size-full wp-image-2663" width="640" height="85" alt="numa_mem_alloc_db_no_bind" />](https://bdrouvot.files.wordpress.com/2015/03/numa_mem_alloc_db_no_bind2.png)

**   3.With binding** (on more than one NUMA nodes), the SGA **is not interleaved** across all the NUMA nodes (SGA bind on nodes 0 and 1):

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/numa_mem_alloc_db_bind_nodes_0_11.png" class="aligncenter size-full wp-image-2665" width="640" height="85" alt="numa_mem_alloc_db_bind_nodes_0_1" />](https://bdrouvot.files.wordpress.com/2015/03/numa_mem_alloc_db_bind_nodes_0_11.png)

**   4.Instance caging has been less performant **(compare to the **cpu binding**) during LIO pressure (by pressure I mean that the database needs **more cpu resources** than the ones it is limited to use) (See this [blog post](https://bdrouvot.wordpress.com/2015/01/15/cpu-binding-processor_group_name-vs-instance-caging-comparison-during-lio-pressure/ "cpu binding (processor_group_name) vs Instance caging comparison during LIO pressure") for more details). Please note that the comparison has been done on the **same** NUMA node (to avoid binding advantage over caging due to observations number 1 and 2).

**   5.**With cpu binding **already** in place (using *processor\_group\_name*) we are able to change **the number of cpus a database is allowed to use on the fly** (See this [blog post](https://bdrouvot.wordpress.com/2015/01/19/modify-on-the-fly-the-cgroup-properties-linked-to-the-processor_group_name-parameter/ "Modify on the fly the cgroup properties linked to the processor_group_name parameter")).

<span style="text-decoration:underline;">Remarks:</span>

-   You can find more details regarding observations 2 and 3 into this [blog post](https://bdrouvot.wordpress.com/2015/02/05/processor_group_name-and-sga-distribution-across-numa-nodes/ "processor_group_name and SGA distribution across NUMA nodes").
-   From 2 and 3 we can conclude that with the caging, the SGA is interleaved across all the NUMA nodes (unless the caging has been setup on top of the binding (mix configuration)).
-   All above mentioned observations are **related to performance**. You may want to run the tests described in observations 1 and 4 in your own environment to **measure the actual impact on your environment**.

**Caging and Binding with CPU pressure: OEM and AWR views**
===========================================================

What do the OEM or AWR views of a database being under CPU pressure show (pressure meaning the database needs **more CPU resources** than it is configured to use) with both caging and binding? Let’s have a look.

**CPU pressure with Caging:**

The database has the caging enabled with the *cpu\_count* parameter set to **6** and is running **9** SLOB users in parallel.

The OEM view:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/new_caging_6_nocg-_for_blog.png" class="aligncenter size-full wp-image-2651" width="640" height="112" alt="new_caging_6_nocg _for_blog" />](https://bdrouvot.files.wordpress.com/2015/03/new_caging_6_nocg-_for_blog.png)As you can see: In average about 6 sessions are running on the CPU and about 3 are waiting in the “Scheduler” wait class.

The AWR view:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/awr_caging_6_nocg.png" class="aligncenter size-full wp-image-2652" width="590" height="239" alt="awr_caging_6_nocg" />](https://bdrouvot.files.wordpress.com/2015/03/awr_caging_6_nocg.png)As you can see: Approximatively 60% of the DB time is spend on CPU and 40% is spend waiting because of the caging (Scheduler wait class).

**CPU pressure with Binding:**

The *processor\_group\_name* is set to a cgroup of **6** CPUs. The Instance is running **9** SLOB users in parallel.

The OEM view:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/new_cg6_for_blog.png" class="aligncenter size-full wp-image-2653" width="640" height="112" alt="new_cg6_for_blog" />](https://bdrouvot.files.wordpress.com/2015/03/new_cg6_for_blog.png)As you can see: In average **6** sessions are running on CPU and **3** are waiting for the CPU (runqueue).

The AWR view:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/awr_cg_6_top10.png" class="aligncenter size-full wp-image-2654" width="640" height="173" alt="awr_cg_6_top10" />](https://bdrouvot.files.wordpress.com/2015/03/awr_cg_6_top10.png)New “Top Events” section as of 12c:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/awr_cg_6_topevents.png" class="size-full wp-image-2655 alignleft" width="433" height="145" alt="awr_cg_6_topevents" />](https://bdrouvot.files.wordpress.com/2015/03/awr_cg_6_topevents.png)

 

 

 

 

To summarize:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/awr_12c_top_event_cg6.png" class="aligncenter size-full wp-image-2656" width="640" height="352" alt="awr_12c_top_event_cg6" />](https://bdrouvot.files.wordpress.com/2015/03/awr_12c_top_event_cg6.png)Then, we can conclude that:

-   65.5% of the DB time has been spent on the CPU.
-   99.62–65.5 = 34.12% of the DB time has been spent into the runqueue.

**CUSTOMER CASES**
==================

The cases below are based on studies for customers.

**Case number 1:** One database uses too much CPU and affects other instances’ performance.** **
------------------------------------------------------------------------------------------------

Then we want to limit its CPU usage:

-   **Caging**:   We are able to cage the database Instance **without** any restart.
-   **Binding**:  We are able to bind the database Instance, but **it needs** a restart.

Example of such a case by implementing Caging:

### [<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/caging_on_the_fly.png" class="aligncenter size-full wp-image-2635" width="640" height="283" alt="caging_on_the_fly" />](https://bdrouvot.files.wordpress.com/2015/03/caging_on_the_fly.png)**What to choose?**

The caging is the best choice in this case (as it is the easiest and we don't need to restart the database).

For the following cases we need to define 2 categories of database:

1.  The paying ones: Customer **paid for guaranteed** CPU resources available.
2.  The **non**-paying ones: Customer **did not pay for guaranteed** CPU resources available.

**Case number 2**: Guarantee CPU resource for some databases. The CPU resource is defined by the customer and needs to be guaranteed at** the database level**.
---------------------------------------------------------------------------------------------------------------------------------------------------------------

### **Caging:**

-   **All** the databases need to be caged (if not, there is no guaranteed resources availability as one non-caged could take all the CPU).
-   As a consequence, you can't mix paying and non-paying customer without caging the non-paying ones.
-   You have to know how to cage each database individually to be sure that *sum(cpu\_count) &lt;= Total number of threads* (If not, CPU resources can't be guaranteed).
-   Be sure that *sum(cpu\_count) of non-paid &lt;= "CPU Total – number of paid CPU"* (If not, CPU resources can't be guaranteed).
-   There are a maximum number of databases that you can put on a machine: As *cpu\_count* &gt;=2 (see facts) then the maximum number of databases = Total number of threads /2.

Graphical view as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/all_caged_databases_level.png" class="aligncenter size-full wp-image-2668" width="601" height="484" alt="all_caged_databases_level" />](https://bdrouvot.files.wordpress.com/2015/03/all_caged_databases_level.png)If you need to add a new database (a paying one or a non-paying one) and you are already in a situation where "*sum(cpu\_count) = Total number of threads*" then you have to review the caging of all non-paying databases (to not over allocate the machine).

### **Binding:**

Each database is linked to a set of CPU (There is no overlap, means that a CPU can be part of only one cgroup). Then, create:

-   One cgroup **per paying** database.
-   One cgroup for **all the non-paying** databases.

That way we can easily mix paying and non-paying customer on the same machine.

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/bind_all_databases_level.png" class="aligncenter size-full wp-image-2669" width="640" height="201" alt="bind_all_databases_level" />](https://bdrouvot.files.wordpress.com/2015/03/bind_all_databases_level.png)If you need to add a new database then, if this is:

-   A non-paying one: Put it into the non-paying group.
-   A paying one: Create a new cgroup for it, assigning CPU taken away from non-paying ones if needed.

### **Binding + caging mix:**

-   create a cgroup for **all the paying** databases and then **put the caging on each paying** database.
-   Put the **non-paying into another cgroup** (no overlap with the paying one) without caging.

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/bind_caging_mix.png" class="aligncenter size-full wp-image-2670" width="640" height="180" alt="bind_caging_mix" />](https://bdrouvot.files.wordpress.com/2015/03/bind_caging_mix.png)If you need to add a new database then, if this is:

-   A non-paying one: Put it into the non-paying group.
-   A paying one: Extend the paying group if needed (taking CPU away from non-paying ones) and cage it accordingly.

### **What to choose?**

I would say there is no "best choice": It all depends of the number of database we are talking about (For a large number of databases I would use the mix approach). And don't forget that with the caging option your can't put more than number of threads/2 databases.

**Case number 3:** Guarantee CPU resource for **exactly one** **group** of databases. The CPU resource needs to be guaranteed **at the group level** (Imagine that customer paid for 24 CPUs whatever the number of database is).
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **Caging:**

We have 2 options:

1\. Option 1:

-   Cage all the **non-paying databases.  
    **
-   Be sure that *sum(cpu\_count) of non-paid &lt;= "CPU Total – number of paid CPU"* (If not, CPU resources can't be guaranteed).
-   **Don't cage** the paying databases.

That way the paying databases have guaranteed **at least** the resources **they paid for.**

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/caging_1group_option1.png" class="aligncenter size-full wp-image-2672" width="640" height="435" alt="caging_1group_option1" />](https://bdrouvot.files.wordpress.com/2015/03/caging_1group_option1.png)If you need to add a new database then, if this is:

-   A non-paying one: Cage it but you may need to re-cage all the non-paying ones to guarantee the paying ones still get its "minimum" resource.
-   A paying one: Nothing to do.

2\. Option 2:

-   Cage **all** the databases (paying and non-paying).
-   You have to know how to cage each database individually to be sure that *sum(cpu\_count) &lt;= Total number of threads* (If not, CPU resources can't be guaranteed).
-   Be sure that *sum(cpu\_count) of non-paid &lt;= "CPU Total – number of paid CPU"* (If not, CPU resources can't be guaranteed).

That way the paying databases have guaranteed **exactly** the resources **they paid for.**

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/caging_1group_option2.png" class="aligncenter size-full wp-image-2673" width="640" height="435" alt="caging_1group_option2" />](https://bdrouvot.files.wordpress.com/2015/03/caging_1group_option2.png)If you need to add a new database (a paying one or a non-paying one) and you are already in a situation where "*sum(cpu\_count) = Total number of threads*" then you have to review the caging of all non-paying databases (to not over allocate the machine).

### **Binding**:

Again 2 options:

1\. Option 1:

-   **Put all the non-paying** databases into a "CPU Total – number of paid CPU " cgroup.
-   Allow all the CPUs to be used by the paying databases (**don't create a cgroup** for the **paying** databases).
-   That way the paying group has guaranteed **at least** the resources **it paid for.**

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/bind_1group_option1.png" class="aligncenter size-full wp-image-2675" width="640" height="357" alt="bind_1group_option1" />](https://bdrouvot.files.wordpress.com/2015/03/bind_1group_option1.png)If you need to add a new database then, if this is:

-   A non-paying one: Put it in the non-paying cgroup.
-   A paying one: Create it outside the non-paying cgroup.

2\. Option 2:

-   **Put all the paying databases** into a cgroup.
-   **Put all the non-paying databases into another cgroup** (no overlap with the paying cgroup).

That way the paying databases have guaranteed **exactly **the resources **they paid for.**

Graphical View:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/bind_1group_option2.png" class="aligncenter size-full wp-image-2676" width="640" height="173" alt="bind_1group_option2" />](https://bdrouvot.files.wordpress.com/2015/03/bind_1group_option2.png)If you need to add a new database, then if this is:

-   A non-paying one: Put it in the non-paying cgroup.
-   A paying one: Put it in the paying cgroup.

### **Binding + caging mix:**

I don't see any benefits here.

### **What to choose?**

I like the option 1) in both cases as it allows customer to get more than what they paid for (with the guarantee of what they paid for). Then the choice between caging and binding really depends of the number of databases we are talking about (binding is preferable and easily manageable for large number of databases).

**Case number 4: ** The need to guarantee CPU resources for **more than one group** of databases (Imagine that one group=one customer). The CPU resource needs to be assured at **each group level**.
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### **Caging:**

-   **All** the databases need to be caged (if not, there is no guaranteed resources availability as one non-caged could take all the CPU).
-   As a consequence, you can't mix paying and non-paying customer without caging the non-paying ones.
-   You have to know how to cage each database individually to be sure that *sum(cpu\_count) &lt;= Total number of threads* (If not, CPU resources can't be guaranteed).
-   Be sure that *sum(cpu\_count) of non-paid &lt;= "CPU Total – number of paid CPU"* (If not, CPU resources can't be guaranteed).
-   There are a maximum number of databases that you can put on a machine: As *cpu\_count* &gt;=2 (see facts) then the maximum number of databases = Total number of threads /2.

Graphical View as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/caging_2groups.png" class="aligncenter size-full wp-image-2677" width="640" height="435" alt="caging_2groups" />](https://bdrouvot.files.wordpress.com/2015/03/caging_2groups.png)If you need to add a new database (a paying one or a non-paying one) and you are already in a situation where "*sum(cpu\_count) = Total number of threads*" then you have to review the caging of all non-paying databases (to not over allocate the machine).

Overlapping is not possible in that case (means you can't guarantee resources with overlapping).

### **Binding:**

-   Create a cgroup for **each** paying database group and also for the non-paying ones (no overlap between all the groups).
-   Overlapping is not possible in that case (means you can't guarantee resources with overlapping).

Graphical view as an example:

[<img src="%7B%7B%20site.baseurl%20%7D%7D/assets/images/bind_2groups.png" class="aligncenter size-full wp-image-2678" width="640" height="115" alt="bind_2groups" />](https://bdrouvot.files.wordpress.com/2015/03/bind_2groups.png)

If you need to add a database, then simply put it in the right group.

### **Binding + caging mix:**

I don't see any benefits here.

### **What to choose? **

It really depends of the number of databases we are talking about (binding is preferable and easily manageable for large number of databases).

**REMARKS**
===========

-    If you see (or faced) more cases, then feel free to share and to comment on this blog post.
-   I did not pay attention on potential impacts on LIO performance linked to any possible choice (caging vs binding with or without NUMA). I just mentioned them into the observations, but did not include them into the use cases (I just discussed the feasibility of the implementation).
-   I really like the options 1 into the case number 3. This option has been proposed by Karl during our conversation.

**CONCLUSION**
==============

The choice between caging and binding depends of:

-   Your needs.
-   The number of databases you are consolidating on the server.
-   Your performance expectations.

I would like to thank you [Karl Arao](https://karlarao.wordpress.com/), [Frits Hoogland](https://fritshoogland.wordpress.com/), [Yves Colin](https://ycolin.wordpress.com/) and [David Coelho](https://twitter.com/ohleoc) for their efficient reviews of this blog post.

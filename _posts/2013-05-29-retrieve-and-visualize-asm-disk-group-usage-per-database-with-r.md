---
layout: post
title: Retrieve and visualize ASM Disk Group Usage per database with R
date: 2013-05-29 16:03:01.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- R scripts
tags: [ASM, oracle]
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:148;}s:2:"wp";a:1:{i:0;i:32;}}
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/05/29/retrieve-and-visualize-asm-disk-group-usage-per-database-with-r/"
---

If you are using Oracle Enterprise Manager and ASM, then you may already be familiar with the following page that displays the Disk Group Usage per database for a particular Disk Group:

<img src="{{ site.baseurl }}/assets/images/oem_archive_dg_usage.png" class="aligncenter size-full wp-image-1022" width="620" height="160" alt="oem_archive_dg_usage" />
And it is most likely that you don't need to read this little post, as it provides a R script to achieve more or less the same result ;-).

But if you are not using Oracle Enterprise Manager, then you may find this post useful.

I created a **R script "<span style="color:#0000ff;">graph\_asm\_dg\_usage.r</span> " **(You can download it from this [repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit "Perl Scripts Shared Directory"))** that provides:**

1.  **A graph** similar to the OEM one.
2.  **A pdf file** that contains the graph.
3.  **A text file** that contains the values used to build the graph.

The graph is available from both outputs (X11 and the pdf file). In case the X11 environment does not work, **the pdf file is generated anyway**.

<span style="text-decoration:underline;">Let's launch it to see the result:</span>

    ./graph_asm_dg_usage.r 
    Building the thin jdbc connection string....

    host ?: bdt_host
    port ?: 1521
    service_name for ASM ?: +ASM
    sys as sysasm password ?: dontreadthis
    Display which disk group (no quotation marks, no +) ?: ARCHIVE
    Loading required package: methods
    Loading required package: DBI
    Loading required package: rJava
    Please enter any key to exit:

<span style="text-decoration:underline;">The output is the following:</span>

<img src="{{ site.baseurl }}/assets/images/archive_dg_usage.png" class="aligncenter size-full wp-image-1024" width="620" height="475" alt="archive_dg_usage" />

<span style="text-decoration:underline;">As you can see you are prompted for:</span>

-   jdbc thin “like” details to connect to the ASM instance (You can launch the R script outside the host hosting the ASM instance).
-   oracle sys user password.
-   The ASM disk group you want to focus on.

<span style="text-decoration:underline;">**Remarks:**</span>

-   It has been tested on a 11.2.0.3 asm instance.
-   If you want to install R, you should begin to read  "Getting Starting” from [this link](http://www.r-project.org/).

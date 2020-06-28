---
layout: post
title: Running 12.1.0.2 Oracle Database on Docker
date: 2016-08-18 18:07:07.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Other
tags: [oracle]
meta:
  _edit_last: '40807211'
  geo_public: '0'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6172086245448048640&type=U&a=hTS2
  _publicize_job_id: '1'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/766320560691290112";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/5UGjaWZa2ZH
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/08/18/running-12-1-0-2-oracle-database-on-docker/"
---

Introduction
============

As you may have noticed, Oracle has released a few weeks ago [Docker build files for the Oracle Database on Github](https://github.com/oracle/docker-images/tree/master/OracleDatabase "Oracle Database Docker build files") and an associated [blog post](https://blogs.oracle.com/developer/entry/creating_and_oracle_database_docker). Should you read the blog post, you would notice that you need to download manually the oracle binaries.

The purpose of this post is to provide a modified version of the original build files (focusing on the 12.1.0.2 database EE only), so that the oracle binaries are downloaded during the build of the image (thanks to Maris Elsins's [getMOSPatch](https://www.pythian.com/blog/getmospatch-sh-downloading-patches-from-my-oracle-support/) script).

Create the 12.1.0.2 database docker image
=========================================

The manual steps are:

### Install the docker engine (You can find detailed instructions [here](https://docs.docker.com/engine/installation/linux/)), basically you have to add the yum repository and launch:

    root@docker# yum install docker-engine

### Change the maximum container size to 20 GB:

    root@docker# docker daemon --storage-opt dm.basesize=20G

### Clone those files from github:

    root@docker# git clone https://github.com/bdrouvot/oracledb-docker.git

### Update the oracledb-docker/Dockerfile file (ENV section only) with the appropriate values:

-   ORACLE\_PWD="&lt;put\_the\_password\_you\_want&gt;"
-   MOSU="&lt;your\_oracle\_support\_username&gt;"
-   MOSP="&lt;your\_oracle\_support\_password&gt;"

### Build the Image:

    root@docker# docker build --force-rm=true --no-cache=true -t oracle/database:12.1.0.2 .

That's it, now we can:

Use the Image
=============

Simply start a docker container that way:

    root@docker# docker run -p 1521:1521 --name bdt12c oracle/database:12.1.0.2

The host that is running the docker engine is "docker".  You can connect to the database as you would do normally, for example, using Oracle SQL Developper:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2016-08-18-at-18-46-30.png" class="aligncenter size-full wp-image-3120" width="640" height="333" alt="Screen Shot 2016-08-18 at 18.46.30" />

Note that the Hostname is "docker", that is to say the one that is hosting the docker engine.

Remarks
=======

-   At the time of this writing Oracle Database on Docker is **NOT** supported by Oracle. Use these files at your own discretion.
-   If you are interested in this kind of stuff, then you should also read Frits Hoogland's [blog post](https://fritshoogland.wordpress.com/2016/08/02/a-total-unattended-install-of-linux-and-the-oracle-database/).
-   The Dockerfile used is very closed to the one provided by Oracle (Gerald Venzl). Only a few things have been modified to download the oracle binaries during the image creation.
-   Thanks to Maris Elsins for [getMOSPatch](https://www.pythian.com/blog/getmospatch-sh-downloading-patches-from-my-oracle-support/).

---
layout: post
title: 'Welcome to pgdfv: PostgreSQL data file visualizer'
date: 2017-12-10 16:42:42.000000000 +01:00
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
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6345648787758944256&type=U&a=FFC-
  _publicize_job_id: '1'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/939883104314982400";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _oembed_cf31d0bf4d450319ecb77d19dff328b3: "{{unknown}}"
  _oembed_c14720333fc533ffe7f6b3175598edd1: "{{unknown}}"
  _oembed_6e930e8c608f97c3d1c840c78ea3a535: "{{unknown}}"
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/"
---

Introduction
------------

As you may know the PostgreSQL database page contains a lot of informations that is documented [here.](https://www.postgresql.org/docs/9.6/static/storage-page-layout.html) A great study has been done by Frits Hoogland in [this series of blogposts](https://fritshoogland.wordpress.com/2017/07/07/postgresql-block-internals-part-3/). I strongly recommend to read Frits series before to read this blog post (unless you are familiar with PostgreSQL block internals).

By reading the contents of a page we can extract:

-   The percentage of free space within a page
-   The percentage of current rows within a page
-   The percentage of unused rows within a page

Welcome to pgdfv
----------------

**pgdfv** stands for: **P**ost**g**reSQL **d**ata **f**ile **v**isualizer. It helps to visualize the data file pages in a easy way.

For each block:

-   A color is assigned (depending of the percentage of free space)
-   The color can be displayed in 2 ways (depending if more than 50 percent of the rows are current)
-   A number is assigned (based on the percentage of unused rows)

At the end the utility provides a summary for all the blocks visited. As a picture is worth a thousand words, let's see some examples.

Examples
--------

Let's create a table, insert 4 rows in it and inspect its content:

    pgdbv=# create table bdtable(id int not null, f varchar(30) );
    CREATE TABLE
    pgdbv=# insert into bdtable ( id, f ) values (1, 'aaaaaaaaaa');
    INSERT 0 1
    pgdbv=# insert into bdtable ( id, f ) values (2, 'aaaaaaaaaa');
    INSERT 0 1
    pgdbv=# insert into bdtable ( id, f ) values (3, 'aaaaaaaaaa');
    INSERT 0 1
    pgdbv=# insert into bdtable ( id, f ) values (4, 'aaaaaaaaaa');
    INSERT 0 1
    pgdbv=# checkpoint;
    CHECKPOINT
    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |   8152 |        1 |     39 |   1945 |      0 |        0 | (0,1)  |           2 |       2050 |     24 |        |       | \x010000001761616161616161616161
      2 |   8112 |        1 |     39 |   1946 |      0 |        0 | (0,2)  |           2 |       2050 |     24 |        |       | \x020000001761616161616161616161
      3 |   8072 |        1 |     39 |   1947 |      0 |        0 | (0,3)  |           2 |       2050 |     24 |        |       | \x030000001761616161616161616161
      4 |   8032 |        1 |     39 |   1948 |      0 |        0 | (0,4)  |           2 |       2050 |     24 |        |       | \x040000001761616161616161616161
    (4 rows)

In PostgreSQL, each table is stored in a separate file. When a table exceeds 1 GB, it is divided into gigabyte-sized segments.

Let's check which file contains the table:

    pgdbv=# SELECT pg_relation_filepath('bdtable');
     pg_relation_filepath
    ----------------------
     base/16416/16448

Let's use the utility on this file:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-23-26.png" class="aligncenter size-full wp-image-3276" width="1100" height="134" />

So one block is detected, with more than 75% of free space (so the green color), more than 50% of the rows are current (100% in our case as tx\_max = 0 for all the rows) and unused are less than 10% (0 is displayed) (0% in our case as no rows with lp\_flags = 0).

Let's delete, 3 rows:

    pgdbv=# delete from bdtable where id <=3;
    DELETE 3
    pgdbv=# checkpoint;
    CHECKPOINT
    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |   8152 |        1 |     39 |   1945 |   1949 |        0 | (0,1)  |        8194 |        258 |     24 |        |       | \x010000001761616161616161616161
      2 |   8112 |        1 |     39 |   1946 |   1949 |        0 | (0,2)  |        8194 |        258 |     24 |        |       | \x020000001761616161616161616161
      3 |   8072 |        1 |     39 |   1947 |   1949 |        0 | (0,3)  |        8194 |        258 |     24 |        |       | \x030000001761616161616161616161
      4 |   8032 |        1 |     39 |   1948 |      0 |        0 | (0,4)  |           2 |       2306 |     24 |        |       | \x040000001761616161616161616161
    (4 rows)

and launch the tool again:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-24-23.png" class="aligncenter size-full wp-image-3277" width="1100" height="132" />

As you can see we still have more than 75% of free space in the block but the way to display the color has been changed (because now **less** than 50% of the rows are current aka tx\_max = 0) and unused is still less than 10% (0 is displayed).

Let's vacuum the table:

    pgdbv=# vacuum bdtable;
    VACUUM
    pgdbv=# checkpoint;
    CHECKPOINT
    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
      2 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
      3 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
      4 |   8152 |        1 |     39 |   1948 |      0 |        0 | (0,4)  |           2 |       2306 |     24 |        |       | \x040000001761616161616161616161
    (4 rows)

and launch the utility:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-25-38.png" class="aligncenter size-full wp-image-3278" width="1100" height="124" />

As you can see we still have more than 75% of free space, less than 50% of the rows are current (only one in our case) and now there is between 70 and 80% of unused rows in the block (so 7 is displayed).

The legend and summary are **dynamic** and depend of the contents of the scanned blocks.

For example, on a newly created table (made of 355 blocks) you could end up with something like:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-29-03.png" class="aligncenter size-full wp-image-3280" width="1100" height="125" />

Then, you delete half of the rows:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-30-27.png" class="aligncenter size-full wp-image-3281" width="1100" height="144" />

Then, once the table has been vacuum:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-31-34.png" class="aligncenter size-full wp-image-3282" width="1100" height="141" />

And once new rows have been inserted:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-32-40.png" class="aligncenter size-full wp-image-3283" width="1100" height="154" />

Remarks
-------

-   The tool is available in this [repository](https://github.com/bdrouvot/pgdfv).
-   The tool is 100% inspired by Frits Hoogland blogpost series and by [odbv](http://blog.ora-600.pl/2016/11/03/oracle-database-block-visualizer/) (written by Kamil Stawiarski) that can be used to visualize oracle database blocks.

Conclusion
----------

pgdfv is a new utility that can be used to visualize PostgreSQL data file pages.

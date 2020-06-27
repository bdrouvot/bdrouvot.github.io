---
layout: post
title: Visualize PostgreSQL index file with pgdfv
date: 2017-12-21 20:09:06.000000000 +01:00
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
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/943921311738486784";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _publicize_job_id: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6349686993726902273&type=U&a=9VLC
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2017/12/21/visualize-postgresql-index-file-with-pgdfv/"
---

Introduction
------------

In the previous [blog post](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/) pgdfv (PostgreSQL data file visualizer) has been introduced. At that time the utility was able to display data file. It is now able to display index file. If you are not familiar with PostgreSQL block internals I would suggest to read Frits Hoogland study in [this series of blogposts.](https://fritshoogland.wordpress.com/2017/07/07/postgresql-block-internals-part-3/)

The utility usage is:

    $ ./pgdfv
    -df     Path to a datafile (mandatory if indexfile is used)
    -if     Path to an indexfile
    -b      Block size (default 8192)

As you can see you can now specify an indexfile. In that case the following information will be displayed:

-   The percentage of free space within an index page
-   The percentage of current rows an index page refers to
-   The percentage of HOT redirect rows an index page refers to

It works for B-tree index and will display those informations for the leaf blocks only.

As a picture is worth a thousand words, let’s see some examples.

Examples
--------

Let’s create a table, an index, insert 4 rows:

    pgdbv=# create table bdtable(id int not null, f varchar(30) );
    CREATE TABLE
    pgdbv=# insert into bdtable ( id, f ) values (1, 'aaaaaaaaaa'), (2, 'bbbbbbbbbb'), (3, 'cccccccccc'), (4, 'dddddddddd');
    INSERT 0 4
    pgdbv=# create index bdtindex on bdtable (f);
    CREATE INDEX

and inspect the table and index content:

    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |   8152 |        1 |     39 |   2489 |      0 |        0 | (0,1)  |           2 |       2050 |     24 |        |       | \x010000001761616161616161616161
      2 |   8112 |        1 |     39 |   2489 |      0 |        0 | (0,2)  |           2 |       2050 |     24 |        |       | \x020000001762626262626262626262
      3 |   8072 |        1 |     39 |   2489 |      0 |        0 | (0,3)  |           2 |       2050 |     24 |        |       | \x030000001763636363636363636363
      4 |   8032 |        1 |     39 |   2489 |      0 |        0 | (0,4)  |           2 |       2050 |     24 |        |       | \x040000001764646464646464646464
    (4 rows)

    pgdbv=# select * from bt_page_items('bdtindex',1);
     itemoffset | ctid  | itemlen | nulls | vars |                      data
    ------------+-------+---------+-------+------+-------------------------------------------------
              1 | (0,1) |      24 | f     | t    | 17 61 61 61 61 61 61 61 61 61 61 00 00 00 00 00
              2 | (0,2) |      24 | f     | t    | 17 62 62 62 62 62 62 62 62 62 62 00 00 00 00 00
              3 | (0,3) |      24 | f     | t    | 17 63 63 63 63 63 63 63 63 63 63 00 00 00 00 00
              4 | (0,4) |      24 | f     | t    | 17 64 64 64 64 64 64 64 64 64 64 00 00 00 00 00
    (4 rows)

In PostgreSQL, each table is stored in a separate file. When a table exceeds 1 GB, it is divided into gigabyte-sized segments.

Let’s check which file contains the table and which one contains the index:

    pgdbv=# SELECT pg_relation_filepath('bdtable');
     pg_relation_filepath
    ----------------------
     base/16416/16497
    (1 row)

    pgdbv=# SELECT pg_relation_filepath('bdtindex');
     pg_relation_filepath
    ----------------------
     base/16416/16503
    (1 row)

So that we can use the utility on those files that way:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-21-at-16-41-53.png" class="aligncenter size-full wp-image-3323" width="1100" height="365" />

So the data block has been displayed (same behavior as the previous [blog post](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/)) and also 2 index blocks.

The index block display can be :

-   a letter or a question mark: M for meta-page, R for root page and ? for "non root, non meta and non leaf".
-   a number: It represents the percentage of HOT redirect rows the index refers to (instead of unused rows as it is the case for the data block).

The number is displayed only in case of a leaf block.

So, one leaf index block has been detected with more than 75% of free space (so the green color), more than 50% of the rows the index refers to are current (100% in our case as tx\_max = 0 for all the rows) and HOT redirect are less than 10% (0 is displayed) (0% in our case as no rows with lp\_flags = 2).

Let's update 3 rows: one update on the indexed column and 2 updates on the non indexed column:

    pgdbv=# update bdtable set f='aqwzsxedc' where id =1;
    UPDATE 1
    pgdbv=# update bdtable set id=id+10 where id in (2,3);
    UPDATE 2
    pgdbv=# checkpoint;
    CHECKPOINT
    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |   8152 |        1 |     39 |   2489 |   2490 |        0 | (0,5)  |           2 |       1282 |     24 |        |       | \x010000001761616161616161616161
      2 |   8112 |        1 |     39 |   2489 |   2491 |        0 | (0,6)  |       16386 |        258 |     24 |        |       | \x020000001762626262626262626262
      3 |   8072 |        1 |     39 |   2489 |   2491 |        0 | (0,7)  |       16386 |        258 |     24 |        |       | \x030000001763636363636363636363
      4 |   8032 |        1 |     39 |   2489 |      0 |        0 | (0,4)  |           2 |       2306 |     24 |        |       | \x040000001764646464646464646464
      5 |   7992 |        1 |     38 |   2490 |      0 |        0 | (0,5)  |           2 |      10498 |     24 |        |       | \x01000000156171777a7378656463
      6 |   7952 |        1 |     39 |   2491 |      0 |        0 | (0,6)  |       32770 |      10242 |     24 |        |       | \x0c0000001762626262626262626262
      7 |   7912 |        1 |     39 |   2491 |      0 |        0 | (0,7)  |       32770 |      10242 |     24 |        |       | \x0d0000001763636363636363636363
    (7 rows)

    pgdbv=# select * from bt_page_items('bdtindex',1);
     itemoffset | ctid  | itemlen | nulls | vars |                      data
    ------------+-------+---------+-------+------+-------------------------------------------------
              1 | (0,1) |      24 | f     | t    | 17 61 61 61 61 61 61 61 61 61 61 00 00 00 00 00
              2 | (0,5) |      24 | f     | t    | 15 61 71 77 7a 73 78 65 64 63 00 00 00 00 00 00
              3 | (0,2) |      24 | f     | t    | 17 62 62 62 62 62 62 62 62 62 62 00 00 00 00 00
              4 | (0,3) |      24 | f     | t    | 17 63 63 63 63 63 63 63 63 63 63 00 00 00 00 00
              5 | (0,4) |      24 | f     | t    | 17 64 64 64 64 64 64 64 64 64 64 00 00 00 00 00
    (5 rows)

And launch the tool again:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-21-at-16-47-34.png" class="aligncenter size-full wp-image-3324" width="1100" height="367" />

As you can see we still have more than 75% of free space in the leaf index block but the way to display the color has been changed (because now **less** than 50% of the rows the index refers to are current aka tx\_max = 0) and HOT redirect is still less than 10% (0 is displayed).

Let’s vacuum the table:

    pgdbv=# vacuum  bdtable;
    VACUUM
    pgdbv=# checkpoint;
    CHECKPOINT
    pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
     lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
    ----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
      1 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       |
      2 |      6 |        2 |      0 |        |        |          |        |             |            |        |        |       |
      3 |      7 |        2 |      0 |        |        |          |        |             |            |        |        |       |
      4 |   8152 |        1 |     39 |   2489 |      0 |        0 | (0,4)  |           2 |       2306 |     24 |        |       | \x040000001764646464646464646464
      5 |   8112 |        1 |     38 |   2490 |      0 |        0 | (0,5)  |           2 |      10498 |     24 |        |       | \x01000000156171777a7378656463
      6 |   8072 |        1 |     39 |   2491 |      0 |        0 | (0,6)  |       32770 |      10498 |     24 |        |       | \x0c0000001762626262626262626262
      7 |   8032 |        1 |     39 |   2491 |      0 |        0 | (0,7)  |       32770 |      10498 |     24 |        |       | \x0d0000001763636363636363636363
    (7 rows)

    pgdbv=# select * from bt_page_items('bdtindex',1);
     itemoffset | ctid  | itemlen | nulls | vars |                      data
    ------------+-------+---------+-------+------+-------------------------------------------------
              1 | (0,5) |      24 | f     | t    | 15 61 71 77 7a 73 78 65 64 63 00 00 00 00 00 00
              2 | (0,2) |      24 | f     | t    | 17 62 62 62 62 62 62 62 62 62 62 00 00 00 00 00
              3 | (0,3) |      24 | f     | t    | 17 63 63 63 63 63 63 63 63 63 63 00 00 00 00 00
              4 | (0,4) |      24 | f     | t    | 17 64 64 64 64 64 64 64 64 64 64 00 00 00 00 00
    (4 rows)

and launch the utility:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-21-at-16-50-52.png" class="aligncenter size-full wp-image-3325" width="1100" height="362" />

As you can see we still have more than 75% of free space. Now more than 50% of the rows the index refers to are current. The index refers to between 50 and 60% of HOT redirect rows (so 5 is displayed) (for ctid (0,2) and (0,3): so 2 rows out of 4 the index refers to).

The legend and summary are **dynamic** and depend of the contents of the scanned blocks (data and index).

For example on a table made of 757 blocks, you could end up with something like:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-21-at-17-16-49.png" class="aligncenter size-full wp-image-3327" width="1100" height="577" />

The same table, once FULL vacuum would produce:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-21-at-17-21-25.png" class="aligncenter size-full wp-image-3328" width="1100" height="415" />

Remarks
-------

-   The tool is available in this [repository](https://github.com/bdrouvot/pgdfv).
-   The tool is for B-tree index only.
-   The tool is 100% inspired by Frits Hoogland blogpost series and by [odbv](http://blog.ora-600.pl/2016/11/03/oracle-database-block-visualizer/) (written by Kamil Stawiarski) that can be used to visualize oracle database blocks.

Conclusion
----------

pgdfv can now be used to visualize PostgreSQL data and index file pages.

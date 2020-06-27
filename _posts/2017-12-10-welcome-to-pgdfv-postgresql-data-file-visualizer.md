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
<h2>Introduction</h2>
<p>As you may know the PostgreSQL database page contains a lot of informations that is documented <a href="https://www.postgresql.org/docs/9.6/static/storage-page-layout.html" target="_blank" rel="noopener">here.</a> A great study has been done by Frits Hoogland in <a href="https://fritshoogland.wordpress.com/2017/07/07/postgresql-block-internals-part-3/" target="_blank" rel="noopener">this series of blogposts</a>. I strongly recommend to read Frits series before to read this blog post (unless you are familiar with PostgreSQL block internals).</p>
<p>By reading the contents of a page we can extract:</p>
<ul>
<li>The percentage of free space within a page</li>
<li>The percentage of current rows within a page</li>
<li>The percentage of unused rows within a page</li>
</ul>
<h2>Welcome to pgdfv</h2>
<p><strong>pgdfv</strong> stands for: <strong>P</strong>ost<strong>g</strong>reSQL <strong>d</strong>ata <strong>f</strong>ile <strong>v</strong>isualizer. It helps to visualize the data file pages in a easy way.</p>
<p>For each block:</p>
<ul>
<li>A color is assigned (depending of the percentage of free space)</li>
<li>The color can be displayed in 2 ways (depending if more than 50 percent of the rows are current)</li>
<li>A number is assigned (based on the percentage of unused rows)</li>
</ul>
<p>At the end the utility provides a summary for all the blocks visited. As a picture is worth a thousand words, let's see some examples.</p>
<h2>Examples</h2>
<p>Let's create a table, insert 4 rows in it and inspect its content:</p>
<pre style="padding-left:30px;">pgdbv=# create table bdtable(id int not null, f varchar(30) );
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
(4 rows)</pre>
<p>In PostgreSQL, each table is stored in a separate file. When a table exceeds 1 GB, it is divided into gigabyte-sized segments.</p>
<p>Let's check which file contains the table:</p>
<pre style="padding-left:30px;">pgdbv=# SELECT pg_relation_filepath('bdtable');
 pg_relation_filepath
----------------------
 base/16416/16448
</pre>
<p>Let's use the utility on this file:</p>
<p><a href="https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-23-26/" rel="attachment wp-att-3276"><img class="aligncenter size-full wp-image-3276" src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-23-26.png" alt="" width="1100" height="134" /></a></p>
<p>So one block is detected, with more than 75% of free space (so the green color), more than 50% of the rows are current (100% in our case as tx_max = 0 for all the rows) and unused are less than 10% (0 is displayed) (0% in our case as no rows with lp_flags = 0).</p>
<p>Let's delete, 3 rows:</p>
<pre style="padding-left:30px;">pgdbv=# delete from bdtable where id &lt;=3;
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
(4 rows)</pre>
<p>and launch the tool again:</p>
<p><a href="https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-24-23/" rel="attachment wp-att-3277"><img class="aligncenter size-full wp-image-3277" src="{{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-24-23.png" alt="" width="1100" height="132" /></a></p>
<p>As you can see we still have more than 75% of free space in the block but the way to display the color has been changed (because now <strong>less</strong> than 50% of the rows are current aka tx_max = 0) and unused is still less than 10% (0 is displayed).</p>
<p>Let's vacuum the table:</p>
<pre style="padding-left:30px;">pgdbv=# vacuum bdtable;
VACUUM
pgdbv=# checkpoint;
CHECKPOINT
pgdbv=# select * from heap_page_items(get_raw_page('bdtable',0));
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |              t_data
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------
1 | 0 | 0 | 0 | | | | | | | | | | 2 | 0 | 0 | 0 | | | | | | | | | | 3 | 0 | 0 | 0 | | | | | | | | | | 4 | 8152 | 1 | 39 | 1948 | 0 | 0 | (0,4) | 2 | 2306 | 24 | | | \x040000001761616161616161616161 (4 rows)

and launch the utility:

[![]({{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-25-38.png)](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-25-38/)

As you can see we still have more than 75% of free space, less than 50% of the rows are current (only one in our case) and now there is between 70 and 80% of unused rows in the block (so 7 is displayed).

The legend and summary are **dynamic** and depend of the contents of the scanned blocks.

For example, on a newly created table (made of 355 blocks) you could end up with something like:

[![]({{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-29-03.png)](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-29-03/)

Then, you delete half of the rows:

[![]({{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-30-27.png)](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-30-27/)

Then, once the table has been vacuum:

[![]({{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-31-34.png)](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-31-34/)

And once new rows have been inserted:

[![]({{ site.baseurl }}/assets/images/screen-shot-2017-12-10-at-16-32-40.png)](https://bdrouvot.wordpress.com/2017/12/10/welcome-to-pgdfv-postgresql-data-file-visualizer/screen-shot-2017-12-10-at-16-32-40/)

## Remarks

- The tool is available in this [repository](https://github.com/bdrouvot/pgdfv).
- The tool is 100% inspired by Frits Hoogland blogpost series and by [odbv](http://blog.ora-600.pl/2016/11/03/oracle-database-block-visualizer/)&nbsp;(written by&nbsp;Kamil Stawiarski) that can be used to visualize oracle database blocks.

## Conclusion

pgdfv is a new utility that can be used to visualize PostgreSQL data file pages.


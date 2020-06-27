---
layout: post
title: Retrieve PostgreSQL variable-length storage information thanks to pageinspect
date: 2020-01-18 13:49:47.000000000 +01:00
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
  _wpas_skip_7950430: '1'
  timeline_notification: '1579351791'
  _publicize_job_id: '39682411733'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1218515849352548358";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_linkedin_url: ''
  _publicize_done_22399261: '1'
  _wpas_done_23399164: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2020/01/18/retrieve-postgresql-variable-length-storage-information-thanks-to-pageinspect/"
---
<h3>Introduction</h3>
<p>In PostgreSQL a variable-length datatype value can be stored in-line or out-of-line (as a TOAST). It can also be compressed or not (see the <a href="https://www.postgresql.org/docs/current/storage-toast.html" target="_blank" rel="noopener">documentation</a> for more details).</p>
<p>Let's make use of the <a href="https://www.postgresql.org/docs/current/pageinspect.html" target="_blank" rel="noopener">pageinspect</a> extension and the information about variable-length datatype found in <a href="https://github.com/postgres/postgres/blob/master/src/include/postgres.h" target="_blank" rel="noopener">postgres.h</a> to build a query to retrieve tuples variable-length storage information.</p>
<h3>The query</h3>
<p>The query is the following:</p>
<pre style="padding-left:40px;">$ cat toast_info.sql
select
t_ctid,
-- See postgres.h
CASE
  WHEN (fo is NULL) THEN 'null'
  -- VARATT_IS_EXTERNAL_ONDISK: VARATT_IS_EXTERNAL (fo = 'x01') &amp;&amp; tag == VARTAG_ONDISK (x12)
  -- rawsize - VARHDRSZ &gt; extsize
  WHEN (fo = 'x01') AND (tag = 'x12') AND (osize - 4 &gt; ssize) THEN 'toasted (compressed)'
-- rawsize - VARHDRSZ &lt;= extsize
  WHEN (fo = 'x01') AND (tag = 'x12') AND (osize - 4 &lt;= ssize) THEN 'toasted (uncompressed)'
  -- VARATT_IS_EXTERNAL_INDIRECT: VARATT_IS_EXTERNAL &amp;&amp; tag == VARTAG_INDIRECT (x01)
  WHEN (fo = 'x01') AND (tag = 'x01') then 'indirect in-memory'
  -- VARATT_IS_EXTERNAL_EXPANDED: VARATT_IS_EXTERNAL &amp;&amp; VARTAG_IS_EXPANDED(VARTAG_EXTERNAL)
  WHEN (fo = 'x01') AND (tag = 'x02' OR tag = 'x03') then 'expanded in-memory'
  -- VARATT_IS_SHORT (va_header &amp; 0x01) == 0x01)
  WHEN (fo &amp; 'x01' = 'x01') THEN 'short in-line'
  -- VARATT_IS_COMPRESSED (va_header &amp; 0x03) == 0x02)
  WHEN (fo &amp; 'x03' = 'x02') THEN 'long in-line (compressed)'
  ELSE 'long in-line (uncompressed)'
END as toast_info
from
(
select
page_items.t_ctid,
substr(page_items.t_attrs[1]::text,2,3)::bit(8) as fo,
('x'||substr(page_items.t_attrs[1]::text,5,2))::bit(8) as tag,
('x'||regexp_replace(substr(page_items.t_attrs[1]::text,7,8),'(\w\w)(\w\w)(\w\w)(\w\w)','\4\3\2\1'))::bit(32)::int as osize ,
('x'||regexp_replace(substr(page_items.t_attrs[1]::text,15,8),'(\w\w)(\w\w)(\w\w)(\w\w)','\4\3\2\1'))::bit(32)::int as ssize
from
generate_series(0, pg_relation_size('bdttoast'::regclass::text) / 8192 - 1) blkno ,
heap_page_item_attrs(get_raw_page('bdttoast',blkno::int), 'bdttoast'::regclass) as page_items
) as hp;
</pre>
<p>As you can see, the query focus on the bdttoast table. Let's create this table and put some data in it to see the query in action.</p>
<h3>Let's see the query in action</h3>
<p>Let's create this table:</p>
<pre style="padding-left:40px;">postgres=# CREATE TABLE bdttoast ( message text );
CREATE TABLE
</pre>
<p>the message field storage type is "extended":</p>
<pre style="padding-left:40px;">postgres=# \d+ bdttoast
                                 Table "public.bdttoast"
 Column  | Type | Collation | Nullable | Default | Storage  | Stats target | Description
---------+------+-----------+----------+---------+----------+--------------+-------------
 message | text |           |          |         | extended |              |</pre>
<p>extended allows both compression and out-of-line storage. This is the default for most <acronym class="acronym">TOAST</acronym>-able data types. Compression will be attempted first, then out-of-line storage if the row is still too big.</p>
<p>let's add one tuple:</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES ('default');
INSERT 0 1</pre>
<p>and check how the message field has been stored thanks to the query:</p>
<pre style="padding-left:40px;">postgres=# \i toast_info.sql
 t_ctid |  toast_info
--------+---------------
 <strong>(0,1)</strong>  | short in-line
(1 row)</pre>
<p>as you can see it has been stored in-line.</p>
<p>Add another tuple with more data in the message field, and check its storage information:</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES (repeat('a',10000));
INSERT 0 1
postgres=# \i toast_info.sql
 t_ctid |        toast_info
--------+---------------------------
 (0,1)  | short in-line
 <strong>(0,2)</strong>  | long in-line (compressed)
(2 rows)</pre>
<p>as you can see this field value has been stored as long in-line and is compressed.</p>
<p>Add another tuple with even more data in the message field, and check its storage information:</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES (repeat('b',1000000));
INSERT 0 1
postgres=# \i toast_info.sql
 t_ctid |        toast_info
--------+---------------------------
 (0,1)  | short in-line
 (0,2)  | long in-line (compressed)
 <strong>(0,3)</strong>  | toasted (compressed)
(3 rows)</pre>
<p>this time it has been stored as TOAST-ed and is compressed.</p>
<p>Let's change the message column storage to external:</p>
<pre style="padding-left:40px;">postgres=# ALTER TABLE bdttoast ALTER COLUMN message SET STORAGE EXTERNAL;
ALTER TABLE
postgres=# \d+ bdttoast
                                 Table "public.bdttoast"
 Column  | Type | Collation | Nullable | Default | Storage  | Stats target | Description
---------+------+-----------+----------+---------+----------+--------------+-------------
 message | text |           |          |         | external |              |</pre>
<p>external means it allows out-of-line storage but not compression.</p>
<p>Now, let's add 3 tuples with the same field message size as the 3 ones previously added and compare the storage information.</p>
<p>Let's add one tuple with the same message size as the one previously stored as "short in-line":</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES ('externa');
INSERT 0 1
postgres=# \i toast_info.sql
 t_ctid |        toast_info
--------+---------------------------
 (0,1)  | short in-line
 (0,2)  | long in-line (compressed)
 (0,3)  | toasted (compressed)
 <strong>(0,4)</strong>  | short in-line
(4 rows)</pre>
<p>this one is still short in-line (same as t_ctid (0,1)).</p>
<p>Let's add one tuple with the same message size as the one previously stored as "long in-line (compressed)":</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES (repeat('c',10000));
INSERT 0 1
postgres=# \i toast_info.sql
 t_ctid |        toast_info
--------+---------------------------
 (0,1)  | short in-line
 (0,2)  | long in-line (compressed)
 (0,3)  | toasted (compressed)
 (0,4)  | short in-line
 <strong>(0,5)</strong>  | toasted (uncompressed)
(5 rows)</pre>
<p>this one is TOAST-ed and uncompressed with storage external (as compare with t_ctid (0,2) with storage extended).</p>
<p>Let's add one tuple with the same message size as the one previously stored as "toasted (compressed)":</p>
<pre style="padding-left:40px;">postgres=# INSERT INTO bdttoast VALUES (repeat('d',1000000));
INSERT 0 1
postgres=# \i toast_info.sql
 t_ctid |        toast_info
--------+---------------------------
(0,1)   | short in-line
(0,2)   | long in-line (compressed)
(0,3)   | toasted (compressed)
(0,4)   | short in-line
(0,5)   | toasted (uncompressed)
<strong>(0,6)</strong>   | toasted (uncompressed)
(6 rows)</pre>

this one is TOAST-ed and uncompressed with storage external (as compare with t\_ctid (0,3) with storage extended).

### Remarks

- use this query on little-endian machines only (bit layouts would not be the same on big-endian and would impact the query accuracy)
- t\_attrs[1] is used in the query to retrieve the information. This is because the message field is the 1st of the relation

### Conclusion

Thanks to the pageinspect extension we have been able to write a query to retrieve variable-length storage information. We have been able to compare how our data has been stored depending on the column storage being used (extended or external).


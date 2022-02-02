---
layout: post
title: Compare a PostgreSQL block from memory and from file
date: 2022-02-01 12:26:32.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Postgresql
tags: [PostgreSQL]
author:
  login: bdrouvot
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2022/02/01-compare-a-postgresql-block-from-memory-and-file"
---

### Introduction

This post shows a way to compare the content of a PostgreSQL block from memory and from its related file.
To achieve this goal we will use the [pageinspect](https://www.postgresql.org/docs/current/pageinspect.html) extension and [pg_filedump](https://github.com/df7cb/pg_filedump). 

### Let's compare

#### Find a dirty block

First of all, for this example, I want to be sure that I will be comparing a dirty block (means its content between the shared buffer and the file it belongs to differs).

A dirty block is a block that has been modified since the last checkpoint.

-   Let's use the [pg_buffercache](https://www.postgresql.org/docs/current/pgbuffercache.html) extension to find a dirty block that belongs to the relation bdttab:

	postgres=# select c.relname, b.relblocknumber, b.isdirty from pg_buffercache AS b, pg_class AS c where c.relfilenode = b.relfilenode and c.relname = 'bdttab' and isdirty is true;
	 relname | relblocknumber | isdirty
	---------+----------------+---------
	 bdttab  |              1 | t
	(1 row)

So the block number 1 is a dirty block, let's compare its content from its file and memory.

#### Get the content from the relation file:

To do so, let's:

-   get its associated filepath that way:

	postgres=# SELECT pg_relation_filepath('bdttab');
	 pg_relation_filepath
	----------------------
	 base/13580/32771

-   use pg_filedump that way:

	$ ./pg_filedump -R 1 /home/postgres/pg/pg_installed/data/base/13580/32771

	*******************************************************************
	* PostgreSQL File/Block Formatted Dump Utility
	*
	* File: /home/postgres/pg/pg_installed/data/base/13580/32771
	* Options used: -R 1
	*******************************************************************

	Block    1 ********************************************************
	<Header> -----
	 Block Offset: 0x00002000         Offsets: Lower      84 (0x0054)
	 Block: Size 8192  Version    4            Upper    7592 (0x1da8)
	 LSN:  logid      0 recoff 0x01951ae0      Special  8192 (0x2000)
	 Items:   15                      Free Space: 7508
	 Checksum: 0xc042  Prune XID: 0x00000000  Flags: 0x0000 ()
	 Length (including item array): 84

	<Data> -----
	 Item   1 -- Length:   40  Offset: 8152 (0x1fd8)  Flags: NORMAL
	 Item   2 -- Length:   40  Offset: 8112 (0x1fb0)  Flags: NORMAL
	 Item   3 -- Length:   40  Offset: 8072 (0x1f88)  Flags: NORMAL
	 Item   4 -- Length:   40  Offset: 8032 (0x1f60)  Flags: NORMAL
	 Item   5 -- Length:   40  Offset: 7992 (0x1f38)  Flags: NORMAL
	 Item   6 -- Length:   40  Offset: 7952 (0x1f10)  Flags: NORMAL
	 Item   7 -- Length:   40  Offset: 7912 (0x1ee8)  Flags: NORMAL
	 Item   8 -- Length:   40  Offset: 7872 (0x1ec0)  Flags: NORMAL
	 Item   9 -- Length:   40  Offset: 7832 (0x1e98)  Flags: NORMAL
	 Item  10 -- Length:   40  Offset: 7792 (0x1e70)  Flags: NORMAL
	 Item  11 -- Length:   40  Offset: 7752 (0x1e48)  Flags: NORMAL
	 Item  12 -- Length:   40  Offset: 7712 (0x1e20)  Flags: NORMAL
	 Item  13 -- Length:   40  Offset: 7672 (0x1df8)  Flags: NORMAL
	 Item  14 -- Length:   40  Offset: 7632 (0x1dd0)  Flags: NORMAL
	 Item  15 -- Length:   40  Offset: 7592 (0x1da8)  Flags: NORMAL

	*** End of Requested Range Encountered. Last Block Read: 1 ***

Now let's get the information from memory:

#### Get the content from memory:

-   First, let's dump the block from memory to a file thanks to pageinspect that way:

	$ psql postgres -tA -c "select encode(get_raw_page::bytea, 'hex') from get_raw_page('bdttab',1)" | xxd -r -p > bdttab_block_1.out

	$ du -sh bdttab_block_1.out
	8.0K    bdttab_block_1.out

As you can see the generated file size is 8K, which is the default PostgreSQL block size (that i did not change).
 
-   and then use the dumped block as the pg_filedump input that way:

	$ ./pg_filedump ./bdttab_block_1.out

	*******************************************************************
	* PostgreSQL File/Block Formatted Dump Utility
	*
	* File: ./bdttab_block_1.out
	* Options used: None
	*******************************************************************

	Block    0 ********************************************************
	<Header> -----
	 Block Offset: 0x00000000         Offsets: Lower      88 (0x0058)
	 Block: Size 8192  Version    4            Upper    7552 (0x1d80)
	 LSN:  logid      0 recoff 0x0197c9b0      Special  8192 (0x2000)
	 Items:   16                      Free Space: 7464
	 Checksum: 0xc042  Prune XID: 0x00000000  Flags: 0x0000 ()
	 Length (including item array): 88

	<Data> -----
	 Item   1 -- Length:   40  Offset: 8152 (0x1fd8)  Flags: NORMAL
	 Item   2 -- Length:   40  Offset: 8112 (0x1fb0)  Flags: NORMAL
	 Item   3 -- Length:   40  Offset: 8072 (0x1f88)  Flags: NORMAL
	 Item   4 -- Length:   40  Offset: 8032 (0x1f60)  Flags: NORMAL
	 Item   5 -- Length:   40  Offset: 7992 (0x1f38)  Flags: NORMAL
	 Item   6 -- Length:   40  Offset: 7952 (0x1f10)  Flags: NORMAL
	 Item   7 -- Length:   40  Offset: 7912 (0x1ee8)  Flags: NORMAL
	 Item   8 -- Length:   40  Offset: 7872 (0x1ec0)  Flags: NORMAL
	 Item   9 -- Length:   40  Offset: 7832 (0x1e98)  Flags: NORMAL
	 Item  10 -- Length:   40  Offset: 7792 (0x1e70)  Flags: NORMAL
	 Item  11 -- Length:   40  Offset: 7752 (0x1e48)  Flags: NORMAL
	 Item  12 -- Length:   40  Offset: 7712 (0x1e20)  Flags: NORMAL
	 Item  13 -- Length:   40  Offset: 7672 (0x1df8)  Flags: NORMAL
	 Item  14 -- Length:   40  Offset: 7632 (0x1dd0)  Flags: NORMAL
	 Item  15 -- Length:   40  Offset: 7592 (0x1da8)  Flags: NORMAL
	 Item  16 -- Length:   40  Offset: 7552 (0x1d80)  Flags: NORMAL

	*** End of File Encountered. Last Block Read: 0 ***

As you can see the blocks are different (among other things, there is one more item in memory).

### Conclusion

Thanks to pageinspect and pg_filedump we have been able to compare a block content from memory and from the file it belongs to.

---
layout: post
title: Flipping a bit one by one in a PostgreSQL block looking for a checksum
date: 2020-08-08 12:26:32.000000000 +01:00
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
permalink: "/2020/08/08/flipping-a-bit-one-by-one-in-a-postgres-block-looking-for-a-checksum/"
---

### Introduction

Nikolay Samokhvalov started recently a thread for collecting useful ideas, tools for dealing with PostgreSQL DATA CORRUPTION, and BUGS (see the [thread](https://twitter.com/samokhvalov/status/1289069826531422208)).

One potential corruption could be a bit flip, let me share one utility that could starting the investigation in such a case.

say you got:

     postgres=# select * from  bdt;
     WARNING:  page verification failed, calculated checksum 20317 but expected 51845
     ERROR:  invalid page in block 0 of relation base/13287/24877

then, copy the block:

     $ dd status=none bs=8192 count=1 if=/usr/local/pgsql11.8-last/data/base/13287/24877 skip=0 of=./for_bit_flip_investigation

launch the `flip_bit_and_checksum.bin` utility to look for the expected checksum (51845 in this example):

     $ ./flip_bit_and_checksum.bin
     ./flip_bit_and_checksum.bin: Flip one bit one by one and compute the checksum.
     ./flip_bit_and_checksum.bin: The bit that has been flipped is displayed if the computed checksum matches the one in argument.

     Usage:
     ./flip_bit_and_checksum.bin [OPTION] <block_path>
     -c, --checksum=CHECKSUM to look for

     $ ./flip_bit_and_checksum.bin ./for_bit_flip_investigation -c `51845`
     Warning: Keep in mind that numbering starts from 0 for both bit and byte
     checksum ca85 (51845) found while flipping bit 1926 (bit 6 in byte 240)

So, by flipping bit 1926 the expected checksum is returned.  
It's an indication that the corruption might be due to a bit flip at that position, that's a good start for deeper investigations.

### Remarks

-   The `flip_bit_and_checksum.bin` utility can be found [here](https://github.com/bdrouvot/pg_toolkit/blob/master/c/flip_bit_and_checksum.c).
-   Having found the expected cheksum while flipping a bit is not a guarantee that a flip bip actually happened and led to the corruption. But it's a good way to start the investigations with.
-   The utility does not modify the original block.

### Conclusion

Thanks to the utility we can look for a bit flip that could generate the expected checksum.

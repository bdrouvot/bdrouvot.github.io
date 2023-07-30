---
layout: post
title: '"Rusting" a new Postgres C extension that records subtransactions counters'
date: 2023-07-30 08:00:00.000000000 +01:00
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
permalink: "/2023/07/30/rusting-a-new-postgres-c-extension-that-records-subtransactions-counters/"
---

## Introduction

The purpose of this blog post is twofold:

- Firstly, to introduce a new Postgres C extension to record subtransactions counters
- Secondly, to share its Rust version

## Introduce the extension

The new extension, namely **pg_subxact_counters** can be found in this [repo](https://github.com/bdrouvot/pg_subxact_counters).

Subtransactions can lead to performance issue, as
mentioned in some posts, see for example:

- [Subtransactions and performance in PostgreSQL](https://www.cybertec-postgresql.com/en/subtransactions-and-performance-in-postgresql/)
- [PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)
- [Why we spent the last month eliminating PostgreSQL subtransactions](https://about.gitlab.com/blog/2021/09/29/why-we-spent-the-last-mo
nth-eliminating-postgresql-subtransactions/)

The purpose of this extension is to provide counters to
monitor the subtransactions (generation rate, overflow, state).  

One idea could be to sample the **pg_subxact_counters** view (coming with the extension) on a regular basis.

### Example:

```
postgres=# \dx+ pg_subxact_counters
Objects in extension "pg_subxact_counters"
       Object description
--------------------------------
 function pg_subxact_counters()
 view pg_subxact_counters
(2 rows)

postgres=# select * from pg_subxact_counters;
 subxact_start | subxact_commit | subxact_abort | subxact_overflow
---------------+----------------+---------------+------------------
             0 |              0 |             0 |                0
(1 row)

postgres=# begin;
BEGIN
postgres=*# savepoint a;
SAVEPOINT
postgres=*# savepoint b;
SAVEPOINT
postgres=*# commit;
COMMIT

postgres=# select * from pg_subxact_counters;
 subxact_start | subxact_commit | subxact_abort | subxact_overflow
---------------+----------------+---------------+------------------
             2 |              2 |             0 |                0
(1 row)
```

### Fields meaning

* subxact_start: number of substransactions that started
* subxact_commit: number of substransactions that committed
* subxact_abort: number of substransactions that aborted (rolled back)
* subxact_overflow: number of times a top level XID have had substransactions overflowed

### Remarks

* subxact_start - subxact_commit - subxact_abort: number of subtransactions that have started and not yet committed/rolled back.
* subxact_overflow does not represent the number of substransactions that exceeded PGPROC_MAX_CACHED_SUBXIDS.
* the counters are global, means they record the activity for all the databases in the instance.

## Sharing a Rust version

As you can see the [repo](https://github.com/bdrouvot/pg_subxact_counters) is made of two subdirectories:

- c
- rust

The extension has been initially written in C.

While doing so, and as I'm in my journey of learning Rust, I thought it would
be a great learning experience to also write it in Rust.

### Why?

Why Rust?  
Rust is very popular and it also provides a lot of existing modules/crates.  
One could imagine using some of those existing modules/crates to build Postgres extensions.

Why providing C and Rust code?  
For people that are:
- in their journey of learning to write Postgres extensions in Rust
- used to write C extensions

having a simple example (as this one) written in both languages might help.

### Remark

The Rust part relies on [pgrx](https://github.com/pgcentralfoundation/pgrx): this is an awesome framework for developing PostgreSQL extensions in Rust.

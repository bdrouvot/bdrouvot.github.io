--- layout: post title: Retrieve PostgreSQL variable-length storage information thanks to pageinspect date: 2020-01-18 13:49:47.000000000 +01:00 type: post parent\_id: '0' published: true password: '' status: publish categories: - Postgresql tags: \[\] meta: \_edit\_last: '40807211' geo\_public: '0' \_wpas\_skip\_7950430: '1' timeline\_notification: '1579351791' \_publicize\_job\_id: '39682411733' \_publicize\_done\_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:62:"https://twitter.com/BertrandDrouvot/status/1218515849352548358";}} \_publicize\_done\_2558296: '1' \_wpas\_done\_2225791: '1' publicize\_twitter\_user: BertrandDrouvot publicize\_linkedin\_url: '' \_publicize\_done\_22399261: '1' \_wpas\_done\_23399164: '1' author: login: bdrouvot email: bdtoracleblog@gmail.com display\_name: bdrouvot first\_name: '' last\_name: '' permalink: "/2020/01/18/retrieve-postgresql-variable-length-storage-information-thanks-to-pageinspect/" ---

### Introduction

In PostgreSQL a variable-length datatype value can be stored in-line or out-of-line (as a TOAST). It can also be compressed or not (see the [documentation](https://www.postgresql.org/docs/current/storage-toast.html) for more details).

Let's make use of the [pageinspect](https://www.postgresql.org/docs/current/pageinspect.html) extension and the information about variable-length datatype found in [postgres.h](https://github.com/postgres/postgres/blob/master/src/include/postgres.h) to build a query to retrieve tuples variable-length storage information.

### The query

The query is the following:

    $ cat toast_info.sql
    select
    t_ctid,
    -- See postgres.h
    CASE
      WHEN (fo is NULL) THEN 'null'
      -- VARATT_IS_EXTERNAL_ONDISK: VARATT_IS_EXTERNAL (fo = 'x01') && tag == VARTAG_ONDISK (x12)
      -- rawsize - VARHDRSZ > extsize
      WHEN (fo = 'x01') AND (tag = 'x12') AND (osize - 4 > ssize) THEN 'toasted (compressed)'
    -- rawsize - VARHDRSZ <= extsize
      WHEN (fo = 'x01') AND (tag = 'x12') AND (osize - 4 <= ssize) THEN 'toasted (uncompressed)'
      -- VARATT_IS_EXTERNAL_INDIRECT: VARATT_IS_EXTERNAL && tag == VARTAG_INDIRECT (x01)
      WHEN (fo = 'x01') AND (tag = 'x01') then 'indirect in-memory'
      -- VARATT_IS_EXTERNAL_EXPANDED: VARATT_IS_EXTERNAL && VARTAG_IS_EXPANDED(VARTAG_EXTERNAL)
      WHEN (fo = 'x01') AND (tag = 'x02' OR tag = 'x03') then 'expanded in-memory'
      -- VARATT_IS_SHORT (va_header & 0x01) == 0x01)
      WHEN (fo & 'x01' = 'x01') THEN 'short in-line'
      -- VARATT_IS_COMPRESSED (va_header & 0x03) == 0x02)
      WHEN (fo & 'x03' = 'x02') THEN 'long in-line (compressed)'
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

As you can see, the query focus on the bdttoast table. Let's create this table and put some data in it to see the query in action.

### Let's see the query in action

Let's create this table:

    postgres=# CREATE TABLE bdttoast ( message text );
    CREATE TABLE

the message field storage type is "extended":

    postgres=# \d+ bdttoast
                                     Table "public.bdttoast"
     Column  | Type | Collation | Nullable | Default | Storage  | Stats target | Description
    ---------+------+-----------+----------+---------+----------+--------------+-------------
     message | text |           |          |         | extended |              |

extended allows both compression and out-of-line storage. This is the default for most TOAST-able data types. Compression will be attempted first, then out-of-line storage if the row is still too big.

let's add one tuple:

    postgres=# INSERT INTO bdttoast VALUES ('default');
    INSERT 0 1

and check how the message field has been stored thanks to the query:

    postgres=# \i toast_info.sql
     t_ctid |  toast_info
    --------+---------------
     (0,1)  | short in-line
    (1 row)

as you can see it has been stored in-line.

Add another tuple with more data in the message field, and check its storage information:

    postgres=# INSERT INTO bdttoast VALUES (repeat('a',10000));
    INSERT 0 1
    postgres=# \i toast_info.sql
     t_ctid |        toast_info
    --------+---------------------------
     (0,1)  | short in-line
     (0,2)  | long in-line (compressed)
    (2 rows)

as you can see this field value has been stored as long in-line and is compressed.

Add another tuple with even more data in the message field, and check its storage information:

    postgres=# INSERT INTO bdttoast VALUES (repeat('b',1000000));
    INSERT 0 1
    postgres=# \i toast_info.sql
     t_ctid |        toast_info
    --------+---------------------------
     (0,1)  | short in-line
     (0,2)  | long in-line (compressed)
     (0,3)  | toasted (compressed)
    (3 rows)

this time it has been stored as TOAST-ed and is compressed.

Let's change the message column storage to external:

    postgres=# ALTER TABLE bdttoast ALTER COLUMN message SET STORAGE EXTERNAL;
    ALTER TABLE
    postgres=# \d+ bdttoast
                                     Table "public.bdttoast"
     Column  | Type | Collation | Nullable | Default | Storage  | Stats target | Description
    ---------+------+-----------+----------+---------+----------+--------------+-------------
     message | text |           |          |         | external |              |

external means it allows out-of-line storage but not compression.

Now, let's add 3 tuples with the same field message size as the 3 ones previously added and compare the storage information.

Let's add one tuple with the same message size as the one previously stored as "short in-line":

    postgres=# INSERT INTO bdttoast VALUES ('externa');
    INSERT 0 1
    postgres=# \i toast_info.sql
     t_ctid |        toast_info
    --------+---------------------------
     (0,1)  | short in-line
     (0,2)  | long in-line (compressed)
     (0,3)  | toasted (compressed)
     (0,4)  | short in-line
    (4 rows)

this one is still short in-line (same as t\_ctid (0,1)).

Let's add one tuple with the same message size as the one previously stored as "long in-line (compressed)":

    postgres=# INSERT INTO bdttoast VALUES (repeat('c',10000));
    INSERT 0 1
    postgres=# \i toast_info.sql
     t_ctid |        toast_info
    --------+---------------------------
     (0,1)  | short in-line
     (0,2)  | long in-line (compressed)
     (0,3)  | toasted (compressed)
     (0,4)  | short in-line
     (0,5)  | toasted (uncompressed)
    (5 rows)

this one is TOAST-ed and uncompressed with storage external (as compare with t\_ctid (0,2) with storage extended).

Let's add one tuple with the same message size as the one previously stored as "toasted (compressed)":

    postgres=# INSERT INTO bdttoast VALUES (repeat('d',1000000));
    INSERT 0 1
    postgres=# \i toast_info.sql
     t_ctid |        toast_info
    --------+---------------------------
     (0,1)  | short in-line
     (0,2)  | long in-line (compressed)
     (0,3)  | toasted (compressed)
     (0,4)  | short in-line
     (0,5)  | toasted (uncompressed)
     (0,6)  | toasted (uncompressed)
    (6 rows)

this one is TOAST-ed and uncompressed with storage external (as compare with t\_ctid (0,3) with storage extended).

### Remarks

-   use this query on little-endian machines only (bit layouts would not be the same on big-endian and would impact the query accuracy)
-   t\_attrs\[1\] is used in the query to retrieve the information. This is because the message field is the 1st of the relation

### Conclusion

Thanks to the pageinspect extension we have been able to write a query to retrieve variable-length storage information. We have been able to compare how our data has been stored depending on the column storage being used (extended or external).

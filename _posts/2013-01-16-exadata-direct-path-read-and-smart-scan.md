---
layout: post
title: 'Exadata : Direct path read and smart scan'
date: 2013-01-16 21:43:40.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
tags:
- exadata
meta:
  _edit_last: '40807211'
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:40;}s:2:"wp";a:1:{i:0;i:11;}}
  geo_public: '0'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
  _publicize_job_id: '14556444312'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/01/16/exadata-direct-path-read-and-smart-scan/"
---
To write this post I took my inspiration mainly from 2 sources:

1. The [Expert Oracle Exadata Book](http://www.expertoracleexadata.com)
2. Frits Hoogland's blog&nbsp;[post](http://fritshoogland.wordpress.com/2013/01/04/oracle-11-2-and-the-direct-path-read-event)

The book (as well as the oracle university course) states that Smart Scan prerequisites are :

- There must be a full scan of an object.
- The scan must use Oracle’s Direct Path Read mechanism.
- The object must be stored on Exadata storage.

and the explanation is:

There is a simple explanation as to why these requirements exist. Oracle is a C program. The&nbsp;function that performs Smart Scans (kcfis\_read) is called by the direct path read function ( **kcbldrget** ),&nbsp;which is called by one of the full scan functions. It’s that simple. You can’t get to the **kcfis\_read** function&nbsp;without traversing the code path from full scan to direct read. And of course, the storage will have to be&nbsp;running Oracle’s software in order to process Smart Scans.

Here we are: &nbsp;As oracle is a C program, I can try to figure out by my own what's going on during Smart Scan thanks to the gdb debugger (Of course **I have no doubt about the accuracy of the explanation** mentioned above, the exercise is just for fun :-) )

To do this, I will use:

- &nbsp;One session with the trace 10046 enabled. This session is running a query that will produce the "cell smart table scan" wait event.
- &nbsp;One gdb session attached on the foreground process mentioned above (gdb -p \<pid\>)

First let's create a break point on the write() function to trap the stack when the session is writing the "cell smart table scan" into the 10046 trace file.

```
(gdb) break write
```

Let's run the query : The backtrace of all stack frames is the following when the "cell smart table scan" wait event has been written for the first time into the trace file.

```
#13 0x0ddb94a1 in kcfis\_reap\_ioctl () #14 0x0ddb8204 in kcfis\_push () #15 0x0ddcc01a in kcfis\_read () #16 0x0c800afc in kcbl\_predpush\_get () #17 0x0898c40c in kcbldrget () #18 0x0fadc5d2 in kcbgtcr ()
```

So, it looks like we are in the right direction, as we can see the **kcbldrget** and the **kcfis\_read** functions into the frame.

We now just have to check which function is responsible of the "cell smart table scan" wait event.

To do so let's put break points on the functions we discovered above that is to say:

```
(gdb) break **kcbldrget** Breakpoint 1 at 0x898b103 (gdb) break **kcbl\_predpush\_get** Breakpoint 2 at 0xc800a78 (gdb) break **kcfis\_read** Breakpoint 3 at 0xddcb371 (gdb) break **kcfis\_push** Breakpoint 4 at 0xddb78c0
```

and let's continue:

```
(gdb) c Continuing.
```

Now re-launch the sql.

First it breaks on:

```
Breakpoint 1, 0x0898b103 in kcbldrget ()
```

As no wait event "cell smart table scan" has been see so far into the trace file we can continue.

```
(gdb) c Continuing.
```

Now it breaks on:

```
Breakpoint 2, 0x0c800a78 in kcbl\_predpush\_get ()
```

As no wait event "cell smart table scan" has been see so far into the trace file we can continue.

```
(gdb) c Continuing.
```

Now it breaks on:

```
Breakpoint 3, 0x0ddcb371 in kcfis\_read ()
```

As no wait event "cell smart table scan" has been see so far into the trace file we can continue.

```
(gdb) c Continuing.
```

Now it breaks on :

```
Breakpoint 4, 0x0ddb78c0 in kcfis\_push ()
```

Here we are: The **wait event "cell smart table scan" appears** into the trace file.

So the event comes from the function that has been called before the kcfis\_push one. Let's display the calling tree to figure out &nbsp;which one it is:

```
Breakpoint 4, 0x0ddb78c0 in kcfis\_push () (gdb) up #1 &nbsp;0x0ddcc01a in kcfis\_read () (gdb) up #2 &nbsp;0x0c800afc in kcbl\_predpush\_get () (gdb) up #3 &nbsp;0x0898c40c in kcbldrget ()
```

So it's the **kcfis\_read** function that produced the wait event "cell smart table scan". Furthermore as we can see into the calling tree the **kcfis\_read** function has been called by the **kcbldrget** one.

**Conclusion:**

The Smart Scan has been launched from the direct read code path.

But a question still remains: How can I check by myself ( **again just for fun** ) that the **kcfis\_read** function (which is responsible of the smart scan) **can not be called outside the direct read code path**? For the moment I have no clue on this :-)

**Update 2015/09/08:**

As of 12.1.0.1, a more convenient way to display the stack linked to a wait event is:

[code language="sql"] exec DBMS\_MONITOR.SESSION\_TRACE\_ENABLE(waits =\> true, binds =\> false, plan\_stat =\> 'NEVER');

alter session set events 'wait\_event["cell smart table scan"] trace("from %s\n",shortstack())'; [/code]

Enabling the session trace is not mandatory but is useful as it displays the wait event prior to the stack.  
The wait event "cell smart table scan" has been used as an example (as it is the one of interest for this post), but you could use the one of your choice.


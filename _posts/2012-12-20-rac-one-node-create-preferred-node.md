---
layout: post
title: 'Rac One Node : Create preferred node'
date: 2012-12-20 09:23:39.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Rac
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:19;}s:2:"wp";a:1:{i:0;i:8;}}
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/20/rac-one-node-create-preferred-node/"
---
In my previous [Rac One Node post](http://bdrouvot.wordpress.com/2012/12/13/rac-one-node-avoid-automatic-database-relocation/ "RAC One Node : Avoid automatic database relocation") I gave a way to "stick" one Rac One Node database to a particular node of your cluster so that :

1. It should re-start automatically on the "stick" node in case of database or node crash.
2. It does not relocate automatically on other nodes.

In this post I'll give a way to create a "preferred" node for a Rac One Node database so that:

1. It should re-start and relocate automatically on the "preferred" node as soon as it is available.
2. It relocates automatically on other nodes in case the "preferred" node crash.

Let's go, first modify the SERVER\_NAMES attribute of the associated server pool to keep only the "preferred" node:

To which pool belongs the db ?

```
crsctl status resource ora.bdto.db -p | grep -i pool SERVER\_POOLS=ora.BDTO
```

Which servers are part of this pool ?

```
crsctl status serverpool ora.BDTO -p | grep -i server SERVER\_NAMES=bdtnode1 bdtnode2
```

Modify the server pool to define the “preferred” node for the database (bdtnode2 for example):

```
crsctl modify serverpool ora.BDTO -attr SERVER\_NAMES="bdtnode2" crsctl status serverpool ora.BDTO -p | grep -i server SERVER\_NAMES=bdtnode2
```

Second step is to change the PLACEMENT attribute of the database.

Change the PLACEMENT attribute from restricted to favored:

```
crsctl status resource ora.bdto.db -p | grep -i place ACTIVE\_PLACEMENT=1 PLACEMENT=restricted crsctl modify resource ora.bdto.db -attr PLACEMENT="favored" crsctl status resource ora.bdto.db -p | grep -i place ACTIVE\_PLACEMENT=1 PLACEMENT=favored
```

Now the BDTO RAC One Node database:

- will be re-started automatically on node bdtnode1 in case the "preferred" node bdtnode2 crash.
- will be re-started and relocated automatically on the "preferred" node bdtnode2 as soon as it will be again available.

Remarks:

Be&nbsp;careful:&nbsp;The database will be relocated&nbsp;automatically on the "preferred" node as soon as it is up (Even if the database is currently running on other node).

If you try to relocate manually the database on the “non&nbsp;preferred” node:

```
srvctl relocate database -d BDTO -n bdtnode1
```

It will be done and the node bdtnode1 will be again member of the server pool (as a consequence the node "bdtnode2" is not anymore the "preferred" one).

**Conclusion:** &nbsp;You have the choice,

- Stick the database to one node so that it does not relocate&nbsp;automatically on surviving nodes ([This post](http://bdrouvot.wordpress.com/2012/12/13/rac-one-node-avoid-automatic-database-relocation/ "RAC One Node : Avoid automatic database relocation")).
- Defined a "preferred" node and allow the database to relocate&nbsp;automatically on surviving nodes and on the "preferred" node as soon as it is available (The current post).

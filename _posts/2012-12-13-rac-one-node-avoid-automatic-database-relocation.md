---
layout: post
title: 'Rac One Node : Avoid automatic database relocation'
date: 2012-12-13 21:30:40.000000000 +01:00
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
  _wpas_done_2077996: '1'
  publicize_twitter_user: BertrandDrouvot
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  publicize_reach: a:2:{s:7:"twitter";a:1:{i:2225791;i:8;}s:2:"wp";a:1:{i:0;i:5;}}
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2012/12/13/rac-one-node-avoid-automatic-database-relocation/"
---

Imagine you need to ensure that <span style="color:#0000ff;">one of your many</span> <span style="color:#0000ff;">RAC One Node</span> database:

1.  Should re-start automatically on the current node in case of database or node crash.
2.  Does not relocate automatically on other nodes.

So, you need somehow to "<span style="color:#0000ff;">stick</span>" this particular RAC One Node database to one node of your cluster.

For a good understanding of what RAC One Node is, you can have a look to [Martin's post](http://martincarstenbach.wordpress.com/2011/02/16/rac-one-node-and-database-protection/).

Disabling the database will not help as it will not satisfy the second point mentioned above.

To answer this need, I'll modify one <span style="color:#0000ff;">property of the database's server pool</span> (database is administrator managed):

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">To which pool belongs the db ?</span></span>

    crsctl status resource ora.BDTO.db -p | grep -i pool 
    SERVER_POOLS=ora.BDTO

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Which servers are part of this pool ?</span></span>

    crsctl status serverpool ora.BDTO -p | grep -i server
    SERVER_NAMES=bdtnode1 bdtnode2

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Modify the server pool to "stick" the database to one node (bdtnode2 for example):</span></span>

    crsctl modify serverpool ora.BDTO -attr SERVER_NAMES="bdtnode2"
    crsctl status serverpool ora.BDTO -p | grep -i server
    SERVER_NAMES=bdtnode2

<span style="text-decoration:underline;color:#0000ff;">Now the BDTO RAC One Node database:</span>

-   will not be re-started automatically on node bdtnode1 in case bdtnode2 crash.
-   will be re-started automatically on node bdtnode2 as soon as it will be again available.

<span style="text-decoration:underline;"><span style="color:#0000ff;text-decoration:underline;">Remarks:</span></span>

1.  If the server pool "host" many databases, all of them will be affected by the change.
2.  If you try to <span style="color:#0000ff;">relocate manually</span> the database on the "excluded" node:

<!-- -->

    srvctl relocate database -d BDTO -n bdtnode1

It will be done and the "excluded" bdtnode1 will be <span style="color:#ff0000;">again member</span> of the server pool (as a consequence the database is <span style="color:#ff0000;">not sticked to bdtnode2 anymore</span>).

If you need another behaviour (create a preferred node), then you have to change the PLACEMENT attribute of the database ([see this post](http://bdrouvot.wordpress.com/2012/12/20/rac-one-node-create-preferred-node/ "Rac One Node : Create preferred node")).

**Conclusion:** The purpose of this post is not to explain why you could choose to stick one RAC One Node database to a particular node of your cluster, it just provide a way to do so :-)

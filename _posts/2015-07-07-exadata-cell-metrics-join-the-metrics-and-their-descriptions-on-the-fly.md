---
layout: post
title: 'Exadata cell metrics: Join the metrics and their descriptions on the fly'
date: 2015-07-07 17:19:02.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Exadata
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6024219886249664512&type=U&a=9cw1
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/6hY5z1hu5yy
  _publicize_job_id: '12482648234'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/618454198095474689";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/07/07/exadata-cell-metrics-join-the-metrics-and-their-descriptions-on-the-fly/"
---
## Introduction

Cells metrics are very useful but their name are not so friendly. The name is a concatenation of abbreviations for the type of component, delimited by the underscore character. Then you have to understand the naming convention to understand the meaning of the metric name.

For example, knowing that:

- CD stands for "Cell Disks metrics"
- IO\_BY stands for "Number of megabytes"
- R stands for Read
- LG stands for Large

You can conclude&nbsp;that the CD\_IO\_BY\_R\_LG metric is linked to the "Number of megabytes read in large blocks from a cell disk".

Hopefully the metrics are explained into the [Oracle Documentation](http://docs.oracle.com/cd/E50790_01/doc/doc.121/e50471/monitoring.htm#SAGUG20463)&nbsp;and you can also retrieve their description from cellcli:

```
$ cellcli -e "list metricdefinition attributes name,description where name='CD\_IO\_BY\_R\_LG'" CD\_IO\_BY\_R\_LG "Number of megabytes read in large blocks from a cell disk"
```

## Lack of description

That said, as an example, let's query the current metric values for a particular&nbsp;database that way:

```
$ cellcli -e "list metriccurrent attributes metricObjectName,name,metricValue where name like 'DB.\*' and metricObjectName='BDT'" BDT DB\_FC\_IO\_BY\_SEC 0 MB/sec BDT DB\_FC\_IO\_RQ 47,638 IO requests BDT DB\_FC\_IO\_RQ\_SEC 2.1 IO/sec BDT DB\_FD\_IO\_BY\_SEC 0 MB/sec BDT DB\_FD\_IO\_LOAD 19,885 BDT DB\_FD\_IO\_RQ\_LG 36 IO requests BDT DB\_FD\_IO\_RQ\_LG\_SEC 0.0 IO/sec BDT DB\_FD\_IO\_RQ\_SM 47,602 IO requests BDT DB\_FD\_IO\_RQ\_SM\_SEC 2.1 IO/sec BDT DB\_FL\_IO\_BY 0.000 MB BDT DB\_FL\_IO\_BY\_SEC 0.000 MB/sec BDT DB\_FL\_IO\_RQ 0 IO requests BDT DB\_FL\_IO\_RQ\_SEC 0.0 IO/sec BDT DB\_IO\_BY\_SEC 0 MB/sec BDT DB\_IO\_LOAD 0.0 BDT DB\_IO\_RQ\_LG 0 IO requests BDT DB\_IO\_RQ\_LG\_SEC 0.0 IO/sec BDT DB\_IO\_RQ\_SM 19 IO requests BDT DB\_IO\_RQ\_SM\_SEC 0.0 IO/sec BDT DB\_IO\_UTIL\_LG 0 % BDT DB\_IO\_UTIL\_SM 0 % BDT DB\_IO\_WT\_LG 0 ms BDT DB\_IO\_WT\_LG\_RQ 0.0 ms/request BDT DB\_IO\_WT\_SM 0 ms BDT DB\_IO\_WT\_SM\_RQ 0.0 ms/request
```

As you can see the metric description is not there and there is no way to retrieve it from&nbsp;metriccurrent (or&nbsp;metrichistory) because this is not an attribute:

```
$ cellcli -e "describe metriccurrent" name alertState collectionTime metricObjectName metricType metricValue objectType $ cellcli -e "describe metrichistory" name alertState collectionTime metricObjectName metricType metricValue metricValueAvg metricValueMax metricValueMin objectType
```

But if you send&nbsp;the result of our example to&nbsp;someone that don't know (or don't remember) the naming convention (or if you are not 100% sure of the definition of a particular metric) then he/you'll have to:

- go back to the oracle documentation
- query the&nbsp;metricdefinition with cellcli

## New script:&nbsp;_exadata\_metrics\_desc.pl_

Thanks to the&nbsp;_exadata\_metrics\_desc.pl_ script, you can&nbsp;add (to the cellcli output) the description of the metric on the fly.

Let's launch the same query (as the one used in the previous example) and add a call to&nbsp;_exadata\_metrics\_desc.pl_ that way_:_

```
$ cellcli -e "list metriccurrent attributes metricObjectName,name,metricValue where name like 'DB.\*' and metricObjectName='BDT'" | ./exadata\_metrics\_desc.pl BDT DB\_FC\_IO\_BY\_SEC (Number of megabytes of I/O per second for this database to flash cache) 0 MB/sec BDT DB\_FC\_IO\_RQ (Number of IO requests issued by a database to flash cache) 48,123 IO requests BDT DB\_FC\_IO\_RQ\_SEC (Number of IO requests issued by a database to flash cache per second) 2.1 IO/sec BDT DB\_FD\_IO\_BY\_SEC (Number of megabytes of I/O per second for this database to flash disks) 0 MB/sec BDT DB\_FD\_IO\_LOAD (Average I/O load from this database for flash disks) 4,419 BDT DB\_FD\_IO\_RQ\_LG (Number of large IO requests issued by a database to flash disks) 36 IO requests BDT DB\_FD\_IO\_RQ\_LG\_SEC (Number of large IO requests issued by a database to flash disks per second) 0.0 IO/sec BDT DB\_FD\_IO\_RQ\_SM (Number of small IO requests issued by a database to flash disks) 48,087 IO requests BDT DB\_FD\_IO\_RQ\_SM\_SEC (Number of small IO requests issued by a database to flash disks per second) 2.1 IO/sec BDT DB\_FL\_IO\_BY (The number of MB written to the Flash Log) 0.000 MB BDT DB\_FL\_IO\_BY\_SEC (The number of MB written per second to the Flash Log) 0.000 MB/sec BDT DB\_FL\_IO\_RQ (The number of I/O requests issued to the Flash Log) 0 IO requests BDT DB\_FL\_IO\_RQ\_SEC (The number of I/O requests per second issued to the Flash Log) 0.0 IO/sec BDT DB\_IO\_BY\_SEC (Number of megabytes of I/O per second for this database to hard disks) 0 MB/sec BDT DB\_IO\_LOAD (Average I/O load from this database for hard disks) 0.0 BDT DB\_IO\_RQ\_LG (Number of large IO requests issued by a database to hard disks) 0 IO requests BDT DB\_IO\_RQ\_LG\_SEC (Number of large IO requests issued by a database to hard disks per second) 0.0 IO/sec BDT DB\_IO\_RQ\_SM (Number of small IO requests issued by a database to hard disks) 19 IO requests BDT DB\_IO\_RQ\_SM\_SEC (Number of small IO requests issued by a database to hard disks per second) 0.0 IO/sec BDT DB\_IO\_UTIL\_LG (Percentage of disk resources utilized by large requests from this database) 0 % BDT DB\_IO\_UTIL\_SM (Percentage of disk resources utilized by small requests from this database) 0 % BDT DB\_IO\_WT\_LG (IORM wait time for large IO requests issued by a database) 0 ms BDT DB\_IO\_WT\_LG\_RQ (Average IORM wait time per request for large IO requests issued by a database) 0.0 ms/request BDT DB\_IO\_WT\_SM (IORM wait time for small IO requests issued by a database) 0 ms BDT DB\_IO\_WT\_SM\_RQ (Average IORM wait time per request for small IO requests issued by a database) 0.0 ms/request
```

As you can see the description of each metric being part of the initial output has been added.

## Remarks

- You can download the script from&nbsp;[this repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)&nbsp;or from [GitHub](https://github.com/bdrouvot/exadata_metrics_desc).
- Feel free to build the query you want on the metrics. You just need to add a call to&nbsp;_exadata\_metrics\_desc.pl&nbsp;_to see the metric description being added on the fly (as long as the metric name appears in the output of your initial query).
- The idea of this script is all to credit to [Martin Bach](https://martincarstenbach.wordpress.com/).
- This script works with any input (could be a text file):

[![Screen Shot 2015-09-14 at 15.34.36]({{ site.baseurl }}/assets/images/screen-shot-2015-09-14-at-15-34-36.png?w=300)](https://bdrouvot.files.wordpress.com/2015/07/screen-shot-2015-09-14-at-15-34-36.png)

## Conclusion

The&nbsp;_exadata\_metrics\_desc.pl&nbsp;_can be used to join on the fly the metric name, its value (and whatever attribute you would love to see) with its&nbsp;associated description.


---
layout: post
title: Extract Exadata cells historical metrics in CSV format
date: 2015-07-01 16:40:29.000000000 +02:00
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
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/616270167383764992";}}
  _publicize_job_id: '12219434797'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6022035852769714176&type=U&a=XCZV
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/SV16sVw2kuh
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2015/07/01/extract-exadata-cells-historical-metrics-in-csv-format/"
---
<p>Exadata provides a lot of useful metrics to monitor the Cells and you may want to retrieve historical values for some metrics. To do so, you can use the "LIST METRICHISTORY" command through CellCLI on the cell.</p>
<p>But as usual, visualising the metrics is even more better. For this purpose, you can use a perl script (see the download link in the remarks section) that extracts the historical metrics in CSV format so that you can graph them with the visualisation tool of your choice.</p>
<p><span style="text-decoration:underline;"><strong>Let's see the help:</strong></span></p>
<pre style="padding-left:30px;">Usage: ./csv_exadata_metrics_history.pl [cell=|groupfile=] [serial] [type=] [name=] [objectname=] [name!=] [objectname!=] [ago_unit=] [ago_value=]

 Parameter                 Comment                                                      Default
 ---------                 -------                                                      -------
CELL= comma-separated list of cells GROUPFILE= file containing list of cells SERIAL serialize execution over the cells (default is no) TYPE= Metrics type to extract: Cumulative|Rate|Instantaneous ALL NAME= Metrics to extract (wildcard allowed) ALL OBJECTNAME= Objects to extract (wildcard allowed) ALL NAME!= Exclude metrics (wildcard allowed) EMPTY OBJECTNAME!= Exclude objects (wildcard allowed) EMPTY AGO\_UNIT= Unit to retrieve historical metrics back: day|hour|minute HOUR AGO\_VALUE= Value associated to Unit to retrieve historical metrics back 1 utility assumes passwordless SSH from this cell node to the other cell nodes utility assumes ORACLE\_HOME has been set (with celladmin user for example) Example : ./csv\_exadata\_metrics\_history.pl cell=cell Example : ./csv\_exadata\_metrics\_history.pl groupfile=./cell\_group Example : ./csv\_exadata\_metrics\_history.pl cell=cell objectname='CD\_disk03\_cell' Example : ./csv\_exadata\_metrics\_history.pl cell=cell name='.\*BY.\*' objectname='.\*disk.\*' Example : ./csv\_exadata\_metrics\_history.pl cell=enkcel02 name='.\*DB\_IO.\*' objectname!='ASM' name!='.\*RQ.\*' ago\_unit=minute ago\_value=4 Example : ./csv\_exadata\_metrics\_history.pl cell=enkcel02 type='Instantaneous' name='.\*DB\_IO.\*' objectname!='ASM' name!='.\*RQ.\*' ago\_unit=hour ago\_value=4 Example : ./csv\_exadata\_metrics\_history.pl cell=enkcel01,enkcel02 type='Instantaneous' name='.\*DB\_IO.\*' objectname!='ASM' name!='.\*RQ.\*' ago\_unit=minute ago\_value=4 serial

You have to setup passwordless SSH from one cell to the other cells (Then you can launch the script from this cell and retrieve data from the other cells).

**The main options/features are:**

1. You can specify the cells on which you want to collect the metrics thanks to the **cell** or **groupfile** parameter.
2. You can choose to serialize the execution over the cells thanks to the **serial** parameter.
3. You can choose the type of metrics you want to retrieve (Cumulative, rate or instantaneous) thanks to the **type** &nbsp;parameter.
4. You can focus on some metrics thanks to the **name** &nbsp;parameter (wildcard allowed).
5. You can exclude some metrics thanks to the **name!** &nbsp;parameter (wildcard allowed).
6. You can focus on some metricobjectname thanks to the **objectname** &nbsp;parameter (wildcard allowed).
7. You can exclude some metricobjectname thanks to the **objectname!** &nbsp;parameter (wildcard allowed).
8. You can choose the unit to retrieve metrics back (day, hour, minute) thanks to the **ago\_unit** &nbsp;parameter.
9. You can choose the value associated to the unit to&nbsp;retrieve metrics back thanks to the **ago\_value** &nbsp;parameter.

**Let's see an example:**

I want to retrieve in csv format the metrics from 2 cells related to databases for the last 20 minutes:

```
$\> ./csv\_exadata\_metrics\_history.pl cell=enkx3cel01,enkx3cel02 name='DB\_.\*' ago\_unit=minute ago\_value=20 Cell;metricType;DateTime;name;objectname;value;unit enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;ACSTBY;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;ASM;15,779;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;BDT;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;BIGDATA;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;DBFS;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;DBM;15,779;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;DEMO;794,329;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;DEMOX3;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;EXDB;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;WZSDB;0.000;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_BY\_ALLOCATED;\_OTHER\_DATABASE\_;48,764;MB enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;ACSTBY;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;ASM;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;BDT;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;BIGDATA;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;DBFS;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;DBM;15;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;DEMO;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;DEMOX3;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;EXDB;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;WZSDB;0;MB/sec enkx3cel01;Instantaneous;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_BY\_SEC;\_OTHER\_DATABASE\_;0;MB/sec enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;ACSTBY;2,318;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;ASM;0;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;BDT;2,966;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;BIGDATA;25,415;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;DBFS;3,489;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;DBM;1,627,066;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;DEMO;4,506;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;DEMOX3;4,172;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;EXDB;0;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;WZSDB;4,378;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ;\_OTHER\_DATABASE\_;6,227;IO requests enkx3cel01;Cumulative;2015-07-01T08:57:59-05:00;DB\_FC\_IO\_RQ\_LG;ACSTBY;0;IO requests . . .
```

That way, you could visualize your data the way you feel comfortable with. For example, I used [tableau](https://public.tableau.com/s/) to create this "Database metrics dashboard" on DB\_\* rate metrics:

[![cells_metrics]({{ site.baseurl }}/assets/images/cells_metrics.png)](https://bdrouvot.files.wordpress.com/2015/07/cells_metrics.png)

**Remarks:**

- If you retrieve too much data, you could receive something like:

```
Error: enkx3cel02 is returning over 100000 lines; output is truncated !!! Command could be retried with the serialize option: --serial Killing child pid 15720 to enkx3cel02...
```

Then, you can launch the script with the serial option (see the help).

- You can download&nbsp;the _csv\_exadata\_metrics\_history.pl_ script from&nbsp;[this repository](https://docs.google.com/folder/d/0B7Jf_4JdsptpRHdyOWk1VTdUdEU/edit)&nbsp;, or from [Github](https://github.com/bdrouvot/csv_exadata_metrics_history).

**Conclusion:**

You probably already have a way to build&nbsp;your own graph of&nbsp;the historical metrics. But if you don't, feel free to use this script and the visualisation tool of your choice.


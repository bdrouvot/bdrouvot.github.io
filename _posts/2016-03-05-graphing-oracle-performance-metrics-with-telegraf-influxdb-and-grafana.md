---
layout: post
title: Graphing Oracle performance metrics with Telegraf, InfluxDB and Grafana
date: 2016-03-05 12:09:35.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Performance
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_job_id: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6111839879925166080&type=U&a=OWRW
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/706074194241409024";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/QKXMJbKHKVD
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
  _wpas_skip_7950430: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/"
---
## Introduction

As a picture is worth a thousand words, we may want&nbsp;to visualise the oracle performance metrics. There is already several tools to do so, but what if you could do it the way you want? build your own graph? Let's try to achieve this with 3 layers:

- [telegraf](https://influxdata.com/time-series-platform/telegraf/): to collect the oracle metrics
- [InfluxDB](https://influxdata.com/time-series-platform/influxdb/): to store the time-series oracle metrics
- [grafana](http://grafana.org/): to visualise the oracle time-series metrics

## Installation

Let's install InfluxDB and grafana that way (for example on the same host, namely influxgraf):

- Install InfluxDB (detail [here](https://influxdata.com/downloads/)):

```
[root@influxgraf ~]# wget https://s3.amazonaws.com/influxdb/influxdb-0.10.2-1.x86\_64.rpm [root@influxgraf ~]# yum localinstall influxdb-0.10.2-1.x86\_64.rpm
```

- Install grafana (detail [here](http://docs.grafana.org/installation/)):

```
[root@influxgraf ~]# yum install https://grafanarel.s3.amazonaws.com/builds/grafana-2.6.0-1.x86\_64.rpm
```

Let's install telegraf on the oracle host (namely Dprima) that way (detail [here](https://influxdata.com/downloads/)):

```
[root@Dprima ~]# wget http://get.influxdb.org/telegraf/telegraf-0.10.4.1-1.x86\_64.rpm [root@Dprima ~]# yum localinstall telegraf-0.10.4.1-1.x86\_64.rpm
```

## Setup

- Create a telegraf database into InfluxDB (using the web interface):

[![Screen Shot 2016-03-05 at 09.06.10]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-09-06-10.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-09-06-10/)

- Create a root user into InfluxDB (using the web interface):

[![Screen Shot 2016-03-05 at 09.08.38]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-09-08-38.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-09-08-38/)

- Write a script to collect the oracle metrics. I am using python but this is not mandatory at all. Only the output of the script does matter, it has to be InfluxDB [line-protocol](https://docs.influxdata.com/influxdb/v0.9/write_protocols/line/). The script query the&nbsp;_v$sysmetric_ and&nbsp;_v$eventmetric_&nbsp;views to get the wait class and the wait event metrics during the last minute. I am not reinventing the wheel, I am using [Kyle Hailey](http://datavirtualizer.com/wait-event-and-wait-class-metrics-vs-vsystem_event/)'s queries. The python code is:

```
[oracle@Dprima scripts]$ cat oracle\_metrics.py
```

[sourcecode language="python" wraplines="false" collapse="false"]  
import os  
import sys  
import cx\_Oracle  
import argparse  
import re

class OraMetrics():  
 def \_\_init\_\_(self, user, passwd, sid):  
 import cx\_Oracle  
 self.user = user  
 self.passwd = passwd  
 self.sid = sid  
 self.connection = cx\_Oracle.connect( self.user , self.passwd , self.sid )  
 cursor = self.connection.cursor()  
 cursor.execute("select HOST\_NAME from v$instance")  
 for hostname in cursor:  
 self.hostname = hostname[0]

def waitclassstats(self, user, passwd, sid):  
 cursor = self.connection.cursor()  
 cursor.execute("""  
 select n.wait\_class, round(m.time\_waited/m.INTSIZE\_CSEC,3) AAS  
 from v$waitclassmetric m, v$system\_wait\_class n  
 where m.wait\_class\_id=n.wait\_class\_id and n.wait\_class != 'Idle'  
 union  
 select 'CPU', round(value/100,3) AAS  
 from v$sysmetric where metric\_name='CPU Usage Per Sec' and group\_id=2  
 union select 'CPU\_OS', round((prcnt.busy\*parameter.cpu\_count)/100,3) - aas.cpu  
 from  
 ( select value busy  
 from v$sysmetric  
 where metric\_name='Host CPU Utilization (%)'  
 and group\_id=2 ) prcnt,  
 ( select value cpu\_count from v$parameter where name='cpu\_count' ) parameter,  
 ( select 'CPU', round(value/100,3) cpu from v$sysmetric where metric\_name='CPU Usage Per Sec' and group\_id=2) aas  
 """)  
 for wait in cursor:  
 wait\_name = wait[0]  
 wait\_value = wait[1]  
 print "oracle\_wait\_class,host=%s,db=%s,wait\_class=%s wait\_value=%s" % (self.hostname, sid,re.sub(' ', '\_', wait\_name), wait\_value )

def waitstats(self, user, passwd, sid):  
 cursor = self.connection.cursor()  
 cursor.execute("""  
 select  
 n.wait\_class wait\_class,  
 n.name wait\_name,  
 m.wait\_count cnt,  
 round(10\*m.time\_waited/nullif(m.wait\_count,0),3) avgms  
 from v$eventmetric m,  
 v$event\_name n  
 where m.event\_id=n.event\_id  
 and n.wait\_class \<\> 'Idle' and m.wait\_count \> 0 order by 1 """)  
 for wait in cursor:  
 wait\_class = wait[0]  
 wait\_name = wait[1]  
 wait\_cnt = wait[2]  
 wait\_avgms = wait[3]  
 print "oracle\_wait\_event,host=%s,db=%s,wait\_class=%s,wait\_event=%s count=%s,latency=%s" % (self.hostname, sid,re.sub(' ', '\_', wait\_class),re.sub(' ', '\_', wait\_name)  
, wait\_cnt,wait\_avgms)

if \_\_name\_\_ == "\_\_main\_\_":  
 parser = argparse.ArgumentParser()  
 parser.add\_argument('-u', '--user', help="Username", required=True)  
 parser.add\_argument('-p', '--passwd', required=True)  
 parser.add\_argument('-s', '--sid', help="SID", required=True)

args = parser.parse\_args()

stats = OraMetrics(args.user, args.passwd, args.sid)  
 stats.waitclassstats(args.user, args.passwd, args.sid)  
 stats.waitstats(args.user, args.passwd, args.sid)  
[/sourcecode]

The output looks like (the output format is the InfluxDB [line-protocol](https://docs.influxdata.com/influxdb/v0.9/write_protocols/line/)):

```
[oracle@Dprima scripts]$ python "/home/oracle/scripts/oracle\_metrics.py" "-u" "system" "-p" "bdtbdt" "-s" PBDT oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Administrative wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=CPU wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=CPU\_OS wait\_value=0.035 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Commit wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Concurrency wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Configuration wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Network wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Other wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=Scheduler wait\_value=0 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=System\_I/O wait\_value=0.005 oracle\_wait\_class,host=Dprima,db=PBDT,wait\_class=User\_I/O wait\_value=0 oracle\_wait\_event,host=Dprima,db=PBDT,wait\_class=System\_I/O,wait\_event=control\_file\_sequential\_read count=163,latency=0.009 oracle\_wait\_event,host=Dprima,db=PBDT,wait\_class=System\_I/O,wait\_event=control\_file\_parallel\_write count=60,latency=3.933 oracle\_wait\_event,host=Dprima,db=PBDT,wait\_class=System\_I/O,wait\_event=log\_file\_parallel\_write count=60,latency=1.35 oracle\_wait\_event,host=Dprima,db=PBDT,wait\_class=User\_I/O,wait\_event=Disk\_file\_operations\_I/O count=16,latency=0.037 oracle\_wait\_event,host=Dprima,db=PBDT,wait\_class=User\_I/O,wait\_event=Parameter\_File\_I/O count=16,latency=0.004
```

- On the oracle host, configure telegraf to execute&nbsp;the python script with 60 seconds interval and send the output to InfluxDB. Edit the&nbsp;_/etc/telegraf/telegraf.conf_ file so that it contains:

```
############################################################################### # OUTPUTS # ############################################################################### # Configuration for influxdb server to send metrics to [[outputs.influxdb]] urls = ["http://influxgraf:8086"] # required database = "telegraf" # required precision = "s" timeout = "5s" ############################################################################### # INPUTS # ############################################################################### # Oracle metrics [[inputs.exec]] # Shell/commands array commands = ["/home/oracle/scripts/oracle\_metrics.sh"] # Data format to consume. This can be "json", "influx" or "graphite" (line-protocol) # NOTE json only reads numerical measurements, strings and booleans are ignored. data\_format = "influx" interval = "60s"
```

The&nbsp;_oracle\_metrics.sh_ script contains the call to the python script:

```
#!/bin/env bash export LD\_LIBRARY\_PATH=/u01/app/oracle/product/11.2.0/dbhome\_1//lib export ORACLE\_HOME=/u01/app/oracle/product/11.2.0/dbhome\_1/ python "/home/oracle/scripts/oracle\_metrics.py" "-u" "system" "-p" "bdtbdt" "-s" PBDT
```

- Launch telegraf:

```
[root@Dprima ~]# telegraf -config /etc/telegraf/telegraf.conf
```

## Graph the metrics

- First check that the metrics are stored the way we want into InfluxDB:

[![Screen Shot 2016-03-05 at 09.49.09]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-09-49-09.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-09-49-09/)

&nbsp;

- Configure InfluxDB as datasource in grafana:

[![Screen Shot 2016-03-05 at 10.03.55]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-10-03-55.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-10-03-55/)

- In grafana, create a dashboard and create some variables (hosts, db and wait\_class):

[![Screen Shot 2016-03-05 at 10.59.00]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-10-59-00.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-10-59-00/)

[![Screen Shot 2016-03-05 at 10.59.57]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-10-59-57.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-10-59-57/)

- Now let's create a graph:

[![Screen Shot 2016-03-05 at 11.04.09]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-11-04-09.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-11-04-09/)

- Get the metrics:

[![Screen Shot 2016-03-05 at 11.06.18]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-11-06-18.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-11-06-18/)

- So that the graph looks like:

[![Screen Shot 2016-03-05 at 11.20.18]({{ site.baseurl }}/assets/images/screen-shot-2016-03-05-at-11-20-18.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-03-05-at-11-20-18/)

## Remarks

- Thanks to the variables defined in grafana, we can filter on the host, database and wait class (instead of visualising all of them).
- We can also&nbsp;graph&nbsp;the wait events we collected, or whatever you want to collect.

## Conclusion

Thanks to:

- telegraf and InfluxDB, we are able to collect and store the metrics we want to.
- grafana, we are able to visualise the metrics the way we want to.

### Update 04/21/2016:

As another&nbsp;example, with more informations collected:

[![dash1]({{ site.baseurl }}/assets/images/dash1.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/dash1/)

[![dash2]({{ site.baseurl }}/assets/images/dash2.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/dash2/)

[![dash3]({{ site.baseurl }}/assets/images/dash3.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/dash3/)

[![dash4]({{ site.baseurl }}/assets/images/dash4.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/dash4/)

[![dash5]({{ site.baseurl }}/assets/images/dash5.png)](https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/dash5/)

&nbsp;


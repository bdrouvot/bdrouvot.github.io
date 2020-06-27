---
layout: post
title: Push the oracle alert.log and listener.log into Elasticsearch and analyze/visualize
  their content with Kibana
date: 2016-03-26 09:09:53.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Other
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _oembed_f85637d8bd7e8f7068b1362423d5eb07: "{{unknown}}"
  _publicize_job_id: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6119404803438239744&type=U&a=1MLY
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/713639119654559744";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/V8NUMZ1f9rm
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/"
---
## Introduction

The oracle _alert.log_ and _listener.log_ contain useful information to provide answer to questions like:

- When did the Instance&nbsp;start?
- When has the Instance been shutdown?
- When did ORA- occur? With which code?
- Which IP client did connect to the Instance? With which user?
- How did it connect? Through a service? Through the SID?
- Which program has been used to connect?
- A connection storm occurred, what is the source of it?

What about having all this information centralized? What about having the possibility to&nbsp;gather, format, search, analyze and visualize this information in real time?

To achieve this, let's use the [ELK stack](https://www.elastic.co/products):

- [Logstash](https://www.elastic.co/products/logstash) to collect the information the way we want to.
- [Elasticsearch](https://www.elastic.co/products/elasticsearch)&nbsp;as an analytics engine.
- [Kibana](https://www.elastic.co/products/kibana) to visualize the data.

## Installation

The installation is very simple.

- Install elasticsearch

```
[root@elk ~]# wget https://download.elasticsearch.org/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.2.1/elasticsearch-2.2.1.rpm [root@elk ~]# yum localinstall elasticsearch-2.2.1.rpm
```

- Edit the configuration file to mention on which host&nbsp;it has been installed (namely&nbsp;_elk_ in my case):

```
[root@elk ~]# grep network.host /etc/elasticsearch/elasticsearch.yml network.host: elk
```

- Start elasticsearch:

```
[root@elk ~]# /etc/init.d/elasticsearch start Starting elasticsearch: [OK]
```

- Install Kibana:

```
[root@elk ~]# wget https://download.elastic.co/kibana/kibana/kibana-4.4.2-linux-x64.tar.gz [root@elk ~]# tar -xf kibana-4.4.2-linux-x64.tar.gz --directory /opt [root@elk ~]# mv /opt/kibana-4.4.2-linux-x64 /opt/kibana
```

- Edit the configuration file so that the url is updated accordingly:

```
[root@elk ~]# grep elasticsearch.url /opt/kibana/config/kibana.yml elasticsearch.url: "http://elk:9200"
```

- Start Kibana:

```
[root@elk ~]# /opt/kibana/bin/kibana
```

- Install logstash on the oracle host (namely _dprima_ in my case):

```
[root@dprima ~]# wget https://download.elastic.co/logstash/logstash/packages/centos/logstash-2.2.2-1.noarch.rpm [root@dprima ~]# yum localinstall logstash-2.2.2-1.noarch.rpm
```

## Configure logstash to push and format the alert.log to elasticsearch the way we want to

So that:

- The @timestamp field is reflecting the timestamp at which the log entry was created (rather than when logstash read the log entry).
- It traps ORA- entries and creates a field _ORA-_ when it occurs.
- It traps the start of the Instance (and fill a field _oradb\_status_ accordingly).
- It traps the shutdown of the Instance (and fill a field _oradb\_status_ accordingly).
- It traps the fact that the Instance is running&nbsp;(and fill a field _oradb\_status_ accordingly).

New fields are being created **so that we can analyze/visualize** them later on with Kibana.

- To trap and format this information, let's create an&nbsp;_alert\_log.conf_ configuration file that looks like (the filter part contains the important stuff):

[sourcecode language="python" wraplines="false" collapse="false"]  
input {  
 file {  
 path =\> "/u01/app/oracle/diag/rdbms/pbdt/PBDT/trace/alert\_PBDT.log"  
 }  
}

filter {

# Join lines based on the time  
 multiline {  
 pattern =\> "%{DAY} %{MONTH} %{MONTHDAY} %{TIME} %{YEAR}"  
 negate =\> true  
 what =\> "previous"  
 }

# Create new field: oradb\_status: starting,running,shutdown  
 if [message] =~ /Starting ORACLE instance/ {  
 mutate {  
 add\_field =\> ["oradb\_status", "starting"]  
 }  
 } else if [message] =~ /Instance shutdown complete/ {  
 mutate {  
 add\_field =\> ["oradb\_status", "shutdown"]  
 }  
 } else {  
 mutate {  
 add\_field =\> ["oradb\_status", "running"]  
 }  
 }

# Search for ORA- and create field if match

if [message] =~ /ORA-/ {  
 grok {  
 match =\> ["message","(?\<ORA-\>ORA-[0-9]\*)" ]  
 }  
}

# Extract the date and the rest from the message  
 grok {  
 match =\> ["message","%{DAY:day} %{MONTH:month} %{MONTHDAY:monthday} %{TIME:time} %{YEAR:year}(?\<log\_message\>.\*$)"]  
 }

mutate {  
 add\_field =\> {  
 "timestamp" =\> "%{year} %{month} %{monthday} %{time}"  
 }  
 }  
# replace the timestamp by the one coming from the alert.log  
 date {  
 locale =\> "en"  
 match =\> ["timestamp" , "yyyy MMM dd HH:mm:ss"]  
 }

# replace the message (remove the date)  
 mutate { replace =\> ["message", "%{log\_message}"] }

mutate {  
 remove\_field =\> ["time" ,"month","monthday","year","timestamp","day","log\_message"]  
 }

}

output {  
elasticsearch {  
hosts =\> ["elk:9200"]  
index =\> "oracle-%{+YYYY.MM.dd}"  
}  
}  
[/sourcecode]

- Start logstash with this configuration file:

```
[root@dprima ~]# /opt/logstash/bin/logstash -f /etc/logstash/conf.d/alert\_log.conf
```

- So that for example an entry in the _alert.log_ file like:

```
Sat Mar 26 08:30:26 2016 ORA-1653: unable to extend table SYS.BDT by 8 in tablespace BDT
```

will be formatted and send to elasticsearch that way:

```
{ "message" =\> "\nORA-1653: unable to extend table SYS.BDT by 8 in tablespace BDT ", "@version" =\> "1", "@timestamp" =\> "2016-03-26T08:30:26.000Z", "path" =\> "/u01/app/oracle/diag/rdbms/pbdt/PBDT/trace/alert\_PBDT.log", "host" =\> "Dprima", "tags" =\> [[0] "multiline" ], "oradb\_status" =\> "running", "ORA-" =\> "ORA-1653" }
```

## Configure logstash to push and format the listener.log to elasticsearch the way we want to

So that:

- The @timestamp field is reflecting the timestamp at which the log entry was created (rather than when logstash read the log entry).
- It traps the connections and records the program&nbsp;into a dedicated field&nbsp;_program_.
- It traps the connections and records the user&nbsp;into a dedicated field&nbsp;_user_.
- It traps the connections and records the ip of the client&nbsp;into a dedicated field&nbsp;_ip\_client._
- It traps the connections and records the destination into a dedicated field _dest_.
- It traps the connections and records the destination type (SID or service\_name) into a dedicated field _dest\_type_.
- It traps the command (stop, status, reload) and records it into a dedicated field _command_.

New fields are being created **so that we can analyze/visualize** them later on with Kibana.

- To trap and format this information, let's create a&nbsp;_lsnr\_log.conf_ configuration file that looks like (the filter part contains the important stuff):

[sourcecode language="python" wraplines="false" collapse="false"]  
input {  
 file {  
 path =\> "/u01/app/oracle/diag/tnslsnr/Dprima/listener/trace/listener.log"  
 }  
}

filter {

if [message] =~ /(?i)CONNECT\_DATA/ {

# Extract the date and the rest from the message  
 grok {  
 match =\> ["message","(?\<the\_date\>.\*%{TIME})(?\<lsnr\_message\>.\*$)"]  
 }

# Extract COMMAND (like status,reload,stop) and add a field  
 if [message] =~ /(?i)COMMAND=/ {

grok {  
 match =\> ["lsnr\_message","^.\*(?i)COMMAND=(?\<command\>.\*?)\).\*$"]  
 }

} else {

# Extract useful Info (USER,PROGRAM,IPCLIENT) and add fields  
 grok {  
 match =\> ["lsnr\_message","^.\*PROGRAM=(?\<program\>.\*?)\).\*USER=(?\<user\>.\*?)\).\*ADDRESS.\*HOST=(?\<ip\_client\>%{IP}).\*$"]  
 }  
 }

# replace the timestamp by the one coming from the listener.log  
 date {  
 locale =\> "en"  
 match =\> ["the\_date" , "dd-MMM-yyyy HH:mm:ss"]  
 }

# replace the message (remove the date)  
 mutate { replace =\> ["message", "%{lsnr\_message}"] }

# remove temporary fields  
 mutate { remove\_field =\> ["the\_date","lsnr\_message"] }

# search for SID or SERVICE\_NAME, collect dest and add dest type  
 if [message] =~ /(?i)SID=/ {  
 grok { match =\> ["message","^.\*(?i)SID=(?\<dest\>.\*?)\).\*$"] }  
 mutate { add\_field =\> ["dest\_type", "SID"] }  
 }

if [message] =~ /(?i)SERVICE\_NAME=/ {  
 grok { match =\> ["message","^.\*(?i)SERVICE\_NAME=(?\<dest\>.\*?)\).\*$"] }  
 mutate { add\_field =\> ["dest\_type", "SERVICE"] }  
 }

} else {  
 drop {}  
 }  
}

output {  
elasticsearch {  
hosts =\> ["elk:9200"]  
index =\> "oracle-%{+YYYY.MM.dd}"  
}  
}  
[/sourcecode]

- Start logstash with this configuration file:

```
[root@Dprima conf.d]# /opt/logstash/bin/logstash -f /etc/logstash/conf.d/lsnr\_log.conf
```

- So that for example an entry in the _listener.log_ file like:

```
26-MAR-2016 08:34:57 \* (CONNECT\_DATA=(SID=PBDT)(CID=(PROGRAM=SQL Developer)(HOST=\_\_jdbc\_\_)(USER=bdt))) \* (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.1)(PORT=50379)) \* establish \* PBDT \* 0
```

will be formatted and send to elasticsearch that way:

```
{ "message" =\> " \* (CONNECT\_DATA=(SID=PBDT)(CID=(PROGRAM=SQL Developer)(HOST=\_\_jdbc\_\_)(USER=bdt))) \* (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.1)(PORT=50380)) \* establish \* PBDT \* 0", "@version" =\> "1", "@timestamp" =\> "2016-03-26T08:34:57.000Z", "path" =\> "/u01/app/oracle/diag/tnslsnr/Dprima/listener/trace/listener.log", "host" =\> "Dprima", "program" =\> "SQL Developer", "user" =\> "bdt", "ip\_client" =\> "192.168.56.1", "dest" =\> "PBDT", "dest\_type" =\> "SID" }
```

## Analyze and Visualize the data with Kibana

- Connect to the elk host, (http://elk:5601) and create an index pattern (Pic 1):

[![elk-index-pattern]({{ site.baseurl }}/assets/images/elk-index-pattern.png)](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/elk-index-pattern/)

- Check that all our custom&nbsp;fields have been indexed (this is the default behaviour) (Pic 2):

[![all_indices]({{ site.baseurl }}/assets/images/all_indices3.png)](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/all_indices-3/)

so that we can now visualize them.

- Example 1: thanks to the _listener.log_ data, let's graph the connection repartition to our databases by _program_ and by _dest\_type (Pic 3)_:

[![kibana_example]({{ site.baseurl }}/assets/images/kibana_example.png)](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/kibana_example/)

- Example 2:&nbsp;thanks to the _listener.log_ data, visualize when a connection "storm" occurred and where it came&nbsp;from (_ip\_client_ field):

[![storm]({{ site.baseurl }}/assets/images/storm1.png)](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/storm-2/)

## Remarks

- As you can see (into the Pic 2) the index&nbsp;on the _program_ field has&nbsp;not been analyzed. By doing so, a connection to&nbsp;the database with "SQL Developer" will be&nbsp;stored in the index as "SQL Developer" and this is what we want. While an analyzed index would have stored 2 distincts values ("SQL" and "Developer"). The same apply for the _ORA-_ field: ORA-1653 would store ORA and 1653 if analyzed (This is why it is specified as not analyzed as well). You can find more details [here](https://www.elastic.co/guide/en/elasticsearch/guide/current/mapping-intro.html).

- To get the indexes on the&nbsp;_program_ and _ORA-_&nbsp;fields not analyzed, a template has been created that way:

```
curl -XDELETE elk:9200/oracle\* curl -XPUT elk:9200/\_template/oracle\_template -d ' { "template" : "oracle\*", "settings" : { "number\_of\_shards" : 1 }, "mappings" : { "oracle" : { "properties" : { "program" : {"type" : "string", "index": "not\_analyzed" }, "ORA-" : {"type" : "string", "index": "not\_analyzed" } } } } }'
```

- The configuration files are using grok. You can find more information about it&nbsp;[here](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html).
- You can find much more informations about the ELK stack (and another way to use it) into this [blog post](http://www.rittmanmead.com/2014/10/monitoring-obiee-with-elasticsearch-logstash-and-kibana/) from Robin Moffatt
- All you need to do to visualize the data is to extract the fields of interest from the log files and be sure an index is created on each field you want to visualize.
- All this information coming from all the&nbsp;machines of a datacenter being centralized into a single place is a gold mine from my point of view.

## Conclusion

Thanks to the ELK stack you&nbsp;can gather, centralize, analyze and visualize the content of the _alert.log_ and&nbsp;_listener.log_&nbsp;files for&nbsp;your&nbsp;whole datacenter the way you&nbsp;want to. This information is a gold mine and the&nbsp;imagination is the only limit.


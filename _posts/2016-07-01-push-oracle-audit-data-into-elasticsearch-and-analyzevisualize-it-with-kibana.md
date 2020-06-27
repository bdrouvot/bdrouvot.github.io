---
layout: post
title: Push oracle audit data into Elasticsearch and analyze/visualize it with Kibana
date: 2016-07-01 17:13:08.000000000 +02:00
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
  _publicize_job_id: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6154678040413683712&type=U&a=4YeP
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/748912356361506816";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/NEjxVTrwUcF
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/07/01/push-oracle-audit-data-into-elasticsearch-and-analyzevisualize-it-with-kibana/"
---

Introduction
------------

Auditing the oracle database may lead to a wide variety of information.

What about having all this information centralized? What about having the possibility to gather, format, search, analyze and visualize this information in near real time?

To achieve this, let’s use the [ELK stack](https://www.elastic.co/products):

-   [Logstash](https://www.elastic.co/products/logstash) to collect the information the way we want to.
-   [Elasticsearch](https://www.elastic.co/products/elasticsearch) as an analytics engine.
-   [Kibana](https://www.elastic.co/products/kibana) to visualize the data.

We'll focus on the audit information coming from:

-   The *dba\_audit\_trail* oracle view.
-   The audit files (linked to the *audit\_file\_dest* parameter).

You should first read this post: [Push the oracle alert.log and listener.log into Elasticsearch and analyze/visualize their content with Kibana](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/) prior to this one. The reason is that the current post relies on it (as the current post gives less explanation about Installation, setup and so on).

Installation
------------

The Installation of those 3 layers is the same as described into this [blog post](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/).

Configure logstash to push and format the dba\_audit\_trail records to elasticsearch the way we want to
-------------------------------------------------------------------------------------------------------

To achieve this we'll use the logstash’s JDBC input (Robin Moffatt provided an interesting use case and explanation of the logstash's JDBC input into this [blog post](https://www.elastic.co/blog/visualising-oracle-performance-data-with-the-elastic-stack)) so that:

-   The @timestamp field is reflecting the timestamp at which audit information has been recorded (rather than when logstash read the information).
-   It records the *os\_username*, *username*, *userhost*, *action\_name*, *sessionid*, *returncode*, *priv\_used* and *global\_uid* fields coming from the *dba\_audit\_trail* view into the elasticsearch.
-   It traps the kind of authentification (database, directory password..) and external name (if any) from the *comment\_text field* of the *dba\_audit\_trail* view.

To trap and format this information, let’s create an *audit\_database.conf* configuration file that looks like:

```
input {  
jdbc {  
jdbc_validate_connection =&gt; true  
jdbc_connection_string =&gt; "jdbc:oracle:thin:@localhost:1521/PBDT"  
jdbc_user =&gt; "system"  
jdbc_password =&gt; "bdtbdt"  
jdbc_driver_library =&gt; "/u01/app/oracle/product/11.2.0/dbhome_1/jdbc/lib/ojdbc6.jar"  
jdbc_driver_class =&gt; "Java::oracle.jdbc.driver.OracleDriver"  
statement =&gt; "select os_username,username,userhost,timestamp,action_name,comment_text,sessionid,returncode,priv_used,global_uid from dba_audit_trail where timestamp &gt; :sql_l  
ast_value"  
schedule =&gt; "*/2 * * * *"  
}  
}

filter {  
# Set the timestamp to the one of dba_audit_trail  
mutate { convert =&gt; [ "timestamp" , "string" ]}  
date { match =&gt; ["timestamp", "ISO8601"]}

if [comment_text] =~ /(?i)Authenticated by/ {

grok {  
match =&gt; [ "comment_text","^.*(?i)Authenticated by: (?&lt;authenticated_by&gt;.*?);.*$" ]  
}

if [comment_text] =~ /(?i)EXTERNAL NAME/ {  
grok {  
match =&gt; [ "comment_text","^.*(?i)EXTERNAL NAME: (?&lt;external_name&gt;.*?);.*$" ]  
}  
}  
}

# remove temporary fields  
mutate { remove_field =&gt; ["timestamp"] }  
}

output {  
elasticsearch {  
hosts =&gt; ["elk:9200"]  
index =&gt; "audit_databases_oracle-%{+YYYY.MM.dd}"  
}  
}  
```

so that an entry into *dba\_audit\_trail* like:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2016-07-01-at-06-58-40.png" class="aligncenter size-full wp-image-3097" width="640" height="280" alt="Screen Shot 2016-07-01 at 06.58.40" />

will be formatted and send to elasticsearch that way:

    {
             "os_username" => "bdt",
                "username" => "ORG_USER",
                "userhost" => "bdts-MacBook-Pro.local",
             "action_name" => "LOGON",
            "comment_text" => "Authenticated by: DIRECTORY PASSWORD;EXTERNAL NAME: cn=bdt_dba,cn=users,dc=bdt,dc=com; Client address: (ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.1)(PORT=49515))",
               "sessionid" => 171615.0,
              "returncode" => 0.0,
               "priv_used" => "CREATE SESSION",
              "global_uid" => "4773e70a9c6f4316be03169d8a06ecab",
                "@version" => "1",
              "@timestamp" => "2016-07-01T06:47:51.000Z",
        "authenticated_by" => "DIRECTORY PASSWORD",
           "external_name" => "cn=bdt_dba,cn=users,dc=bdt,dc=com"
    }

Configure logstash to push and format the \*.aud files content to elasticsearch the way we want to
--------------------------------------------------------------------------------------------------

So that:

-   The @timestamp field is reflecting the timestamp at which audit information has been recorded (rather than when logstash read the information).
-   It records the action, the database user, the privilege, the client user, the client terminal, the status and the dbid into the elasticsearch.

To trap and format this information, let’s create an *audit\_files.conf* configuration file that looks like:

```
input {  
file {  
path =&gt; "/u01/app/oracle/admin/PBDT/adump/*.aud"  
}  
}

filter {

# Join lines based on the time  
multiline {  
pattern =&gt; "%{DAY} %{MONTH} *%{MONTHDAY} %{TIME} %{YEAR}.*"  
negate =&gt; true  
what =&gt; "previous"  
}

# Extract the date and the rest from the message  
grok {  
match =&gt; [ "message","%{DAY:day} %{MONTH:month} *%{MONTHDAY:monthday} %{TIME:time} %{YEAR:year}(?&lt;audit_message&gt;.*$)" ]  
}

grok {  
match =&gt; [ "audit_message","^.*ACTION :[[0-9]*] (?&lt;action&gt;.*?)DATABASE USER:[[0-9]*] (?&lt;database_user&gt;.*?)PRIVILEGE :[[0-9]*] (?&lt;privilege&gt;.*?)CLIENT USER:[[0-9]*] (?&lt;cl ient_user&gt;.*?)CLIENT TERMINAL:[[0-9]*] (?&lt;client_terminal&gt;.*?)STATUS:[[0-9]*] (?&lt;status&gt;.*?)DBID:[[0-9]*] (?&lt;dbid&gt;.*$?)" ]  
}

if "_grokparsefailure" in [tags] { drop {} }

mutate {  
add_field =&gt; {  
"timestamp" =&gt; "%{year} %{month} %{monthday} %{time}"  
}  
}

# replace the timestamp by the one coming from the audit file  
date {  
locale =&gt; "en"  
match =&gt; [ "timestamp" , "yyyy MMM dd HH:mm:ss" ]  
}

# remove temporary fields  
mutate { remove_field =&gt; ["audit_message","day","month","monthday","time","year","timestamp"] }

}

output {  
elasticsearch {  
hosts =&gt; ["elk:9200"]  
index =&gt; "audit_databases_oracle-%{+YYYY.MM.dd}"  
}  
}  
```

so that an audit file content like:

    Fri Jul  1 07:13:56 2016 +02:00
    LENGTH : '160'
    ACTION :[7] 'CONNECT'
    DATABASE USER:[1] '/'
    PRIVILEGE :[6] 'SYSDBA'
    CLIENT USER:[6] 'oracle'
    CLIENT TERMINAL:[5] 'pts/1'
    STATUS:[1] '0'
    DBID:[10] '3270644858'

will be formatted and send to elasticsearch that way:

    {
                "message" => "Fri Jul  1 07:13:56 2016 +02:00\nLENGTH : '160'\nACTION :[7] 'CONNECT'\nDATABASE USER:[1] '/'\nPRIVILEGE :[6] 'SYSDBA'\nCLIENT USER:[6] 'oracle'\nCLIENT TERMINAL:[5] 'pts/1'\nSTATUS:[1] '0'\nDBID:[10] '3270644858'\n",
               "@version" => "1",
             "@timestamp" => "2016-07-01T07:13:56.000Z",
                   "path" => "/u01/app/oracle/admin/PBDT/adump/PBDT_ora_2387_20160701071356285876143795.aud",
                   "host" => "Dprima",
                   "tags" => [
            [0] "multiline"
        ],
                 "action" => "'CONNECT'\n",
          "database_user" => "'/'\n",
              "privilege" => "'SYSDBA'\n",
            "client_user" => "'oracle'\n",
        "client_terminal" => "'pts/1'\n",
                 "status" => "'0'\n",
                   "dbid" => "'3270644858'\n"
    }

Analyze and Visualize the data with Kibana
------------------------------------------

The Kibana configuration has already been described into this [blog post](https://bdrouvot.wordpress.com/2016/03/26/push-the-oracle-alert-log-and-listener-log-into-elasticsearch-and-analyzevisualize-their-content-with-kibana/).

Let's see 2 examples of audit data visualisation:

-   Example 1: thanks to the *dba\_audit\_trail* data, let’s graph the connection repartition to our databases by *authentification type, username* and *returncode:*

<img src="{{ site.baseurl }}/assets/images/screen-shot-2016-07-01-at-07-42-20.png" class="aligncenter size-full wp-image-3098" width="640" height="292" alt="Screen Shot 2016-07-01 at 07.42.20" />

As we can see most of the connections are authenticated by Directory Password and are successful.

-   Example 2: thanks to the *\*.aud files *data, let’s graph the sysdba connection over time and their status:

<img src="{{ site.baseurl }}/assets/images/screen-shot-2016-07-01-at-08-06-11.png" class="aligncenter size-full wp-image-3100" width="640" height="287" alt="Screen Shot 2016-07-01 at 08.06.11" />

As we can see, some of the sysdba connections are not successful between 07:57 am and 7:58 am. Furthermore the number of unsuccessful connections is greater than the number of successful ones.

Conclusion
----------

Thanks to the ELK stack you can gather, centralize, analyze and visualize the oracle audit data* *for your whole datacenter the way you want to.

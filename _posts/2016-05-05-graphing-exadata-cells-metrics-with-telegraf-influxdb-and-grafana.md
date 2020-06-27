---
layout: post
title: Graphing Exadata cells metrics with Telegraf, InfluxDB and Grafana
date: 2016-05-05 16:05:07.000000000 +02:00
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
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6134004811189805057&type=U&a=N1gz
  _publicize_job_id: '1'
  _publicize_done_2435436: '1'
  _wpas_done_2077996: '1'
  _wpas_skip_7950430: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:2225791;s:61:"https://twitter.com/BertrandDrouvot/status/728239125225082880";}}
  _publicize_done_2558296: '1'
  _wpas_done_2225791: '1'
  publicize_twitter_user: BertrandDrouvot
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/ExT8KFiejvt
  _publicize_done_8471251: '1'
  _wpas_done_8482624: '1'
  _wpas_skip_8482624: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2016/05/05/graphing-exadata-cells-metrics-with-telegraf-influxdb-and-grafana/"
---
<h2>Introduction</h2>
<p>As a picture is worth a thousand words, we may want to visualise the Exadata cells metrics. What if you could do it the way you want? build your own graph? Let’s try to achieve this with 3 layers:</p>
<ul>
<li><a href="https://influxdata.com/time-series-platform/telegraf/" target="_blank">telegraf</a>: to collect the Exadata metrics</li>
<li><a href="https://influxdata.com/time-series-platform/influxdb/" target="_blank">InfluxDB</a>: to store the time-series Exadata metrics</li>
<li><a href="http://grafana.org/" target="_blank">grafana</a>: to visualise the Exadata metrics</li>
</ul>
<p>You should first read this post: <a href="https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/" target="_blank">Graphing Oracle performance metrics with Telegraf, InfluxDB and Grafana</a> prior to this one. The reason is that the current post relies on it (as the current post gives less explanation about Installation, setup and so on).</p>
<h2>Installation</h2>
<ul>
<li>The Installation of those 3 layers is the same as described into this <a href="https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/" target="_blank">blog post</a>.</li>
<li>Telegraf has been installed on one database node.</li>
<li>The <a href="https://uhesse.com/tag/dcli/" target="_blank">dcli</a> utility has also been copied on this database node.</li>
</ul>
<h2>Setup</h2>
<p>The setup is the same as described into this <a href="https://bdrouvot.wordpress.com/2016/03/05/graphing-oracle-performance-metrics-with-telegraf-influxdb-and-grafana/" target="_blank">blog post</a> except for the script being used to collect the metrics:</p>
<ul>
<li>The script collects the cells metrics from one database node and prints the output as InfluxDB <a href="https://docs.influxdata.com/influxdb/v0.9/write_protocols/line/" target="_blank">line-protocol</a>.</li>
<li>The script assumes passwordless ssh setup to connect from the database node to the cells.</li>
<li>It collects <strong>all</strong> metriccurrent metrics that are not cumulative: the dcli command being used is:</li>
</ul>
<pre style="padding-left:30px;">$&gt; dcli -g ./cells_group cellcli -e "list metriccurrent attributes name,metricObjectName,metricValue,metricType where metricType !='cumulative'"</pre>
<p>The perl script code is:</p>
<pre style="padding-left:30px;">$&gt; cat influxdb_exadata_metrics.pl</pre>
<p>[code language="perl"]<br />
#!/usr/bin/env perl<br />
#<br />
# Author: Bertrand Drouvot<br />
# influxdb_exadata_metrics.pl : V1.0 (2016/05)<br />
#<br />
# Utility used to extract exadata metrics in InfluxDB line protocol<br />
#<br />
#----------------------------------------------------------------#</p>
<p>BEGIN {<br />
die &quot;ORACLE_HOME not set\n&quot; unless $ENV{ORACLE_HOME};<br />
unless ($ENV{OrAcLePeRl}) {<br />
$ENV{OrAcLePeRl} = &quot;$ENV{ORACLE_HOME}/perl&quot;;<br />
$ENV{PERL5LIB} = &quot;$ENV{PERL5LIB}:$ENV{OrAcLePeRl}/lib:$ENV{OrAcLePeRl}/lib/site_perl&quot;;<br />
$ENV{LD_LIBRARY_PATH} = &quot;$ENV{LD_LIBRARY_PATH}:$ENV{ORACLE_HOME}/lib32:$ENV{ORACLE_HOME}/lib&quot;;<br />
exec &quot;$ENV{OrAcLePeRl}/bin/perl&quot;, $0, @ARGV;<br />
}<br />
}</p>
<p>use strict;<br />
use Time::Local;</p>
<p>#<br />
# Variables<br />
#<br />
my $nbmatch=-1;<br />
my $help=0;<br />
my $goodparam=0;</p>
<p>my $dclicomm='';<br />
my $groupcell_pattern='';<br />
my $metrictype_pattern='ALL';<br />
my $result;</p>
<p># Parameter parsing</p>
<p>foreach my $para (@ARGV) {</p>
<p>if ( $para =~ m/^help.*/i ) {<br />
$nbmatch++;<br />
$help=1;<br />
}</p>
<p>if ( $para =~ m/^groupfile=(.*)$/i ) {<br />
$nbmatch++;<br />
$groupcell_pattern=$1;<br />
$goodparam++;<br />
}<br />
}</p>
<p># Check if groupfile is empty</p>
<p>if ((!$goodparam) | $goodparam &gt; 1) {<br />
print &quot;\n Error while processing parameters : GROUPFILE parameter is mandatory! \n\n&quot; unless ($help);<br />
$help=1;<br />
}</p>
<p># Print usage if a difference exists between parameters checked<br />
#<br />
if ($nbmatch != $#ARGV | $help) {<br />
print &quot;\n Error while processing parameters \n\n&quot; unless ($help);<br />
print &quot; \nUsage: $0 [groupfile=] \n\n&quot;;</p>
<p>printf (&quot; %-25s %-60s %-10s \n&quot;,'Parameter','Comment','Default');<br />
printf (&quot; %-25s %-60s %-10s \n&quot;,'---------','-------','-------
');  
printf (" %-25s %-60s %-10s \n",'GROUPFILE=','file containing list of cells','');  
print ("\n");  
print ("utility assumes passwordless SSH from this node to the cell nodes\n");  
print ("utility assumes ORACLE\_HOME has been set \n");  
print ("\n");  
print ("Example : $0 groupfile=./cell\_group\n");  
print "\n\n";  
exit 0;  
}

# dcli command

$dclicomm="dcli -g ".$groupcell\_pattern. " cellcli -e \"list metriccurrent attributes name,metricObjectName,metricValue,metricType where metricType !='cumulative'\"";

# Launch the dcli command  
my $result=`$dclicomm` ;

if ( $? != 0 )  
{  
print "\n";  
print "\n";  
die "Something went wrong executing [$dclicomm]\n";  
}

# Split the string into array  
my @array\_result = split(/\n/, $result);

foreach my $line ( @array\_result ) {  
# drop tab  
$line =~ s/\t+/ /g;  
# drop blanks  
$line =~ s/\s+/ /g;

#Split each line on 6 pieces based on blanks  
my @tab1 = split (/ +/,$line,6);

# Supress : from the cell name  
$tab1[0] =~ s/://;

# Supress , from the value  
$tab1[3] =~ s/,//g;

# Add "N/A" if no Unit  
if ($tab1[5] eq "") {$tab1[5]="N/A"};

# Print  
print "exadata\_cell\_metrics,cell=$tab1[0],metric\_name=$tab1[1],metricObjectName=$tab1[2],metric\_type=$tab1[5],metric\_unit=$tab1[4] metric\_value=$tab1[3]\n";  
}  
[/code]

The output looks like (the output format is the InfluxDB [line-protocol](https://docs.influxdata.com/influxdb/v0.9/write_protocols/line/)):

```
$\> perl ./influxdb\_exadata\_metrics.pl groupfile=./cells\_group exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk01\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk02\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk03\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk04\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk05\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk06\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk07\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk08\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk09\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=CD\_disk10\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=FD\_00\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_BY\_FC\_DIRTY,metricObjectName=FD\_01\_cell12,metric\_type=Instantaneous,metric\_unit=MB metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk01\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk02\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk03\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk04\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk05\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk06\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk07\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk08\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk09\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=CD\_disk10\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=FD\_00\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 exadata\_cell\_metrics,cell=cell1,metric\_name=CD\_IO\_BY\_R\_LG\_SEC,metricObjectName=FD\_01\_cell12,metric\_type=Rate,metric\_unit=MB/sec metric\_value=0.000 . . .
```

- The output has been cut for readability.
- The groupfile parameter is mandatory: This file contains the list of the cells you want the metrics to be collected on.

On the database node, configure telegraf to execute the perl script with 60 seconds interval and send the output to InfluxDB. Edit the _/etc/telegraf/telegraf.conf_ file so that it contains:

```
############################################################################### # OUTPUT PLUGINS # ############################################################################### # Configuration for influxdb server to send metrics to [[outputs.influxdb]] urls = ["http://influxgraf:8086"] # required database = "telegraf" # required precision = "s" timeout = "5s" ############################################################################### # INPUTS # ############################################################################### # Exadata metrics [[inputs.exec]] # Shell/commands array commands = ["/home/oracle/scripts/exadata\_metrics.sh"] # Data format to consume. This can be "json", "influx" or "graphite" (line-protocol) # NOTE json only reads numerical measurements, strings and booleans are ignored. data\_format = "influx" interval = "60s"
```

The _exadata\_metrics.sh_ script contains the call to the perl&nbsp;script:

```
$\> cat exadata\_metrics.sh #!/bin/env bash export ORACLE\_HOME=/u01/app/oracle/product/12.1.0/dbhome\_1 export PATH=$PATH:/home/oracle/scripts perl /home/oracle/scripts/influxdb\_exadata\_metrics.pl groupfile=/home/oracle/scripts/cells\_group
```

Now you can connect to grafana and create a new dashboard with the Exadata cells metrics the way you want to.

## Example

[![Screen Shot 2016-05-05 at 16.37.40]({{ site.baseurl }}/assets/images/screen-shot-2016-05-05-at-16-37-40.png)](https://bdrouvot.wordpress.com/2016/05/05/graphing-exadata-cells-metrics-with-telegraf-influxdb-and-grafana/screen-shot-2016-05-05-at-16-37-40/)

## Remark

Nothing has been installed on the cells to collect&nbsp;those metrics with telegraf.

## Conclusion

Thanks to:

- telegraf and InfluxDB, we are able to collect and store the Exadata cells metrics we want to.
- grafana, we are able to visualise the Exadata cells metrics the way we want to.

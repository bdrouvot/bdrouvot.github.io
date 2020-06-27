---
layout: post
title: 12c Application Continuity with Data Guard in action
date: 2015-05-07 18:36:44.000000000 +02:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Availability
tags: []
meta:
  _edit_last: '40807211'
  geo_public: '0'
  _publicize_pending: '1'
  publicize_google_plus_url: https://plus.google.com/105401307688264718604/posts/3gq8Bay7nKj
  publicize_facebook_url: https://facebook.com/
  _wpas_done_8482624: '1'
  _publicize_done_external: a:1:{s:11:"google_plus";a:1:{s:21:"105401307688264718604";b:1;}}
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/5w8rE0StHm
  _wpas_done_2225791: '1'
  publicize_linkedin_url: https://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=6002133793006239744&type=U&a=rddk
  _wpas_done_2077996: '1'
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
permalink: "/2015/05/07/12c-application-continuity-with-data-guard-in-action/"
---

Introduction
------------

<span class="s1">Application Continuity enables replay, in a non-disruptive and rapid manner, of a database request when a recoverable error makes the database session unavailable. Application Continuity uses Transaction Guard. You can find more details on those features [here](http://www.oracle.com/technetwork/products/clustering/ac-overview-1967264.html).</span>

With RAC, Application Continuity can be used to launch the replay on another Instance. It can also be used with Data Guard under some circumstances and conditions (Extract from [here](http://docs.oracle.com/database/121/SBYDB/concepts.htm#SBYDB5461)):

[<img src="{{ site.baseurl }}/assets/images/screen-shot-2015-05-07-at-08-52-52.png" class="aligncenter size-full wp-image-2809" width="640" height="83" alt="Screen Shot 2015-05-07 at 08.52.52" />](https://bdrouvot.files.wordpress.com/2015/05/screen-shot-2015-05-07-at-08-52-52.png)

Well, I just want to see this feature in action during a Data Guard switchover. Let's give it a try.

Test preparation
----------------

The test environment is made of:

-   NBDT database (12.1.0.2 and initially primary).
-   SNBDT database (12.1.0.2 and initially standby).

On the primary database, connected as user bdt, let's create the following tables:

    SQL> create table BDT_CONT as select rownum num,rpad('U',20) val from dual connect by level <= 500000;

    Table created.

    SQL> create table BDT_SPY (d date, instance_name varchar2(16), num number);

    Table created.

And the following trigger:

    create or replace trigger BDT_CONT_SPY_T
    before update
    on BDT_CONT
    for each row
    begin
    insert into BDT_SPY values(sysdate,sys_context('USERENV', 'INSTANCE_NAME'), :old.num);
    end;
    /

    Trigger created.

The test application will update the 500000 rows of the BDT\_CONT table in one transaction. The aim of the trigger is to track on which Instance the update has been successfully executed .

Now let's create a service on the primary and standby databases that way:

    # Primary
    $ srvctl add service -d NBDT -s appco -preferred NBDT1,NBDT2 -l primary -policy automatic -j SHORT -B SERVICE_TIME -z 30 -w 10 -commit_outcome TRUE -e TRANSACTION -replay_init_time 1800 -retention 86400 -notification TRUE

    # Standby
    $ srvctl add service -d SNBDT -s appco -preferred SNBDT1,SNBDT2  -l primary -policy automatic -j SHORT -B SERVICE_TIME -z 30 -w 10 -commit_outcome TRUE -e TRANSACTION -replay_init_time 1800 -retention 86400 -notification TRUE

So that the appco service:

-   Is automatically started on the standby if it becomes primary.
-   Is defined for Application Continuity (see more details [here](http://www.oracle.com/technetwork/database/database-cloud/private/application-continuity-wp-12c-1966213.pdf) on how to configure a service for Application Continuity).

Now let's create a JAVA application that will be used to test the Application Continuity:

```
import java.sql.\*;  
import oracle.jdbc.pool.\*;  
import oracle.jdbc.\*;  
import oracle.jdbc.replay.\*;

public class AppCont {  
public String getInstanceName(Connection conn) throws SQLException {  
PreparedStatement preStatement = conn.prepareStatement("select instance\_name from v$instance");  
String r=new String();  
ResultSet result = preStatement.executeQuery();

while (result.next()){  
r=result.getString("instance\_name");  
}  
return r;  
}

public static void main(String args\[\]) throws SQLException {  
Connection conn = null;  
Statement stmt = null;  
try {  
// OracleDataSourceImpl instead of OracleDataSource.  
OracleDataSourceImpl ocpds = new OracleDataSourceImpl();  
// Create the database URL  
String dbURL =  
"jdbc:oracle:thin:@(DESCRIPTION\_LIST="+  
"(LOAD\_BALANCE=off)"+  
"(FAILOVER=on)"+  
"(DESCRIPTION=(CONNECT\_TIMEOUT=90) (RETRY\_COUNT=10)(RETRY\_DELAY=3)"+  
"(ADDRESS\_LIST="+  
"(LOAD\_BALANCE=on)"+  
"(ADDRESS=(PROTOCOL=TCP) (HOST=rac-cluster-scan)(PORT=1521)))"+  
"(CONNECT\_DATA=(SERVICE\_NAME=appco)))"+  
"(DESCRIPTION=(CONNECT\_TIMEOUT=90) (RETRY\_COUNT=10)(RETRY\_DELAY=10)"+  
"(ADDRESS\_LIST="+  
"(LOAD\_BALANCE=on)"+  
"(ADDRESS=(PROTOCOL=TCP)(HOST=srac-cluster-scan)(PORT=1521)))"+  
"(CONNECT\_DATA=(SERVICE\_NAME=appco))))";

ocpds.setURL(dbURL);  
ocpds.setUser("bdt");  
ocpds.setPassword("bdt");

// Get a connection  
conn = ocpds.getConnection();

// Instantiation  
AppCont ex = new AppCont();

// On which Instance are we connected?  
System.out.println("Instance Name = "+ex.getInstanceName(conn));

System.out.println("Performing transaction");  
/\* Don't forget to disable autocommit  
\* And launch the transaction  
\*/  
((oracle.jdbc.replay.ReplayableConnection)conn).beginRequest();  
conn.setAutoCommit(false);  
stmt = conn.createStatement();  
String updsql = "UPDATE BDT\_CONT " +  
"SET val=UPPER(val)";  
stmt.executeUpdate(updsql);  
conn.commit();  
// End  
((oracle.jdbc.replay.ReplayableConnection)conn).endRequest();

stmt.close();

System.out.println("Instance Name = "+ex.getInstanceName(conn));

// On which Instance are we connected?  
conn.close();  
}  
catch (Exception e) {  
e.printStackTrace();  
}  
}  
}  
```

The important parts are:

-   The use of OracleDataSourceImpl instead of OracleDataSource.
-   The database URL definition that includes the access to the appco service for both the primary and standby databases (NBDT is hosted on "rac-cluster" and SNBDT is hosted on "srac-cluster").
-   Autocommit is disabled (Read [this](https://blogs.oracle.com/WebLogicServer/entry/part_2_12c_database_and) if you need more details).

We are ready to launch the application and test the Application Continuity in a Data Guard context.

Run it
------

First let's check the status of the data guard configuration and that it is ready for the switchover:

    DGMGRL for Linux: Version 12.1.0.2.0 - 64bit Production

    Copyright (c) 2000, 2013, Oracle. All rights reserved.

    Welcome to DGMGRL, type "help" for information.
    DGMGRL> connect sys/toto@NBDT
    Connected as SYSDBA.
    DGMGRL> show configuration;

    Configuration - nbdt_dr

      Protection Mode: MaxPerformance
      Members:
      nbdt  - Primary database
        snbdt - Physical standby database

    Fast-Start Failover: DISABLED

    Configuration Status:
    SUCCESS   (status updated 24 seconds ago)

    DGMGRL> validate database SNBDT;

      Database Role:     Physical standby database
      Primary Database:  nbdt

      Ready for Switchover:  Yes
      Ready for Failover:    Yes (Primary Running)

      Flashback Database Status:
        nbdt:   Off
        snbdt:  Off

Launch the application

    $ cat launch.ksh
    export CLASSPATH=/u01/app/oracle/product/12.1.0.2/jdbc/lib/ojdbc7.jar:.
    time java -cp $CLASSPATH:. AppCont
    $ ksh ./launch.ksh
    Instance Name = NBDT1
    Performing transaction

At this stage, launch the switchover (from another terminal):

    DGMGRL> switchover to snbdt;
    Performing switchover NOW, please wait...
    Operation requires a connection to instance "SNBDT1" on database "snbdt"
    Connecting to instance "SNBDT1"...
    Connected as SYSDBA.
    New primary database "snbdt" is opening...
    Oracle Clusterware is restarting database "nbdt" ...
    Switchover succeeded, new primary is "snbdt"

And then wait until the application ends:

    $ ksh ./launch.ksh
    Instance Name = NBDT1
    Performing transaction
    .
    .   <= Switchover occurred here
    .
    Instance Name = SNBDT1

    real    1m46.19s
    user    0m1.33s
    sys 0m0.13s

As we can see:

-   No errors have been reported.
-   The application terminated on the SNBDT1 instance (Then on the new primary database).

Let's check (thanks to the spy table) on which Instance did the transaction commit:

    $ sqlplus bdt/bdt@SNBDT

    SQL*Plus: Release 12.1.0.2.0 Production on Thu May 7 10:04:54 2015

    Copyright (c) 1982, 2014, Oracle.  All rights reserved.

    Last Successful login time: Thu May 07 2015 09:59:05 +02:00

    Connected to:
    Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
    With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
    Advanced Analytics and Real Application Testing options

    SQL> select distinct(instance_name) from bdt_spy;

    INSTANCE_NAME
    ----------------
    SNBDT1

As we can see the transaction has been replayed and did commit on the SNBDT database (The new primary database).

Remarks
-------

1.  Application Continuity is supported for Oracle Data Guard switchovers to physical standby databases. It is also supported for fast-start failover to physical standbys in maximum availability data protection mode.
2.  Don't forget that the primary and standby databases must be licensed for Oracle RAC or Oracle Active Data Guard in order to use Application Continuity.
3.  The bdt oracle user that has been used during the test, has grant execute on DBMS\_APP\_CONT.
4.  This Java Application has been created and shared by [Laurent Leturgez](http://laurent-leturgez.com/) during the [Paris Oracle Meetup](http://www.meetup.com/fr/parisoracle/). User groups are always good, as you can share your work with others. Thanks again Laurent!

Conclusion
----------

We have seen the Application Continuity feature in Action in a Data Guard context.

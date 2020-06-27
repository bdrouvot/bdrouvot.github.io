---
layout: post
title: Recovering after the loss of Redo Log files thanks to the file descriptor
date: 2013-12-10 10:06:08.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- Perl Scripts
tags: []
meta:
  _edit_last: '40807211'
  _publicize_pending: '1'
  publicize_linkedin_url: http://www.linkedin.com/updates?discuss=&scope=16310177&stype=M&topic=5816100316625932288&type=U&a=tdHm
  publicize_twitter_user: BertrandDrouvot
  publicize_twitter_url: http://t.co/uiTxGS20Z5
  _wpas_done_2225791: '1'
  _publicize_done_external: a:1:{s:7:"twitter";a:1:{i:246399475;b:1;}}
  _wpas_done_2077996: '1'
  _wpas_skip_2225791: '1'
  _wpas_skip_2077996: '1'
author:
  login: bdrouvot
  email: bdtoracleblog@gmail.com
  display_name: bdrouvot
  first_name: ''
  last_name: ''
permalink: "/2013/12/10/recovering-after-the-loss-of-redo-log-files-thanks-to-the-file-descriptor/"
---

This morning when I came at work I discovered that all the redo log files (current, active, inactive..) have been removed from one of our database (useless to say by mistake) resulting in:

    Errors in file /ec/prod/server/oracle/orabdt/u000/admin/BDT/diag/rdbms/bdt/BDT/trace/BDT_m000_17002.trc:
    ORA-00313: open failed for members of log group 1 of thread 1
    ORA-00312: online log 1 thread 1: '/ec/prod/server/oracle/orabdt/u801/oraredo/BDT/redoBDT1b.log'
    ORA-27037: unable to obtain file status
    SVR4 Error: 2: No such file or directory
    Additional information: 3

<span style="text-decoration:underline;">As the database is a large one, I tried to avoid to restore it:</span>

1.  My first reflex is to check if the flashback database is set to on: Unfortunately it was not the case.
2.  Then I remembered this blog post of Jason Arneil: [Recovering from rm -rf on a datafile](http://jarneil.wordpress.com/2013/04/23/recovering-from-rm-rf-on-a-datafile/).

<span style="text-decoration:underline;">And then I decided to give it a try for the redo log files that way:</span>

-   First I searched for the lgwr process ID associated to this database

<!-- -->

    ps -ef | grep -i lgwr | grep -i BDT
      oracle 17321  8422   0   Sep 10 ?         291:07 ora_lgwr_BDT

-   Now let's search for the file descriptor linked to the redo logs files: As I don't have access to lsof (nor pfiles) and as I want to know the Seq\#, I did the research that way (redo logs contain the "Thread and Seq\# strings) (output truncated):

<!-- -->

    cd /proc/17321/fd
    for i in *
    > do
    > echo "for fd: $i"
    > strings $i | egrep "Thread.*Seq#"
    > done
    for fd: 258
    Thread 0001, Seq# 0000005285, SCN
    for fd: 259
    Thread 0001, Seq# 0000005285, SCN
    for fd: 260
    Thread 0001, Seq# 0000005284, SCN
    for fd: 261
    Thread 0001, Seq# 0000005284, SCN

-   Then based on the Seq\#, and the v$log and v$logfile output I recreated the redo logs that way:

<!-- -->

    cat 258 > /ec/prod/server/oracle/orabdt/u800/oraredo/BDT/redoBDT1a.log
    cat 259 > /ec/prod/server/oracle/orabdt/u801/oraredo/BDT/redoBDT1b.log
    cat 260 > /ec/prod/server/oracle/orabdt/u800/oraredo/BDT/redoBDT2a.log
    cat 261 > /ec/prod/server/oracle/orabdt/u801/oraredo/BDT/redoBDT2b.log

-   At that time we received:

<!-- -->

    Tue Dec 10 08:06:43 2013
    Archived Log entry 5285 added for thread 1 sequence 5285 ID 0xb2ffdbd dest 1:
    Archiver process freed from errors. No longer stopped
    Tue Dec 10 08:06:43 2013
    Beginning log switch checkpoint up to RBA [0x14a7.2.10], SCN: 9310332806181
    ORA-00322: log 2 of thread 1 is not current copy
    ORA-00312: online log 2 thread 1: '/ec/prod/server/oracle/orabdt/u801/oraredo/BDT/redoBDT2b.log'
    ORA-00322: log 2 of thread 1 is not current copy
    ORA-00312: online log 2 thread 1: '/ec/prod/server/oracle/orabdt/u800/oraredo/BDT/redoBDT2a.log'
    .....
    .....
    ORA-00314: log 1 of thread 1, expected sequence# 5287 doesn't match 5285
    ORA-00312: online log 1 thread 1: '/ec/prod/server/oracle/orabdt/u801/oraredo/BDT/redoBDT1b.log'
    ORA-00314: log 1 of thread 1, expected sequence# 5287 doesn't match 5285
    ORA-00312: online log 1 thread 1: '/ec/prod/server/oracle/orabdt/u800/oraredo/BDT/redoBDT1a.log'

-   So in our case, the "restore" has not been enough. Then we created new redo log files and we cleared and dropped the ones we recovered thanks to the file descriptor that way:

<!-- -->

    ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 1
    WARNING! CLEARING REDO LOG WHICH HAS NOT BEEN ARCHIVED. BACKUPS TAKEN
        BEFORE 12/10/2013 08:10:00 (CHANGE 9310332823323) CANNOT BE USED FOR RECOVERY.
    Clearing online log 1 of thread 1 sequence number 5287
    Archived Log entry 5287 added for thread 1 sequence 5288 ID 0xb2ffdbd dest 1:
    Archiver process freed from errors. No longer stopped
    Tue Dec 10 08:14:44 2013
    Completed: ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 1
    ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 2
    Tue Dec 10 08:14:54 2013
    Beginning global checkpoint up to RBA [0x14a9.5cea.10], SCN: 9310382261905
    Completed checkpoint up to RBA [0x14a9.5cea.10], SCN: 9310382261905
    Tue Dec 10 08:14:55 2013
    WARNING! CLEARING REDO LOG WHICH HAS NOT BEEN ARCHIVED. BACKUPS TAKEN
        BEFORE 12/10/2013 08:06:43 (CHANGE 9310332806181) CANNOT BE USED FOR RECOVERY.
    Clearing online log 2 of thread 1 sequence number 5286
    Completed: ALTER DATABASE CLEAR UNARCHIVED LOGFILE GROUP 2
    ALTER DATABASE DROP LOGFILE GROUP 1
    Completed: ALTER DATABASE DROP LOGFILE GROUP 1
    ALTER DATABASE DROP LOGFILE GROUP 2
    Completed: ALTER DATABASE DROP LOGFILE GROUP 2

And then the database has been able to work as expected and we launched a full backup of it.

<span style="text-decoration:underline;">Remarks:</span>

1.  Thanks to [Jason Jarnei](http://jarneil.wordpress.com/)l and [Frits Hoogland](http://fritshoogland.wordpress.com/) for their initial findings/blog post related to the restore of a lost datafile thanks to the file descriptor.
2.  In our case restoring the redo log files from the file descriptors has not been enough, but it's worth trying as a last chance.
3.  Adding new redo logs should not be necessary. Clearing the "dropped/restored" redo logs with "ALTER DATABASE CLEAR (UNARCHIVED) LOGFILE GROUP &lt;n&gt;" could be enough as stated [here](http://docs.oracle.com/cd/B19306_01/backup.102/b14191/recoscen.htm):

<img src="{{ site.baseurl }}/assets/images/loss_redo.png" class="aligncenter size-full wp-image-1573" width="620" height="205" alt="loss_redo" />

<span style="text-decoration:underline;">Conclusion:</span>

1.  We have been able to restore the service without the need of a database restore/recover (even if the "restore" from the file descriptor has not been enough).
2.  From my point of view you should try this **as a last chance before restoring/stopping** the database: restore from the file descriptor and if this is not enough launch the "alter database clear logfile group &lt;n&gt;" or "alter database clear unarchived logfile group &lt;n&gt;" on those restored redo logs.

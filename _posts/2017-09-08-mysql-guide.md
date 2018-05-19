---
layout: post
title: Beginners guide to understanding and improving your MySQL DB - Part 1
categories: mysql howto performance
---

## Intro
You might find yourself in a similar position to me where you have a big MySQL DB you don’t know much about. Maybe you are even responsible for it and get complaints that it’s too slow or too big. And maybe — like me — you don’t know much about databases apart from writing some simple SELECT, INSERT and other SQL statements. So where do you start to understand this black box?

> ## Disclaimer
> Being a beginner who has done a lot of reading and poking around I still don’t really know what I’m doing. I’ve tried all the things below but they still might be wrong and even dangerous for your DB. Be careful. Don’t run them on production unless you’ve tried them in a test environment. I'd also recommend that you ask more experienced colleagues for their opinion.

## Connecting
First you need a login to connect to the DB. Ask your colleagues if you don’t have one (or look in the source code or config mangement — only works if your company is not following security best practises).
 You can either connect from you laptop through a MySQL client like MySQL Workbench (which is quite nice to explore the tables and data).
 Or ssh to the MySQL server and start a MySQL shell (what I mostly do).  
 Depending how the MySQL users are configured some credentials might only be allowed from localhost others only from (certain) remote hosts. On the MySQL server you can also check /etc/mysql/conf.d for cnf-files that contain credentials. Depending on your configuration and the user you are it might be enough to run mysql or that you need to specify 

{% highlight bash %}
$ mysql -uUSERNAME -p --enable-cleartext-plugin
{% endhighlight %}


## Get some basic info
If you know nothing about the MySQL server the first thing you want to run is probably to list all databases:
{% highlight mysql %}
mysql> show databases;
{% endhighlight %}
Some of them are databases that are used by MySQL itself

* **mysql** — contains for example user table with all MySQL users and their permissions
* **information_schema** — contains lots of interesting information about the DB and tables e.g. the number of rows, size of the data and the indexes etc.
* **performance_schema** — contains very detailed, low-level information what is going on in the database (if enabled) e.g. in which states it is for how long, what it is waiting for etc.

And then there should be one or more used by your services / websites.

 To see what tables are in a database:
{% highlight mysql %}
mysql> use DATABASE_NAME
mysql> show tables;
{% endhighlight %}
If you add full it also tells you if a table is a view or not.
{% highlight mysql %}
mysql> show full tables;
{% endhighlight %}
To get details about a table use either:
{% highlight mysql %}
mysql> describe TABLENAME;
{% endhighlight %}
or
{% highlight mysql %}
mysql> show create table TABLENAME;
{% endhighlight %}
The first one gives you all the columns with their definition. The second one returns the SQL statement to create the table which also contains indexes or partitioning.

> **Note:** You probably know that all statements need to be finished with a “ ; ”. But that’s not completly true — on the MySQL shell you can also use “ \G ” to finish a statement e.g. “ show tables\G “ and that will give the output in some sort of list instead of in an ASCII table (which can be very hard to read sometimes).

## The size of your tables

You might be tempted to run something like
{% highlight mysql %}
mysql> select count(*) from TABLENAME;
{% endhighlight %}
to find out how many rows there are in a table but that is not a good idea and can take minutes on a big table and causes unnecessary load.
It’s much better to get this from the information_schema through something like this:

{% highlight mysql %}
mysql> SELECT table_name, table_rows FROM information_schema.tables where table_schema = "DBNAME" and table_name="TABLENAME";
{% endhighlight %}
This finishes almost instantly even for tables with 100s of millions of rows. But “information_schema.tables” has much more interesting data as well. Below command gives you the number of rows of the biggest tables together with the size of the data, the indexes and total table size. You can also order it by data_length or any other column you like.
{% highlight mysql %}
mysql> SELECT table_name, table_rows, data_length, index_length, round(data_free / (1024*1024)) as "free in MB", round(((data_length + index_length) / (1024*1024)),2) as "size in MB" FROM information_schema.tables where table_schema = "DBNAME" order by table_rows desc limit 20;
{% endhighlight %}

The tables from this command are good candidates to analyse more in depth and also to walk around and ask questions like “what is that table used for”, “why is it so big”, “do we need to keep the data forever or can it be deleted after X days?”, “why do the indexes take up 3 times the size of the table”.   
It’s a good idea to be sitting and somewhere without hard surfaces or sharp edges nearby when you ask those questions. Hearing “oh, we are not really using that one” for a 100 GB and 200 million row table can be slightly shocking.

<center><img src="../public/gifs/faint.gif"></center>



The “free” column shows how much space the rows take that are marked as deleted but have not been deleted yet. This can also contain some nasty surprises. I happened to find a table once that had 260 GB of deleted rows (yes, still taking 260 GB on the file system).

I’ll get to how to reclaim that space and investigating indexes that take to much space in the next part.

## What’s happening right now
OK, now you know what databases and tables there are and might have learned some depressing facts which make you regret ever looking into the DB. Let’s have a look how to find out what’s currently going on that might help when there are immediate problems.

Here is the status of a DB that's quite bored. The amount of slow queries, open tables and queries per seconds (QPS) are probably the most interesting here:
{% highlight bash %}
$ mysqladmin status
 Uptime: 199296 Threads: 3 Questions: 104705 Slow queries: 12 Opens: 3410 Flush tables: 1 Open tables: 512 Queries per second avg: 0.525
{% endhighlight %}

 To see the running queries use:
{% highlight bash %}
 $ mysqladmin processlist
{% endhighlight %}
 on the bash / Linux shell or from the MySQL shell:
{% highlight mysql %}
 mysql> show processlist\G
*************************** 1. row ***************************
     Id: 9
   User: root
   Host: localhost
     db: mysql
Command: Query
   Time: 0
  State: starting
   Info: show processlist
{% endhighlight %}
 It shows the query, the host from which the connection originates, which MySQL user is used, which DB is queried, how long it has been running and more. The command field tells you if it’s still running or an idle connection.

 One of the favourite past time of all admins is killing queries that are running for too long (that’s what the ID is for) and cause other queries to queue up and break things.

Killing one query is simply running:
{% highlight mysql %}
  mysql> kill ID;
{% endhighlight %}

 If you ever need to kill lots of connections that take longer than x seconds, from a certain IP or similar then something like this should work:
{% highlight bash %}
 $ mysql --skip-column-names --batch -e "select id from INFORMATION_SCHEMA.PROCESSLIST where time > 5;" | xargs echo
{% endhighlight %}
 Just customize the where clause to your requirements and replace echo with “*mysqladmin kill*”.

 A useful bash command to, let’s say,  figure out if a service went mad and has opened 1000 connections to the DB is this:
{% highlight bash %}
 $ mysql --batch -e "select host from information_schema.processlist;" | cut -d ":" -f 1 | sort | uniq -c
{% endhighlight %}
 Which returns the number of connections for each IP/host.

 If you have the awesome **percona-toolkit** installed (just one “apt-get install percona-toolkit” away) then you don’t need to rely on having this command always in your .bash_history but instead you just run:
{% highlight bash %}
 $ pt-mysql-summary
{% endhighlight %}

 This gives you a full analysis of the processlist!

 It also contains lots of information about the configuration and some of the most interesting variables regarding what’s going on in the DB, e.g. the number of selects, inserts, updates or tmp files/tables, how depressing your hit rate of the query cache is, the number of waiting queries, pending I/O and much more.
 
 All this information can be quite overwhelming at first but that's normal. Databases are complicated beasts after all. Over time you'll learn about all the different aspects and more and more of the (runtime) variables and configuration parameters will start to make sense. Speaking of configuration let's:

## Get some recommendations for your MySQL configuration
 While it’s important to configure your MySQL server correctly I have sometimes the impression that there is too much voodoo and magical thinking around how to configure the variables just perfectly. The best config won’t help you if your table structure and queries are awful. As it is much easier to change the config it’s also more likely that it is in a better state than the rest.

 However it might be misconfigured by some well-meaning admin or using defaults that were a good idea 5 years ago. If possible it’s good to start with a fresh (i.e. empty) config and only change what you really need to.

 There are a couple of tools that analyse your current config and also take a lot of information about your tables and how the DB is performing when executing queries into account. The recommendations they produce for buffer/memory/cache sizes are then based on whether that cache is full or almost empty when your DB is running.

 *  [mysqltuner.pl](https://github.com/major/MySQLTuner-perl)

{% highlight bash %}
$ wget https://github.com/major/MySQLTuner-perl/blob/master/mysqltuner.pl
$ perl mysqltuner.pl
{% endhighlight %}
 *  [tuning-primer.sh](https://launchpad.net/mysql-tuning-primer)

{% highlight bash %}
$ wget https://launchpad.net/mysql-tuning-primer/trunk/1.6-r1/+download/tuning-primer.sh
$ bash tuning-primer.sh
{% endhighlight %}
 * Or from the percona-toolkit


{% highlight bash %}
$ pt-variable-advisor localhost
{% endhighlight %}
 I’d suggest that for each recommendation you receive you have a look at its MySQL documentation and some StackOverflow posts before you change them.

Before you start implementing all the recommendations and playing around with config values you should think about how you can tell whether your changes helped, hurt or didn't do anything.

## Some thoughts about performance optimisations
Making things run faster is cool and can provide a lot of value. And there is this conception that you just have to do the right change and boom it runs 100% faster. And if you are smart enough (a genius or wizard) it takes almost no effort.   
In reality - when you want to do it right - it is a lot of work. And the gains of each improvement probably aren't that impressive either. Lets have a look why that is.

For some things it is relatively easy to tell that they are helping more than they are hurting. For example if you improve a slow query by only changing the query itself it is very likely that this won't cause problems anywhere else.
If you improve a slow query by adding an index it is a bit less clear what the side effects might be. Adding an index requires disk space and more memory (for when the DB caches/loads it into memory). Inserts, update and deletes will become slightly slower because the index needs to be updated and even selects can be affected negatively by (a lot of) indexes because when MySQL plans the execution it has to consider whether to use an index for a certain select or not. It could also happen that a different select starts using this new index and becomes slower. In most cases adding an index will do more good than harm but if you have many, many indexes they come with a cost.
If you improve a query by changing MySQL configuration variables it becomes very hard to tell what effects this has on other queries and if it improves the overall performance or not.

There are a few different things to test the impact of your changes:

* Run a test query before and after the change
* Look at variables before and after the change (with the help of a monitoring tool like Percona Monitoring and Management)
* Compare the performance of a pre-defined set of queries that you run before and after the change (e.g. with the pt-upgrade tool)
* Run webtests against the application that uses the DB and measure page-loading / test execution time

Especially the last two are a lot of work and can cause you a lot of headaches. In all cases you get better results when you have a separate test DB that nobody else uses during your tests. Even if you have all that and tested everything you can think of there might be applications that you have forgotten and what about the cron jobs that only run once per week?

I think that illustrates quite well that doing performance optimisations right is not easy and a lot of work. But let's not despair and tackle the ways to test your changes one by one.

### Running a test query before and after the change

Quite self-explanatory. You have a query e.g. a select that is slow. Maybe takes a few seconds to execute. You run it, do your change e.g. adding an index and run the query again. When you did the right change your query should be really fast now, hopefully just a few milliseconds.

### Look at variables before and after the change

Your MySQL DB contains lots and lots of information about the state of the DB and how it is performing. There is the `show variables;` command and lots of "runtime" information stored in the performance_schema, information_schema and MySQL databases. Searching for interesting variables there and monitoring them for changes isn't fun at all. One tool that's awesome for that task is [Percona Monitoring and Management (PMM)](https://www.percona.com/software/database-tools/percona-monitoring-and-management) tool. See for yourself at this [demo](https://pmmdemo.percona.com/graph/dashboard/db/mysql-overview?refresh=1m&orgId=1) which cool dashboards you get out of the box.

There is an AWS AMI or Docker container for the server and deb or rpm packages for the client that needs to be installed on the DB server. The setup is very easy too.

So when you try some recommendations for your MySQL config from the mysql-tuner you can check how the metrics in the PMM are affected. 

It's also very helpful to find problems e.g. if a buffer is too small or the DB spends a lot of time waiting for IO.

### Compare the performance of a pre-defined set of queries that you run before and after the change

This is quite a good way to see the broader effect of your changes. It might happen that your change makes one query run faster but slows down another. By running more queries than just the one you optimised you can catch side effects you didn't think of.

Of course the tricky question is which queries should you run against the DB. If you know the DB and your use-cases well you could hand pick the most relevant ones. Or you can log/record the queries that are made against the DB and replay them (and if you want also: analyse e.g. for frequency and pick a subset). That way you can get a pretty realistic snapshot and a meaningful performance test.

I'll cover this in more detail in another part of this series. For now I'll just say pt-upgrade is your friend here. Have a look at the [man page](https://www.percona.com/doc/percona-toolkit/LATEST/pt-upgrade.html).

### Run webtests against the application that uses the DB and measure page-loading / test execution time

By running webtests on a build server like Jenkins the application or website and also your DB is used in the same way as a user would use it. Through this you have the full picture (no components are missing) and if you are lucky you can say through my change on the DB page x loads 10% faster for our users. Awesome. 

On the flip side these tests take the longest (often many hours) and your improvements can be hidden behind the variance of the webtests. Running the webtests a couple of times without making any changes is likely to produce executions times with differences of seconds or even minutes. If your tests exhibit such a degree of variance it is impossible to tell whether your changes play a role there or not. Also if the application code that's being tested changes frequently or more tests are added then the execution time will change because of that too. And you'll have to run the costly tests again in order to establish a new baseline.

If your improvements on the database are never visible in the webtests that could also mean you are trying to improve the wrong part of the system. For example if some DB requests take ~0.1ms and the application usually ~1s then reducing the time a DB requests takes by a few milliseconds won't produce a noticable change and you probably should look into the application first. 

While running webtests is great to see the overall impact of your changes most of the time your probably better of using one of the other methods.

## Summary

In this first (already quite long) part we covered how you can get some basic information about your database, get a sense what tables are used most are good candidates for optimisations, find out what's currently going on and if your configuration is any good and how to approach testing your changes.

In the next part I'd like to get into fixing some problems like finally deleting data that is marked deleted or analysing slow queries, overall query frequency, finding missing and duplicate indices and much more.

Let me know what ugly surprises you have found in your databases and what tips you have to understand and improve them.
    

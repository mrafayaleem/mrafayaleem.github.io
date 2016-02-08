---
layout: post
comments: true
title: Live MySQL Schema Changes on Amazon RDS with Percona - A Walkthrough
---
At [dubizzle](https://dubai.dubizzle.com/about/), we recently reached a roadblock on a project where we had to change schema on around 130 tables in our production database with master/slave replication. The simplest possible solution that could have been was to recreate same tables with new schemas, copy the data and update our application to point to the new ones. This couldn't have been made possible without a significant downtime for the whole website which wasn't acceptible. So, we leveraged Percona Toolkit for MySQL to do the job and we would like to share our learnings in this post.

### What is Percona Toolkit for MySQL?
Percona Toolkit is a collection of open source command line tools to perform a variety of MySQL database tasks. Reconsider the problem that we had in question and how we could have solved it manually. 

- Recreate around 130 tables with updated schema.
- Take the website down or put it in read only mode to ensure zero data discrepancy between old and new tables. 
- Copy everything from old tables to the new ones.
- Rebuild foreign keys.
- Rename new and old tables or update the app to point to the new ones.
- Redeploy the website.

The other solution was to setup a complex set of triggers, copy/read/write operations and have a rollback plan. All of this sounds very risky to manage on your own without expecting any downtime and this is where Percona Toolkit fits in to do the job for you.

Percona Toolkit has a bunch of tools for various tasks. The one we are going to discuss here in this post is `pt-online-schema-change`.

### Preparing for the big day
Simply testing the tool on your local machine before doing it on production is no plan at all. You have to have a similar replica of production at your disposal to test how the change would perform. For this reason, we experimented `pt-online-schema-change` on a production like set up which we call the *staging* environment. Making sure that each and every parameter/configuration on the staging database is exactly the same as production is a very crucial part of testing how the migration would perform in production.

> A note about AWS RDS MySQL instances: AWS RDS doesn't give you SUPER privileges on your MySQL instances so you would have to play around and tweek your DB Parameter Group. The most important parameter in context of this post is the `log_bin_trust_function_creators`. The default value for this in most of the cases is `0` (disabled). 

> Qouting from MySQL docs here:
This variable applies when binary logging is enabled. It controls whether stored function creators can be trusted not to create stored functions that will cause unsafe events to be written to the binary log. If set to 0 (the default), users are not permitted to create or alter stored functions unless they have the SUPER privilege in addition to the CREATE ROUTINE or ALTER ROUTINE privilege. A setting of 0 also enforces the restriction that a function must be declared with the DETERMINISTIC characteristic, or with the READS SQL DATA or NO SQL characteristic. If the variable is set to 1, MySQL does not enforce these restrictions on stored function creation. This variable also applies to trigger creation.

Since `pt-online-schema-change` cannot get SUPER privilege on RDS instances, `log_bin_trust_function_creators` has to be enabled for trigger creation and other routines. You can read more about it at [MySQL docs](https://dev.mysql.com/doc/refman/5.7/en/stored-programs-logging.html). 

### What you need to know about `pt-online-schema-change`
Earlier, we discussed how one of the solutions could be to setup a complex set of triggers and copy/read/write operations to perform schema changes. Internally, `pt-online-schema-change` tool works in pretty much the same way. Lets look at an overview of what it does for that.

- Creates an empty copy of the old table with the new schema.
- Creates triggers on the old tables to update corresponding rows in the new ones.
- Copy all the records from old tables to the new one.
- Rebuild relationships.
- Swap old/new tables by doing atomic RENAME TABLE operation. 
- Drop the old tables (default behaviour).

*Note: The operations listed are not necessarily performed in the same order.*

A typical `pt-online-schema-change` command for performing schema change would look something like this:

{% highlight bash %}
pt-online-schema-change --dry-run --nocheck-replication-filters --recursion-method="dsn=D=<database>,t=dsns" --chunk-size=2000 --alter-foreign-keys-method=rebuild_constraints --alter 'add column other_id INT DEFAULT NULL, add column item_hash VARCHAR(255) DEFAULT NULL, add column json_data TEXT DEFAULT NULL, add column item_id VARCHAR(255) DEFAULT NULL, add column from_other_item TINYINT DEFAULT NULL' h=<dbhost>,D=<database>,t=<table_name>
{% endhighlight %}

#### Command options and explanations:

*Note: Some of the description has been copied verbatim from `pt-online-schema-change` <a href=https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#pt-online-schema-change>docs</a> for berevity.*

**dry-run:** Create and alter the new table, but do not create triggers, copy data, or replace the original table. A safe mechanism to see what actual migration might look like. Note that it cannot reproduce an exact production like migration scenario. Think of it as a mock test.

**nocheck-replication-filters:** Abort if any replication filter is set on any server. The tool looks for server options that filter replication, such as `binlog_ignore_db` and `replicate_do_db`. If it finds any such filters, it aborts with an error.

If the replicas are configured with any filtering options, you should be careful not to modify any databases or tables that exist on the master and not the replicas, because it could cause replication to fail. For more information on replication rules, see <a href=http://dev.mysql.com/doc/en/replication-rules.html target="_blank">http://dev.mysql.com/doc/en/replication-rules.html</a>.

The default values for this option is `yes` so if you don't intend to use it, make sure that there are no replication filters on your slave. Replication filters are rules to make decisions about whether to execute or ignore statements. For your slave, these rules define which statements received from the master needs to be executed or ignored. Now consider a case where you have a filter on your slave that says don't execute any `ALTER` statement for `table_a`. When you do the schema change for `table_a` on master, the slave would never see that change. Eventually, this could cause the replication to fail after the `RENAME` operation. For this reason, the default value for this is `yes`. If you decide to change it to `no`, make yourself aware of the replication filters that you have on your slaves before you get into a situation where replication starts failing for one of your tables.

**recursion-method:** This specifies the methods that the tool uses to find slave hosts. Methods are as follows:

```
METHOD       USES
===========  ==================
processlist  SHOW PROCESSLIST
hosts        SHOW SLAVE HOSTS
dsn=DSN      DSNs from a table
none         Do not find slaves
```

However, for various reasons, your RDS instance might be configured to not give up the correct slave host information to `pt-online-schema-change` as this was the case with our setup. The most concrete way to do this is to create a DSN (Data Source Name) table for `pt-online-schema-change` to read information from. For this, create a table with the following structure:

{% highlight sql %}
CREATE TABLE `dsns` (`id` int(11) NOT NULL AUTO_INCREMENT, `parent_id` int(11) DEFAULT NULL, `dsn` varchar(255) NOT NULL, PRIMARY KEY (`id`));
{% endhighlight %}

After that, you can specify your slave hosts by creating entries for them like this:

{% highlight sql %}
INSERT INTO dsns(dsn) VALUES('h=<slave_host>,P=3306');
{% endhighlight %}

By specifying `recursion-method="dsn=D=mydatabase,t=dsns"`, you are telling percona to find slaves in a table callled `dsns` in database `mydatabase`.

**alter-foreign-keys-method:** This is only required if you have child tables that reference the tables that are going to be changed as part of the schema change. The recommended method is `rebuild_constraints` which uses ALTER TABLE on child tables to drop and re-add foreign key refereneces that reference the new table. The other two riskier options are `drop_swap` and `none` and if you happen to use them, please make sure that you know the intricate details. You can read about them in [pt-online-schema-change docs](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--alter-foreign-keys-method).


**chunk-size:** The tool performs copy operation in small chunks of data (a chunk is a collection of rows). This option governs the number of rows that are selected for the copy operation and overriding the default value would disable the dynamic chunk size adjustment behaviour. This option does not work with tables without primary keys or unique indexes because these are used for chunking the tables.

During the schema change, there could be a situation where the number of rows in a chunk on one of your replicas is more than what is there on master. This could happen because of replica lag and you would have to adjust your `chunk-size` for that. Note that a larger chunk size means more load on your database.

Following are some of the resources to look into how Percona Toolkit handles chunking.

*Related options are:* `chunk-size-limit`, `chunk-time` and `alter-foreign-keys-method`.

*Additional Resources:*

- [How Percona Toolkit divides tables into chunks](http://www.xaprb.com/blog/2012/05/06/how-percona-toolkit-divides-tables-into-chunks/)
- [Chunk Size Command](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#cmdoption-pt-online-schema-change--chunk-size)

**alter:** The schema modification that you want to apply.

You can read more on rest of the options here: [pt-online-schema-change docs](https://www.percona.com/doc/percona-toolkit/2.1/pt-online-schema-change.html#pt-online-schema-change)

### The Migration Plan:

- Prepare a full fledged replica (call it staging) of your production environment with exact master/slave replicaion. Make sure that your staging is sufficiently populated with data.
- Creating artificial load on your staging databases when testing live schema changes is highly recommended. You can use tools such as [Selenium](http://www.seleniumhq.org/) and [JMeter](http://jmeter.apache.org/) to do that.
- Make sure that DB Parameter Group in AWS RDS on prodction and staging are same and `log_bin_trust_function_creators` is set to `1`.
- Prepare a set of scripts to do dry runs and actual schema changes. This may include a list of tables and a shell script which would run `pt-online-schema-change` and store the logs for each run.
- Make sure that you store logs for each and every run (staging and production both).
- Do dry runs on staging and production to see the expected outcome.
- Have someone from your DevOps team to help you out in case if things start falling apart during the production migration.
- Choose an appropriate time do the migration. Ideally, this should be the least busiest time for your website.
- Take backups of production before the migration in case you have to restore.

### Log Sample Output:
Here is a sample log from one of our migrations. 

{% highlight bash %}
Found 3 slaves:
  ip-some-number
  ip-some-number
  ip-some-number
Will check slave lag on:
  ip-some-number
  ip-some-number
  ip-some-number
Operation, tries, wait:
  copy_rows, 10, 0.25
  create_triggers, 10, 1
  drop_triggers, 10, 1
  swap_tables, 10, 1
  update_foreign_keys, 10, 1
Altering `some_database`.`some_table`...
Creating new table...
Created new table some_database._some_table_new OK.
Altering new table...
Altered `some_database`.`_some_table_new` OK.
2015-10-21T02:35:03 Creating triggers...
2015-10-21T02:35:03 Created triggers OK.
2015-10-21T02:35:03 Copying approximately 10000 rows...
2015-10-21T02:35:05 Copied rows OK.
2015-10-21T02:35:05 Swapping tables...
2015-10-21T02:35:06 Swapped original and new tables OK.
2015-10-21T02:35:06 Dropping old table...
2015-10-21T02:35:06 Dropped old table `some_database`.`_some_table_old` OK.
2015-10-21T02:35:06 Dropping triggers...
2015-10-21T02:35:06 Dropped triggers OK.
Successfully altered `some_database`.`some_table`.
{% endhighlight %}

### Errors seen on Production:
{% highlight bash %}
2015-10-19T12:24:52 Error copying rows from `some_database`.`some_table` to `some_database`.`_some_table_new`: 2015-10-19T12:24:52 Cannot copy table `some_database`.`some_table` because on the master it would be checksummed in one chunk but on these replicas it has too many rows:
  4130 rows on ip-some-number
The current chunk size limit is 4000 rows (chunk size=1000 * chunk size limit=4.0).
{% endhighlight %}

Resolution: Increase chunk size

### Conclusion and Results:
`pt-online-schema-change` from Percona worked exceptionally well with us. The migration was tested very well on our staging environment and it worked in pretty much the same way on production.

---

**Helpful commands:**

- To get list of tables with particular column: 
{% highlight sql %}
SELECT DISTINCT TABLE_NAME  FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME NOT REGEXP "^test_.*(_archive)|(_new)|(_old)$" AND TABLE_NAME REGEXP "^test_.*" AND COLUMN_NAME = 'feed_id' AND TABLE_SCHEMA ='some_schema';
{% endhighlight %}
- To find number of connections: 
{% highlight sql %}
SHOW STATUS WHERE `variable_name` = 'Threads_connected';
{% endhighlight %}
- To find if some table is locked: 
{% highlight sql %}
SHOW OPEN TABLES WHERE `Table` LIKE '%[TABLE_NAME]%' AND `Database` LIKE '[DBNAME]' AND In_use > 0;
{% endhighlight %}
- To find all foreign keys on a table:
{% highlight sql %}
SELECT TABLE_NAME,COLUMN_NAME,CONSTRAINT_NAME, REFERENCED_TABLE_NAME,REFERENCED_COLUMN_NAME from INFORMATION_SCHEMA.KEY_COLUMN_USAGE where REFERENCED_TABLE_NAME = '<table_name>';
{% endhighlight %}

**Percona toolkit docs (2.2+)**

- [pt-online-schema-change docs](https://www.percona.com/doc/percona-toolkit/2.2/pt-online-schema-change.html#pt-online-schema-change)

**Other resources:**

- [MySQL Replication Rules](http://dev.mysql.com/doc/refman/5.6/en/replication-rules.html)
- [MySQL Show Slave Hosts](https://dev.mysql.com/doc/refman/5.0/en/show-slave-hosts.html)
- [Percona Table Checksum](https://www.percona.com/doc/percona-toolkit/2.2/pt-table-checksum.html)
- [Check and Fix MySQL Replication Inconsistencies](http://www.fromdual.com/check-and-fix-mysql-replication-inconsistencies)

*Note: This post might contain some edits which can be tracked [here](https://github.com/mrafayaleem/mrafayaleem.github.io).*

---
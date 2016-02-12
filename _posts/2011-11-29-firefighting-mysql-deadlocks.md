---
layout: post
title: Firefighting MySQL Deadlocks
tags:
- databases
- deadlocks
- debugging
- MySQL
- SQL
---
I recently finished solving an issue where an `UPDATE` on
a particular row was timing out due to not being able to obtain a lock on that
row. For example, I was seeing the error:

<output title="Lock wait timeout error message">
Lock wait timeout exceeded; try restarting transaction
</output>

At a high level, an error like this occurs when some transaction is holding on
to a lock for longer than a specified interval (the lock timeout amount) while
another transaction is trying to obtain that lock. It's possible that it's just
a long-running query, but problems like this are usually due to a deadlock. The
process required to solve this problem was a bit nuanced to figure out, so
I've documented how to deal with this issue if it ever happens to you.

# Finding Long-Running Queries

From the MySQL client, run:

{% highlight sql %}
SELECT *
  FROM INFORMATION_SCHEMA.PROCESSLIST
  WHERE command = 'Query'
  ORDER BY time DESC;
{% endhighlight %}

This will list all connections which are currently executing a query, and list
the query as well. If any of these queries has a time column with a value
larger than 50 seconds (though to be honest, any query taking more than a
couple of seconds warrants a quick check to make sure it makes sense), then
you've probably found the problem. Look at the `SQL` being executed to see if
it makes sense for this query to take a long time. If you don't find anything
in this list, then chances are you just have a transaction that finished but
didn't release its lock (or something of that nature). In either case, execute:

{% highlight sql %}
SHOW ENGINE INNODB STATUS;
{% endhighlight %}

This will show you a list of currently running transactions at the bottom of the
output. If on multiple different executions you see any transactions which have
a lock on something, for example:

<output title="Example output of SHOW INNODB STATUS">
---TRANSACTION 173E830078, ACTIVE 31082 sec,
process no 9535, OS thread id 388972583 starting index read, thread declared inside InnoDB 6
mysql tables in use 4, locked 4
11614 lock struct(s), heap size 683328
MySQL thread id 6793410, query id 37057583802 db.example.com 10.10.10.12 app
</output>

From the above output, you should decide if killing it is necessary, or if
you can manually terminate the transaction via the client that initiated it.

# Killing a Query

Once you've identified the culprit, killing the transaction is a piece of cake.
Using the thread ID from the output of `SHOW ENGINE INNODB STATUS`, simply
execute `KILL 6793410` from the client, and the thread will be no more.

Note that this is simply a last resort once deadlocks have been detected.
Usually problems like this are indicative of a deeper problem (a race condition
on your client, for example), and you should investigate to make sure you
prevent the problem from ever happening again.

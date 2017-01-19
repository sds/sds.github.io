---
layout: post
title: The Great Migration
image: /images/posts/migrating-birds.jpg
tags:
- ops
---
# Moving a multi-million MAU site into the cloud in 3 weeks with minimal downtime and 40% savings

It was October 29th, 2014. We were making a routine visit to our datacenter in
south San Francisco, where the servers powering [Causes.com][CAU] were humming
along as always.

In the hallway, we ran into a network engineer who worked at the datacenter.

“So you guys are moving out today, then?” he asked.

Puzzled, we asked what prompted the question.

“Didn't you get the notice? Our provider was unable to renew the lease.
Everyone needs to be out of here by the end of the year”

This is not a situation any company wants to be in. We had two months to move a
multi-million MAU site with more than 180 million registered accounts (and all
the data that comes with it). We were also in the process of building out the
infrastructure for [Brigade][BRI], which [had acquired Causes][BAC] in June.
Panic was not a practical option, so we got to work.

Our team of three ops engineers came up with a plan and executed it. On
November 19th, about 3 weeks after we received the notice, we had Causes
running entirely within [Rackspace][RAC]'s cloud-based infrastructure, requiring
_less than two hours_ of accumulated downtime visible to users over that period.

Here's how we got there.

# Setting Expectations

With a migration of this size and urgency, we quickly met with the relevant
stakeholders to relax the constraints of the problem we faced; an important
tactic when facing a difficult challenge. We agreed that, if necessary:

* We were comfortable with a day or so of intentional downtime
* We could temporarily set services up in a non-[HA][HA] configuration (coming
  back later to improve them)
* We could drop support for old features and services (though we would need to
  propose each instance and discuss separately)

Respectively, the advantages were:

* Allowing for downtime enabled us to avoid taking extra steps to gracefully
  move some services over
* Rebuilding systems without HA redundancy incurs significantly less setup
  complexity
* Removing entire features (like unused employee-only analytics dashboards or
  low-visibility user-facing features) might save us from having to migrate
  large quantities of data or entire subsystems

# Creating a Plan

With the constraints clearly laid out, we now faced the task of figuring out
our approach. At a high level, there were two obvious courses of action we
could take:

* Physically move all the servers to another location, or
* Migrate everything to run alongside our existing infrastructure for Brigade
  in Rackspace.

The first option was potentially quicker; we could simply unplug the racks and
roll them to their new location (likely a different floor in the same
datacenter). However, it would require a complete shutdown of Causes for at
least a day, and that would be a best-case scenario. Furthermore, renegotiating
lease terms and maintaining the existing hardware (ranging in age from 2–7
years old) was undesirable and potentially risky.

The second option of migrating into Rackspace appealed to us for the following
reasons:

* **Utilize existing systems**: A number of the backend systems powering Causes
  were already running in Rackspace as part of Brigade’s infrastructure
  (services like [Graphite][GRA], [Sentry][SEN], etc.). We could piggy-back on
  these existing systems to save time.

* **Ditch the old Chef codebase**: Brigade and Causes infrastructure had
  separate [Chef][CHE] servers and thus a largely different set of cookbook
  code. A lot of the Causes cookbook code had been written before many Chef
  best practices had been popularized, which we had addressed when writing the
  Brigade cookbooks. Furthermore, switching to using the Brigade cookbooks
  would save us having to maintain two codebases.

* **Switch to community cookbooks**: For the cookbooks we hadn’t rewritten (since
  they were not services we used in Brigade), we could switch from using our
  custom-written Causes cookbooks to community cookbooks, giving us the benefit
  of using code actively maintained by others.

* **Upgrade all the things**: There were a number of software components we could
  potentially upgrade that would normally be difficult or time-consuming (and
  that had not been upgraded for that very reason). This included upgrading our
  version of [CentOS][CEN], [Percona MySQL][PER], [Redis][RED], and
  [Beanstalk][BEA], among other services and packages. This would also mean we
  would only need to maintain one version of the respective software across our
  entire infrastructure, easing the maintenance burden on our team. By building
  Causes separately in Rackspace, we essentially had a giant integration test
  where we could test that all of these upgrades worked before we did the
  switchover.

* **Minimize downtime**: We could build a copy of Causes in Rackspace while
  simultaneously leaving the existing Causes running in our data center,
  minimizing the amount of downtime we would need to take when switching over.

* **Reduce machine footprint**: We could consolidate a large number of our
  servers into fewer machines in Rackspace, as the specifications on their
  servers were better than what we were running in our datacenter. Furthermore,
  we could combine our multiple database partitions into one. This would
  drastically reduce our footprint from ~150 machines to under 30!

* **Save $**: Thanks to the reduced machine footprint, we could do all of
  this for less than it was currently costing to run Causes (on the order of
  40% cheaper).

With all that in mind, we opted to try moving everything into Rackspace, with
physically moving the machines as a backup plan if we saw ourselves unable to
finish on time. We set an aggressive goal of having everything ready for
switchover by December 1st, giving us enough time to re-evaluate the situation
before the end-of-year deadline.

# Executing the Plan

We didn’t take much time to make detailed plans, since we knew that ultimately the only way we would be able to tell if this would work would be if we tried it.

We completed the following set of tasks in this approximate order, parallelizing where possible (which was often the case):

1. **Set up a VPN tunnel from our datacenter to Rackspace.**
   This would allow us to replicate data from the old datacenter as we were
   building the infrastructure in Rackspace.

2. **Built MySQL slaves to replicate data from datacenter into Rackspace.**
   We had multiple database partitions, but focused on setting up replication
   on our _main_ partition---the name we used for the primary partition with most
   of the tables for the application; we had two additional partitions
   containing tables which were queried heavily in order to improve
   performance.

   We set the replication master host using the IP address of the main
   partition instead of the hostname. This allowed us to avoid getting DNS
   resolution working between the two domains, as each had their own
   nameservers configured by a separate Chef server.

   These slaves were also running a newer version of Percona (5.6 versus 5.5).
   The upgrade process simply required restoring a backup made via
   [`xtrabackup`][XTR] and executing [`mysql_upgrade`][MSU] once the server was
   running.

3. **Combined other partitions into the main partition on the live site.**
   This was done by syncing the tables from the other [partitions][PAR] to the
   main partition with Percona’s [`pt-table-sync`][PTS]. We were willing to
   temporarily forego the performance benefits of separate partitions for the
   convenience of having all data on a single partition, and the new server in
   Rackspace would be powerful enough to handle the full workload once we
   switched over anyway.

   This was accomplished for each partition in two passes: the first to copy
   the majority of the data over while web and async workers were still running
   (and thus still making changes to the original data, resulting in the copy
   straying from the original), and then a second after we stopped the workers
   so it could ensure they were both in sync.

   If we had stopped the workers before the first pass, we would have been down
   for hours at a time due to the large volume of data being synced.
   Introducing the second pass allowed us to minimize downtime, as only a
   fraction of the data would have been added/updated in the time between the
   end of the first pass and the shutting down of the workers in preparation
   for the second pass.

   We would then deploy a code change to start using the main partition instead
   of the old one, and then shut down the old partition. This process was
   repeated for each partition, with the downtime varying depending on the size
   of the data set, but was usually under 10 minutes.

4. **Upgraded the Beanstalk queues to the latest version.**
   This required us to [fix some issues][BSI] in the [`AsyncObserver`
   client][ASO] that the workers used to fetch jobs, but otherwise was pretty
   straightforward.

5. **Replaced our use of [`memcached`][MEM] with Redis.**
   This was relatively easy thanks to the abstraction provided by
   [`Rails.cache`][RLC], but switching from [DalliStore][DLS] to
   [RedisStore][RDS] changed the signature of `Rails.cache.increment` which
   required some adjustments to fix. This wasn’t in our original high-level
   plan, but we saw an opportunity to further reduce the number of
   technologies we would need to support going forward.

6. **Stood up a new Solr cluster from scratch.**
   Our old cookbook for provision [Solr][SLR] was written in-house and quite
   old, so we took the opportunity to switch to a [wrapper cookbook][WCB]
   using the [community Solr cookbook][CSC] while we were rebuilding the
   cluster.

   We weren’t able to import the indexes because they were sharded across
   multiple hosts, so we just reindexed everything with an async job, which
   took less than a day.
7. **Scrapped our custom deploy scripts in favor of Capistrano.**
   This gave us the benefit of unifying the deployment technology we used for
   all Ruby code in our infrastructure and also allowed us to take advantage
   of the large number of community plugins Capistrano provides for projects
   like [rbenv][RBE], [bundler][BLR], [Rails][RAI], [unicorn][UNI], etc.

8. **Iterated.**
   Throughout this process we developed on a separate `rackspace` branch in
   the Causes Rails app. This branch was what was deployed to Rackspace as we
   stood up the various services and tested them. We used copies of production
   data to give us better confidence that we had our ducks in a row without
   having to actually touch the live site.

   We managed to complete these steps far faster than we had originally
   anticipated. A lot of that was luck, but it was also thanks to the effort
   we had spent making it easy for us to provision new machines. The
   combination of [Rackspace's API][RSA] and the cookbooks we had just written
   for Brigade’s infrastructure made building each of the backend Causes
   services take hours rather than days.

# The Switchover

With a working copy of Causes now running in Rackspace using data replicated
from production, we were now ready to perform the actual switchover.

Our efforts had us at a point where we had a single MySQL server replicating to
a slave in Rackspace (which itself was replicating to another slave in
Rackspace for HA), so there was an up-to-date replica of all production data in
Rackspace. We didn’t need to pre-warm the new caches which meant the only
tricky part of the switchover was changing which host was the MySQL master.

We carried out the following steps:

1. **Stopped all Causes web and async workers.**
   This would stop any further writes from hitting the master.

2. **Waited for all writes to the current MySQL master to stop.**
   Verified there were no connections to the current master by running:

   ```
   SHOW FULL PROCESSLIST;
   ```

3. **Set current master to read-only.**
   ```
   SET GLOBAL read_only = 1;
   FLUSH TABLES WITH READ LOCK;
   ```

   This ensured we remained consistent and avoided the [split-brain][SBP]
   problem. We would have at most one master accepting writes at any time.

4. **Waited for the Rackspace slave to catch up to the master.**
   We waited until the log position in the output of:

   ```
   SHOW MASTER STATUS;
   ```

   ...stopped incrementing. At this point, the slave and the master were in
   sync.

5. **Made the Rackspace slave writable.**
   Executed the following on the slave:

   ```
   STOP SLAVE;
   RESET SLAVE ALL;
   SET GLOBAL read_only = 0;
   ```

   The slave was now the new master, as it was the only server able to accept
   writes.

6. **Start all web and async workers in Rackspace.**
   We had a commit ready that reconfigured the workers to connect to the newly
   promoted master, so deploying that commit and starting all the workers
   resulted in the new Causes application running entirely in Rackspace.

7. **Update public DNS to point to the new load balancers in Rackspace.**
   It would take time for existing DNS caches to expire, so we still needed a
   way to forward traffic heading to our old load balancer to our new one in
   Rackspace.

8. **Set up reverse proxies in the old datacenter.**
   These proxies would simply forward requests to the load balancer in
   Rackspace. This results in the site being slower for a while due to the
   datacenter hop, but is only a temporary measure until all existing DNS
   caches expired.

   We then monitored the proxy logs for the next hour while DNS updated.
   Eventually, requests stopped coming and we were good to shut the proxies
   down.

9. **Celebrate!**

# Closing Thoughts

This was a huge win for our team, not only because we finished way ahead of
schedule, but we also:

* Drastically reduced infrastructure-related technical debt and eliminated some
  services that we needed to maintain.
* Unified our entire infrastructure to be running the same (newer) version of
  all software.
* Minimized affecting our users by spreading out downtime over the 3 week
  process.
* Saved ourselves 40% on the monthly costs for running Causes.

We didn’t panic when presented with a nasty situation; we set aggressive
deadlines and worked hard to meet them; and we took calculated risks in order
to allow ourselves to move quickly but effectively.

Overall, it was a productive November.

[BRI]: https://www.brigade.com
[CAU]: https://www.causes.com
[BAC]: http://techcrunch.com/2014/06/11/sean-parkers-brigade-media-acquires-causes-in-its-quest-to-revitalize-american-democracy/
[RAC]: http://www.rackspace.com/
[HA]:  https://en.wikipedia.org/wiki/High_availability
[GRA]: http://graphite.wikidot.com/
[SEN]: https://github.com/getsentry/sentry
[CHE]: https://www.chef.io/
[CEN]: https://www.centos.org/
[PER]: http://www.percona.com/software/percona-server
[RED]: http://redis.io/
[BEA]: https://kr.github.io/beanstalkd/
[XTR]: http://www.percona.com/doc/percona-xtrabackup/2.2/
[MSU]: https://dev.mysql.com/doc/refman/5.6/en/mysql-upgrade.html
[PTS]: http://www.percona.com/doc/percona-toolkit/2.2/pt-table-sync.html
[PAR]: http://en.wikipedia.org/wiki/Partition_%28database%29
[BSI]: https://github.com/causes/async_observer/compare/31f18ab7...db988fee
[ASO]: https://github.com/causes/async_observer
[MEM]: http://memcached.org/
[RLC]: http://guides.rubyonrails.org/caching_with_rails.html
[DLS]: https://github.com/mperham/dalli
[RDS]: https://github.com/redis-store/redis-rails
[SLR]: http://lucene.apache.org/solr/
[WCB]: https://www.chef.io/blog/2013/12/03/doing-wrapper-cookbooks-right/
[CSC]: https://github.com/dwradcliffe/chef-solr
[RBE]: https://github.com/capistrano/rbenv
[BLR]: https://github.com/capistrano/bundler
[RAI]: https://github.com/capistrano/rails
[UNI]: https://github.com/tablexi/capistrano3-unicorn
[RSA]: https://github.com/opscode/knife-rackspace
[SBP]: http://en.wikipedia.org/wiki/Split-brain_%28computing%29

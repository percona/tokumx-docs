.. _sharded_cluster_offline_all:

===================================
Sharded Cluster Offline: All Shards
===================================

This guide explains how to do an offline data migration from basic |MongoDB| to |TokuMX|, for a sharded cluster, converting the existing data in |MongoDB| to |TokuMX|, converting the entire data set at once.

This allows you to migrate from a |MongoDB| cluster with some number of shards to a |TokuMX| cluster with a different number of shards, or to just start out with a more even balance of chunks, if the |MongoDB| cluster was unevenly balanced.

This guide can also be used to migrate from a |MongoDB| sharded cluster to a single |TokuMX| replica set. Since |TokuMX| already has document-level concurrency, high write throughput, and compression, it is reasonable to expect a single |TokuMX| replica set to support the workload of a sharded cluster of |MongoDB| instances, and migrating down to a single replica set reduces the operations work for maintaining the |TokuMX| database.

For other migration strategies, start with :ref:`tokumx_migration`.

Instructions
============

1. Stop writes

   Stop your application and turn off the balancer, and wait for any migrations to complete. The cluster should be idle.

   On a :program:`mongos` router:

   .. code-block:: javascript

     > sh.setBalancerState(false);
     > while (sh.isBalancerRunning()) {
     ...     print('waiting...');
     ...     sleep(1000);
     ... }

2. Take backups

   Back up each database in the data set by using :program:`mongodump` to connect to one of the :program:`mongos` routers. Do not back up the config database.

   For each database *$db*:

   .. code-block:: bash

     $ mongodump --host mongos0.domain --db $db --out /var/lib/mongodb-$db.backup

6. Uninstall |MongoDB|

   Uninstall basic |MongoDB| from all shard servers and router servers.

   You can also remove the old :variable:`dbpath` from all shards since you have a backup of the data.

   Keep the config servers running for reference.

4. Set up |TokuMX|

   Set up an empty |TokuMX| sharded cluster. Deploying a sharded cluster in |TokuMX| is the same as it is in basic |MongoDB|.

5. Enable sharding

   For each database that was sharded, enable sharding for that database and then enable sharding for each collection with ``sh.shardCollection()``.

   It is a good idea to pre-split all sharded collections so that the import phase loads data evenly. You can use the basic |MongoDB| config database as a guideline, or just pre-split manually. After pre-splitting, allow the balancer to finish moving chunks while they are empty before loading data.

6. Import your backup

   Import the backup of all the non-config databases with :program:`mongorestore` through the |TokuMX| :program:`mongos` routers.

   For each database *$db*:

   .. code-block:: bash

     $ mongorestore /var/lib/mongodb-$db.backup

7. Shut down the |MongoDB| config servers

   Shut down the |MongoDB| config servers, and remove the :variable:`dbpath` for them.

Special Performance Considerations
----------------------------------

With an offline migration through the :program:`mongos` router, the |TokuMX| bulk loader is not used, so the import phase will be slower than importing data directly into :program:`mongod`. This slowness can be mitigated by using multiple concurrent import clients. The work can be divided on a per-db or per-collection basis.

Furthermore, it is possible to take offline backups of each shard individually, and then import them into a running empty |TokuMX| sharded cluster in parallel.

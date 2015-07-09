.. _tokumx_migration:

=====================
 Migrating to TokuMX
=====================

Overview
TokuMX stores all data on disk differently from basic MongoDB, this is key to its performance benefits. Therefore, TokuMX cannot use existing MongoDB data files—all MongoDB data must be migrated to the TokuMX data format.

There are many strategies for migrating data, and choosing the right one depends on your current MongoDB installation and your application's availability requirements. Single servers, replica sets, and sharded clusters can all be converted to TokuMX but the process is different for each.

Strategies
The following guides describe the possible migration strategies:

Single Server
Replica Set Offline
Replica Set Online
Sharded Cluster Offline: Individual Shards
Sharded Cluster Offline: All Shards
Sharded Cluster Online
Offline vs. Online Migration
For replica sets and sharded clusters, you have a choice between offline and online migrations.

If your application can afford downtime, an offline migration is the best choice, because it will be simpler and faster overall.

If an offline migration will take too long for your application to afford, then you can do an online migration. These procedures require that basic MongoDB is running with replication, because a major part of the migration process involves tailing the MongoDB oplog and replaying operations on the TokuMX cluster to keep the clusters in sync until the application can be switched over.

Sharded Cluster Options
When migrating a sharded cluster to TokuMX, there are three main strategies to consider:

Sharded Cluster Offline: Individual Shards
Sharded Cluster Offline: All Shards
Sharded Cluster Online
Just like for replica sets, sharded clusters can be migrated "offline" or "online".

When migrating a sharded cluster offline, it is possible to convert each shard individually, or to convert the entire data set at once.

Individual Shards

When converting each shard individually, you will end up with the same number of shards and chunk distribution that you had in MongoDB. The benefits here are:

The migration process is faster because each shard can be migrated in parallel, and all shards can use the bulk loader, which imports data faster.

Each shard is backed up to disk separately, so the space used by the backup is spread over many disks.

All Shards

When converting the entire data set at once, you have the opportunity to change the number of shards and the chunk distribution, but this method does not use TokuMX’s fast bulk loader, so will typically be a slower process. Converting the entire data set at once also can require a lot of extra disk space because the entire data set needs to be dumped to disk at once.

When the entire data set is dumped to disk in one backup, it is also possible to import this backup into a single TokuMX replica set. Since TokuMX already has document-level concurrency, high write throughput, and compression, it is reasonable to expect a single TokuMX replica set to support the workload of a sharded cluster of MongoDB instances, and migrating down to a single replica set reduces the operations work for maintaining the TokuMX database.

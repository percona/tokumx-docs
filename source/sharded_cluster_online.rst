.. _sharded_cluster_online:

======================
Sharded Cluster Online
======================

This guide explains how to do an online data migration from basic |MongoDB| to |TokuMX|, for a sharded cluster, converting the existing data in |MongoDB| to |TokuMX| on a shard-by-shard basis

For other migration strategies, start with :ref:`tokumx_migration`.

Instructions
============

1. Stop the balancer

   Turn off the balancer, and wait for any migrations to complete.

   On a :program:`mongos` router:

   .. code-block:: javascript

     > sh.setBalancerState(false);
     > while (sh.isBalancerRunning()) {
     ...     print('waiting...');
     ...     sleep(1000);
     ... }

2. Migrate shards

   Perform an online migration of each shard individually as a replica set, until all shards are caught up.

   Learn how in :ref:`replica_set_online`.

   .. tip:: 
     These migrations can be done concurrently.

3. Keep syncing

   Leave one instance of :program:`mongo2toku` running from each |MongoDB| shard to its corresponding |TokuMX| shard.

4. Shut down a config server

   Shut down one of the |MongoDB| config servers.

   This will not stop application reads and writes, but it will stop splits and chunk migrations from completing.

5. Migrate the stopped config server

   Perform an offline migration of the shut down config server to |TokuMX|, and import its backup once for each |TokuMX| config server, to bring the set of |TokuMX| config servers online.

   .. note:: 
     See :ref:`sharded_cluster_offline_individual` for details on how to handle shard and config server hostname changes when migrating |MongoDB| sharded cluster config data.

6. Start |TokuMX| routers

   Start |TokuMX| :program:`mongos` routers on the :program:`TokuMX` config servers.

   In the last step, the config data should have been modified so that the new :program:`mongos` routers can find the |TokuMX| shards, you can check ``sh.status()`` at this point on the |TokuMX| cluster to verify that everything is working normally.

7. Pause and wait for full sync

   Pause your application's writes and wait for all :program:`mongo2toku` processes to report that they are "fully synced".

8. Switch your application to |TokuMX|

   Redirect your application to the |TokuMX| :program:`mongos` routers.

9. Clean up and tear down basic |MongoDB|

   Once your application is running on the |TokuMX| cluster, you can stop all the :program:`mongo2toku` processes, the remaining |MongoDB| config servers, the |MongoDB| shard servers, and the |MongoDB| router servers, delete the |MongoDB| ``dbpaths``, and shut down those machines.



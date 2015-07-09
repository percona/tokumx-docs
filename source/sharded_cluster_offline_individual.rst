.. _sharded_cluster_offline_individual:

==========================================
Sharded Cluster Offline: Individual Shards
==========================================

This guide explains how to do an offline data migration from basic |MongoDB| to |TokuMX|, for a sharded cluster, converting the existing data in |MongoDB| to |TokuMX| on a shard-by-shard basis.

For other migration strategies, start with :ref:`tokumx_migration`.

Instructions
============

1. Shut down |MongoDB|

   Shut down the existing |MongoDB| servers on each machine in the sharded cluster (shards, config servers, and routers) to make sure you get a consistent backup.

2. Migrate shards individually

   Migrate each shard to |TokuMX| offline, following the guide :ref:`replica_set_offline`. You can do this for each shard concurrently.

3. Back up a config server

   Back up one of the |MongoDB| config servers with :program:`mongodump`.

   You will need the :variable:`dbpath` from :file:`/etc/mongodb.conf` (this is often :file:`/var/lib/mongodb`) and you will need to choose a location for the backup (here, :file:`/var/lib/configdb.backup`).

   One config server only:

   .. code-block:: bash

   $ sudo mongodump --dbpath /var/lib/mongodb --out /var/lib/configdb.backup

4. Uninstall |MongoDB| from config servers

   Uninstall |MongoDB| from the config servers. You can also remove the old :variable:`dbpath` since you have a backup.

5. Install |TokuMX| on config servers

   Install |TokuMX| :ref:`from tarballs <installation>` or :ref:`from packages <installation_from_packages>` on the config servers and make sure it starts properly.

   Take a moment to transfer any important configuration from your basic |MongoDB| configuration file (usually :file:`/etc/mongodb.conf`) to your |TokuMX| configuration file (usually :file:`/etc/tokumx.conf`). If you do this, be sure to restart the config servers.

6. Import your backup of the config servers

   Import the backup to each config server with :program:`mongorestore`.

   .. warning:: 
     If you are running with the ``configsvr`` option, remember that this makes the port ``27019`` instead of the default.

   On the machine with the backup:
   
   .. code-block:: bash

     $ mongorestore --host localhost:27019 /var/lib/configdb.backup
     $ mongorestore --host cfg2.domain:27019 /var/lib/configdb.backup
     $ mongorestore --host cfg3.domain:27019 /var/lib/configdb.backup

7. Configure routers

   Copy any relevant configuration from :file:`/etc/mongodb.conf` to :file:`/etc/tokumx.conf` on all :program:`mongos` router machines.

   .. note::
     If your config servers have different hostnames now, you will need to update the ``configdb`` settings for all your :program:`mongos` configurations. See `Migrate Config Servers with Different Hostnames <http://docs.mongodb.org/manual/tutorial/migrate-config-servers-with-different-hostnames/>`_ for more details.

   If your shard servers have different hostnames now, you will need to update their hostnames in the config servers' databases. Connect to the config servers and update the shards' metadata.

   On config servers:

   Connect to the config servers with a connection string similar to what's used for the :program:`mongos` ``--configdb`` option:
   
   .. code-block:: bash

     $ mongo localhost:27019,cfg2.domain:27019,cfg3.domain:27019/config

   Update the metadata:
   
   .. code-block:: javascript

     > db.shards.update({_id: <shard name>},
          {$set: {host: "<replset name>/<hostnames>"}})

8. Start |TokuMX| routers

   Start the |TokuMX| :program:`mongos` server on all router machines, and start your application.



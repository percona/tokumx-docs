.. _replica_set_online:

=====================================
Replica Set Online (without downtime)
=====================================

An online migration is done by having |TokuMX| sync from basic |MongoDB| until the application can be switched over to |TokuMX|. This allows the application to continue operating normally with |MongoDB| until the very instant it is switched over to |TokuMX|.

This is the main benefit of an online migration, however, the drawbacks of an online migration are:

* The migration process is more complicated and can take longer, as |TokuMX| will need to catch up with the running |MongoDB| database, which will have processed more writes while |TokuMX| was bulk importing data.

* The migration requires that a |MongoDB| replica set and a |TokuMX| replica set be online simultaneously. While it is possible to put the |TokuMX| instances on the same physical or virtualized machines that are running |MongoDB|, |TokuMX| will use some of the system resources, so this is dangerous for |MongoDB| databases that are running close to their machine's maximum capacity.

Instructions
============

You will need to prepare by provisioning a number of extra machines where you will build your |TokuMX| replica set.

The online migration of a replica set is broken down into four phases:

1. :ref:`backup`
2. :ref:`import`
3. :ref:`catchup`
4. :ref:`switch`

.. _backup:

Backup
------

.. tip::
  If you already have a backup with a known ``OpTime``, you can skip the first steps that take a backup of a secondary.

1. Shut down a secondary

   Shut down one of the |MongoDB| secondaries.

2. Take a backup

   Take a backup of that secondary with :program:`mongodump`.

   Secondary:

   .. code-block:: bash

     $ sudo mongodump --dbpath /var/lib/mongodb --out /var/lib/mongodb.backup

3. Record the OpTime

   While the secondary is down, connect to the primary and find out the ``OpTime`` that the secondary was at when it was shut down. Look for the ``OpTime`` for the machine in the members array that is ``"(not reachable/healthy)"``.

   Primary:

   .. code-block:: javascript

     rs:PRIMARY> rs.status()
     {
         "set" : "rs",
         "date" : ISODate("2013-07-13T18:19:04Z"),
         "myState" : 1,
         "members" : [
            ...,
            {
                "_id" : 2,
                "name" : "cavil:20002",
                "health" : 0,
                "state" : 8,
                "stateStr" : "(not reachable/healthy)",
                "uptime" : 0,
                "optime" : {
                   "t" : 1373658837,
                   "i" : 999
                },
                ...
            }
        ],
        "ok" : 1
     }

   Save this OpTime along with the backup: {t: 1373658837, i: 999}.

4. Restart the secondary

   Start the secondary back up and let it catch up with its replica set.

.. _import:

Import
------

You need to import this backup into a |TokuMX| replica set. The latter half of the :ref:`offline migration <replica_set_offline>` process applies here:

1. Install |TokuMX|

   Install |TokuMX| :ref:`from tarballs <installation>` or :ref:`from packages <installation_from_packages>` on all machines. If your distribution's package automatically starts |TokuMX|, stop it for now.

   Take a moment to transfer any important configuration from your basic MongoDB configuration file (usually /etc/mongodb.conf) to your TokuMX configuration file (usually /etc/tokumx.conf).

2. Import your backup

   Import your backup to just the primary with :program:`mongorestore`.

   You will need the :variable:`dbpath` from :file:`/etc/tokumx.conf` (by default, :file:`/var/lib/tokumx`).

   Primary server only:

   .. code-block:: bash

     $ mongorestore --dbpath /var/lib/tokumx /var/lib/mongodb.backup

3. Configure replication

   Add the ``replSet`` option to :file:`/etc/tokumx.conf` on all machines, for example, ``replSet=rs0``.

4. Initialize replication

   Start the primary, connect to it, and run ``rs.initiate()`` and then shut it down, to initialize the oplog.

   Primary server only:

   .. code-block:: javascript

     > rs.initiate()
     {
         "info2" : "no configuration explicitly specified -- making one",
         "me" : "db1.localdomain:27017",
         "info" : "Config now saved locally.  Should come online in about a minute.",
         "ok" : 1
     }
     >
     rs0:PRIMARY> db.shutdownServer()

5. Copy data

   With the primary server shut down, copy the :variable:`dbpath` to all the secondaries.

   The data is already compressed, so compressing with :program:`rsync` will not be faster, and it will be much faster than a normal initial sync.

6. Start secondaries

   Add the ``fastsync=true`` option to :file:`/etc/tokumx.conf` on all secondaries, then start all of them.

   Alternatively, you can start the servers manually and add the ``--fastsync`` option on the command line.

7. Add secondaries to the set

   Start the primary, connect to it, and rs.add() each of the secondaries.

   With ``fastsync`` they will not need to do a full initial sync.

   Primary server only:

   .. code-block:: javascript

     rs0:PRIMARY> rs.add('db2.domain:27017')
     { "ok" : 1 }
     rs0:PRIMARY> rs.add('db3.domain:27017')
     { "ok" : 1 }

8. Clean up configs

   Remove the ``fastsync=true`` option from :file:`/etc/tokumx.conf` on each of the secondaries. You do not need to restart them now.


.. _catchup:

Catchup
-------

This phase uses the :program:`mongo2toku` tool to catch the |TokuMX| replica set up with the |MongoDB| replica set by replaying the |MongoDB| oplog.

Below, suppose the |MongoDB| replica set is identified by ``mongodb/v1.domain,v2.domain,v3.domain`` and the |TokuMX| replica set is identified by ``tokumx/t1.domain,t2.domain,t3.domain``.

1. Sync with :program:`mongo2toku`

   Using the ``OpTime`` recorded earlier, start :program:`mongo2toku`. This will read the basic |MongoDB| oplog and replay its operations on the |TokuMX| replica set, but the |TokuMX| replica set will not count for write concern on the |MongoDB| set, and it will not be able to vote in elections on the |MongoDB| set.

   On a TokuMX server:

   .. code-block:: bash
    
      $ mongo2toku --ts=1373658837:999 \
        --from mongodb/m1.domain,m2.domain,m3.domain \
        --host tokumx/t1.domain,t2.domain,t3.domain

Let mongo2toku run until it is fully caught up to (or only a few seconds behind) the MongoDB replica set.

.. note::
  Feel free to stop and start it, taking in to consideration the :ref:`notes_on_mongo2toku`.

Once :program:`mongo2toku` is fully synced, it will continue to tail the |MongoDB| oplog until stopped, so you can leave it running and keep the |TokuMX| replica set synced until it is appropriate to switch over your application.

.. _switch:

Switch
------

When you are ready to switch your application over to |TokuMX|, leave :program:`mongo2toku` running through the entire process.

1. Pause your application

   Pause your application's writes to the |MongoDB| replica set.

2. Wait for full sync

   Wait until :program:`mongo2toku` reports that it is "fully synced" a few times to the log.

3. Switch application to |TokuMX|

   Redirect your application to point to the |TokuMX| replica set.

4. Shut down :program:`mongo2toku`

   Shut down :program:`mongo2toku` (``Control-C`` in the controlling terminal will shut it down cleanly.)

At this point, you can delete the basic |MongoDB| data and shut down the machines it was running on.

.. _notes_on_mongo2toku:

Notes on :program:`mongo2toku`
------------------------------

It is important that :program:`mongo2toku` does not try to replay an operation twice.

If there is an error, or if :program:`mongo2toku` is stopped manually, :program:`mongo2toku` will save a file in the current directory with the ``OpTime`` it had reached while syncing, and will also print that value to the console.

When resuming, it will assume that operation was replayed and will try to replay the following operation. If you do not provide ``--ts`` when restarting :program:`mongo2toku` it will use the value saved in the file, but ``--ts`` can override it.

Advanced Topics
===============

Testing Workloads
-----------------

The :program:`mongo2toku` tool is simple: it reads the oplog from one basic |MongoDB| server or set of servers and replays those operations on another |MongoDB| or |TokuMX| server or set of servers. Therefore, it can be used to test TokuMX's suitability for handling a write workload that |MongoDB| is running, without making the application rely on |TokuMX| for queries.

To do this, you would follow the above guide until |TokuMX| was caught up, then just leave :program:`mongo2toku` running and monitor system load and metrics, response latencies, and anything else important for your workload. You could also try building additional background indexes on |TokuMX|.

Since :program:`mongo2toku` just looks like a normal client application to the |TokuMX| server, you can run multiple instances copying data to the same |TokuMX| replica set. Therefore, it is possible to simulate the result of running a |MongoDB| sharded cluster workload on a single |TokuMX| replica set. For details on how to migrate a |MongoDB| sharded cluster to a single |TokuMX| replica set, see :ref:`sharded_cluster_offline_all`.

Rolling Migration
-----------------

The online migration procedure typically requires approximately double the machine capacity since there need to be two replica sets running concurrently: |MongoDB| and |TokuMX|.

However, it is possible to do an online migration with little or no extra capacity. This process is more complicated and takes longer, but is suitable for applications with limited resources.

The process is site-specific, but this is the general procedure:

1. Take a backup

   Begin as above, by taking a backup and noting the ``OpTime`` of that backup.

2. Clean one secondary

   Instead of starting the secondary back up, remove it from the |MongoDB| replica set and delete its :variable:`dbpath`.

   .. note:: 
     If necessary, add more arbiters to the MongoDB replica set, since it has one fewer members now.

3. Set up one |TokuMX| server

   Import the backup into |TokuMX| on the machine that was just cleaned. Turn it into a replica set (set ``replSet`` and run ``rs.initiate()``) and add some arbiters to that replica set.

   .. tip::
     (Optional) Shut down that TokuMX instance once its oplog is initialized, and copy its dbpath somewhere else, to be used to start other TokuMX secondaries later. Then start it again.

4. Begin syncing

   Run :program:`mongo2toku` as before, from the |MongoDB| replica set to the |TokuMX| replica set.

5. Clean another secondary

   Select another |MongoDB| secondary, remove it from the replica set, and delete its :variable:`dbpath`.

6. Set up another |TokuMX| server

   Install |TokuMX| to that machine, and either add it to the |TokuMX| replica set and let it complete an initial sync, or if the original |TokuMX| :variable:`dbpath` was saved above, copy that to the new server's :variable:`dbpath` and start the server with ``fastsync`` to avoid the initial sync (see the guide for :ref:`replica_set_offline` for more details on ``fastsync``).

   Once this machine joins the replica set, it will catch up to the synced primary.

7. Convert the remaining servers

   Continue taking machines out of the |MongoDB| replica set and replacing them with |TokuMX| machines until there are enough |TokuMX| machines to handle the full application workload. During this process, any application queries that can normally be done on secondaries (with ``slaveOk``) can be redirected to the |TokuMX| replica set if it is suitably well synced.

8. Switch application to |TokuMX|

   When the |TokuMX| replica set is full enough to accept the full application workload, switch the application over to it (remember to wait for :program:`mongo2toku` to be "fully synced" after pausing the application), and then tear down the remaining |MongoDB| machines and replace them with |TokuMX| machines if needed.

Testing the Migration Process
-----------------------------

The ``dbHash`` command can be run on any database and will produce a hash value that is dependent on all the documents in that database. After a successful migration, it will produce the same results on |MongoDB| as on |TokuMX|, so it can be used to verify that a migration process was implemented successfully.

Just remember to make sure |MongoDB| is not accepting writes while ``dbHash`` is running, and make sure that :program:`mongo2toku` says it is "fully synced" before running ``dbHash`` on |TokuMX|.

The dropDups option
-------------------

When creating a unique index in |MongoDB|, it is possible to add the option ``dropDups``. This is an arbitrarily destructive operation, so it was not implemented in |TokuMX|. Even if it were implemented, there is no way for |TokuMX| to be sure that it dropped the same documents that |MongoDB| would have dropped.

Therefore, if an ``ensureIndex`` with ``{dropDups: true}`` is encountered by :program:`mongo2toku`, it will attempt to build the index without that option, but if there actually are duplicates, that index build will fail.

If it fails, the only option is to restart the migration process by taking a new backup. That backup will now be after the index creation happened, so you won't need to process the same index build operation again.

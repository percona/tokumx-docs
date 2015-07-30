.. _replica_set_offline:

===================================
Replica Set Offline (with downtime)
===================================

An offline migration requires that the application be down while the migration takes place, which is a dump from basic |MongoDB| and an import into |TokuMX|. The benefits to an offline migration are:

* The migration process is simple and fast. You simply dump the |MongoDB| data and restore it into |TokuMX|, then copy the restored dbpath to all secondaries.

* The migration can be done on the existing machines provisioned for |MongoDB|, no additional servers or capacity is required.

The drawback of an offline migration is that it requires a maintenance window that can be very long for large data sets.

Instructions
============

1. Shut down |MongoDB|

   Shut down the existing |MongoDB| servers on each machine in the replica set to make sure you get a consistent backup.

2. Back up the primary |MongoDB| database

   Back up the primary |MongoDB| database with :program:`mongodump`.

   For MongoDB 3.0.0 and later versions, use the following command to back up to :file:`/var/lib/mongodb.backup`:

   .. code-block:: bash

     $ sudo mongodump --out /var/lib/mongodb.backup

   For MongoDB versions prior to 3.0.0, you will also need the :variable:`dbpath` from your command-line options or :file:`/etc/mongodb.conf` (this is often :file:`/var/lib/mongodb`):

   .. code-block:: bash 

     $ sudo mongodump --dbpath /var/lib/mongodb --out /var/lib/mongodb.backup

3. Uninstall |MongoDB|

   Uninstall MongoDB from all machines. You can also remove the old dbpath since you have a backup.

4. Install |TokuMX|

      Install |TokuMX| :ref:`from tarballs <installation>` or :ref:`from packages <installation_from_packages>` on all machines. If your distribution's package automatically starts |TokuMX|, stop it for now.

      Take a moment to transfer any important configuration from your basic |MongoDB| configuration file (usually :file:`/etc/mongodb.conf`) to your |TokuMX| configuration file (usually :file:`/etc/tokumx.conf`).

5. Import your backup

   Import your backup to just the primary with :program:`mongorestore`.

   You will need the :variable:`dbpath` from :file:`/etc/tokumx.conf` (by default, :file:`/var/lib/tokumx`).

   Primary server only:

   .. code-block:: bash

     $ mongorestore --dbpath /var/lib/tokumx /var/lib/mongodb.backup

6. Configure replication

   Add the ``replSet`` option to :file:`/etc/tokumx.conf` on all machines, for example, ``replSet=rs0``.

7. Initialize replication

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

8. Copy data

   With the primary server shut down, copy the :variable:`dbpath` to all the secondaries.

   The data is already compressed, so compressing with :program:`rsync` will not be faster, and it will be much faster than a normal initial sync.

9. Start secondaries

   Add the ``fastsync=true`` option to :file:`/etc/tokumx.conf` on all secondaries, then start all of them.

   Alternatively, you can start the servers manually and add the ``--fastsync`` option on the command line.

10. Add secondaries to the set

    Start the primary, connect to it, and ``rs.add()`` each of the secondaries.

    With ``fastsync`` they will not need to do a full initial sync.

    Primary server only:

    .. code-block:: javascript

      rs0:PRIMARY> rs.add('db2.domain:27017')
      { "ok" : 1 }
      rs0:PRIMARY> rs.add('db3.domain:27017')
      { "ok" : 1 }

11. Clean up configs

    Remove the ``fastsync=true`` option from :file:`/etc/tokumx.conf` on each of the secondaries. You do not need to restart them now.


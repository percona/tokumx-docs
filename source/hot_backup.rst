.. _hot_backup:

==========
Hot Backup
==========
TokuMX users have a few options for backing up data:

* Use :program:`mongodump`/:program:`mongorestoreas` used in basic |MongoDB|.

 .. warning::
  The TokuMX versions of these tools do not support the --oplog and --oplogReplay options due to oplog format changes.

* Back up all data files with a `filesystem-level snapshot such as LVM or EBS snapshots <http://docs.mongodb.org/manual/tutorial/backup-with-filesystem-snapshots/>`_, or `xfs_freeze(8) <http://linux.die.net/man/8/xfs_freeze>`_.

 If the server is using a separate :variable:`logDir` from the ``dbpath``, both directories must be backed up, and they must be at a consistent point in time. Enterprise Hot Backup provides this consistency automatically.

 .. note::
  As |TokuMX| has always-on transaction logging (equivalent to the basic |MongoDB| journal), ``db.fsyncLock()`` is unnecessary. In fact, because filesystem writes can happen in the background, ``db.fsyncLock()`` would not provide the guarantees it provides in basic |MongoDB|, and therefore is disabled to prevent accidental misuse.

* |TokuMX| Enterprise Edition includes Hot Backup, which creates a physical data backup on a running server without performance degradation or capacity planning, and provides a more recent backup than a filesystem-level snapshot.

Concepts
========
A Hot Backup copies the data in the ``dbpath`` (and :variable:`logDir` if different) to a backup directory. A Hot Backup has two subsystems for completing the backup:

* One component copies files directly, in the background. This component's I/O consumption can be controlled with the :command:`backupThrottle` command.

* Another component mirrors writes performed by the Fractal Tree indexing library on behalf of clients. Any writes to files in the ``dbpath`` are duplicated to their corresponding backup files, for the duration of the backup.

Since all writes to TokuMX files are duplicated while the backup is running, the state of the files in the backup directory is identical to the main files at the time the backup completes. Compare this with backups performed with filesystem-level snapshots, where the backup data is as it was at the beginning of the backup.

Another effect of mirrored writes is that the write I/O done by clients is doubled for the duration of the backup. As Fractal Tree indexes are write-optimized, this is usually not a problem, but is important to know for capacity planning.

For more details on the internals of Enterprise Hot Backup, there is a series of blog posts describing its design and implementation:

* TokuDB Hot Backup - Part 1

* TokuDB Hot Backup - Part 2

* TokuDB Hot Backup - Part 3


Loading the Plugin
==================
To use Enterprise Hot Backup, ensure that the ``backup_plugin`` plugin is loaded into the server. Before loading, :file:`libbackup_plugin.so` must be in the :variable:`pluginsDir`. The tarballs as :ref:`unpacked during installation <tarball_installation>` and the :ref:`deb and rpm packages <installation_from_packages>` set up these paths and settings correctly by default.

To load the Hot Backup plugin into a running :program:`mongod` server, use the :variable:`loadPlugin` command.

Example:

.. code-block:: javascript

  > db.adminCommand({loadPlugin: 'backup_plugin'})
  {
    "loaded" : {
        "filename" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
        "fullpath" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
        "name" : "backup_plugin",
        "version" : "tokubackup 1.0",
        "checksum" : "c22e35ed8466a01a6cc979f409b4b1a5",
        "commands" : [
            "backupStart",
            "backupThrottle",
            "backupStatus"
        ]
     },
     "ok" : 1
   }

                                                                                                
.. warning:: 
  The running :program:`mongod` must be an Enterprise Edition build of TokuMX. Unlike the `pitr_plugin` plugin, the backup plugin cannot be loaded into a Community Edition :program:`mongod`.

Autoloading the Plugin
======================
To automatically load the Hot Backup plugin on server startup, use the :variable:`loadPlugin` server parameter:

On the command line:

.. code-block:: bash

  $ mongod --loadPlugin=backup_plugin:c22e35ed8466a01a6cc979f409b4b1a5

In the config file:

.. code-block:: bash

  loadPlugin=backup_plugin:c22e35ed8466a01a6cc979f409b4b1a5

.. note::  
  The checksum will be different in different versions of |TokuMX|. You can discover the checksum for your plugin by loading it once with the :variable:`loadPlugin` command.

.. _using_pitr:

Basic Usage
===========
The Enterprise Hot Backup plugin adds three commands to a running :program:`mongod`. For details, see the section on :ref:`hot_backup_commands`.

 * :command:`backupStart` initiates a Hot Backup procedure, and blocks until the backup is complete.

 * :command:`backupStatus` is used to query the Hot Backup system for its progress during a backup.

 * :command:`backupThrottle` sets a bytes-per-second rate limit on the I/O used for a Hot Backup

Creating New Replicas
=====================
A great use case for Hot Backup is creating new secondaries in a replica set.

The normal initial sync procedure can uses normal queries that need to decompress and deserialize data on disk, and then marshall it and send it across the network, then on the secondary, it needs to be indexed, serialized, and compressed all over again. This is a slow process, and furthermore it poisons the cache of the machine being synced from with data that may be irrelevant to the application.

Instead, a Hot Backup can be used to initialize a replica set secondary. This is both faster and less intrusive to application queries and the sync source server's cache.

To create a secondary using Hot Backup, simply move the backup files to the new machine, start the server with the ``--replSet`` option and additionally with ``--fastsync``, then use `rs.add() <http://docs.mongodb.org/manual/reference/method/rs.add/>`_ on the primary to add the new secondary. After the secondary has been added, you should remove the ``--fastsync`` option for future server startups.

.. warning::
  In order to find the oplog position in common between the new secondary and the existing members of the set, the oplog must be present in the Hot Backup. Therefore, when initially creating a replica set from a single server, it is necessary to run `rs.initiate() <http://docs.mongodb.org/manual/reference/method/rs.initiate/>`_ first before taking a backup for the new secondary.

.. tip::
  To minimize impact on a running application, it is recommended to use a backup of an existing secondary to create a new secondary, rather than backing up the primary.

Sharding
========
Since a Hot Backup captures the state of a server at the end of the backup operation, it can be difficult to capture a time-consistent backup of multiple shards simultaneously.

The recommended procedure for taking a backup of a sharded cluster in |TokuMX| is to disconnect one secondary from each shard at the same time, then back up those secondaries with any backup procedure. Additionally, one config server must be backed up at the same time as well.

For most applications, getting a truly consistent backup of a sharded cluster requires that the application pause all writes and the `balancer <http://docs.mongodb.org/manual/tutorial/manage-sharded-cluster-balancer/#disable-the-balancer>`_, wait for one secondary on each shard to catch up fully with the primary, then disconnect one config server and a secondary from each shard. After this, the application can continue (and the balancer as well, once the config server has been backed up), and when the backup is finished, the secondaries will need to catch up again.

Using Point in Time Recovery
============================
For an approximate snapshot of a sharded cluster, Point in Time Recovery can be used to get a backup of each shard up to approximately the same point in time. With a sharded cluster backup using PITR, it is important to make sure the config server backup is synchronized with the shard backups.

To use Hot Backup and Point in Time Recovery to take a sharded backup, first take a hot backup of one member of each shard. Then, stop the balancer and take a backup of a config server. Note the time when the config server is backed up, then use PITR to sync the backup of each shard to this timestamp.

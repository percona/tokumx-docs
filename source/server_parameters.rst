.. _server_parameters:

===================
 Server Parameters
===================

.. _new_and_modified_server_parameters:

New and Modified Server Parameters
==================================

.. variable:: auditDestination

Supported since 2.0.0

Has no effect in |TokuMX| Community Edition. See :ref:`audit` for more details.

The type of audit log that will be created. The only currently supported value is ``file``.

.. variable:: auditFilter

  :default: ``{}``

Supported since 2.0.0

Has no effect in |TokuMX| Community Edition. See :ref:`audit` for more details.

A JSON object used to filter audit events. Should be a valid |MongoDB| query; only audit events that match this query will be logged. By default, all events are logged.

.. variable:: auditFormat

Supported since 2.0.0

Has no effect in |TokuMX| Community Edition. See :ref:`audit` for more details.

The format used for audit events. The only currently supported value is JSON.

.. variable:: auditPath

Supported since 2.0.0

Has no effect in |TokuMX| Community Edition. See :ref:`audit` for more details.

For the file :variable:`auditDestination`, the filename used for the audit log. By default, this is :file:`auditLog.json` in the same directory as ``logpath`` if it exists, otherwise inside ``dbpath`` for :program:`mongod` or inside the current working directory for :program:`mongos`.

.. variable:: cacheSize

  :default: 1/2 of physical RAM, computed at startup.

Amount of memory (in bytes) used to cache documents and indexes.

Can supply values using K/M/G (i.e., '4G' = 4 Gigabytes)

.. variable:: checkpointPeriod

  :default: 60 seconds.

This variable specifies the number of seconds between checkpoints.

This parameter actually controls the minimum number of seconds between the start of consecutive checkpoints. If a checkpoint takes longer than 60 seconds, the subsequent checkpoint will begin immediately upon its predecessor's completion.

.. variable:: cleanerIterations

  :default: 5

This variable specifies how many internal nodes get processed in each cleaner thread run.

Setting this variable to 0 turns off cleaner threads.

.. variable:: cleanerPeriod

  :default: 2 seconds

This variable specifies the number of seconds between cleaner thread runs.

Setting this variable to 0 turns off cleaner threads.

.. variable:: dbpath

  :default: :file:`/data/db`

Directory where the |TokuMX| collections (including the oplog for replication) are stored.

.. variable:: directio

  :default: false

Use direct I/O instead of buffered I/O.

Out of memory workloads typically perform better with direct I/O.

When using direct I/O, we recommend a :variable:`cacheSize` of 80% physical RAM or more.

.. variable:: expireOplogDays / expireOplogHours

  :default: 14 days

Number of days’ worth of data to keep in the primary’s ``local.oplog.rs``. This is effectively how far behind a secondary can get before it needs to do a new initial sync when it gets re-integrated into the set.

This parameter only affects replica sets.

.. note:: 
  This is different behavior than |MongoDB|, where the oplog is a capped collection. You should change this default to the appropriate number of days or hours that you want to retain in the oplog.

.. variable:: fastUpdates

  :default: false

Supported since 2.0.0

If this setting is true, then :ref:`fast_updates` are enabled.

.. variable:: fastUpdatesIgnoreErrors

  :default: false

Supported since 2.0.0

If a :ref:`fast_updates` results in an error (e.g. incrementing a text field), the error may not be encountered until some undetermined time in the future. If this setting is true, such errors are ignored. If false, such errors are occasionally written to the error log.

.. variable:: fsRedzone

  :default: 5%

This variable controls the percentage of the file system that must be available for inserts to be allowed.

.. variable:: loadPlugin

Specified as ``loadPlugin=name:checksum``.

Specifies that the plugin name from :file:`libname.so` should be loaded as a plugin on startup, and that it will not be loaded if it does not have the proper checksum.

You can determine this checksum by running the ``loadPlugin`` command manually, as ``db.adminCommand({loadPlugin: "name"})``. The checksum will be returned in the result object.

This option can be specified multiple times, in a config file or on the command line. The plugins available are :ref:`hot_backup` and :ref:`pitr_plugin`.

.. variable:: loaderCompressTmp

  :default: true

Whether the :ref:`bulk_loader` will compress intermediate files (see :variable:`tmpDir`) while building indexes.

.. variable:: loaderMaxMemory

  :default: 100MB

Controls the amount of memory used by the Bulk Loader.

Can supply values using K/M/G (i.e., '400M' = 400 Megabytes)

.. variable:: lockTimeout

  :default: 4000 ms

This variable controls the amount of time in milliseconds that a transaction will wait for a lock held by another transaction to be released. If the conflicting transaction does not release the lock within the lock timeout, the transaction that was waiting for the lock will get a lock timeout error.

A value of ``0`` disables lock waits.

.. note:: 
 See :ref:`document-level_lock_conflicts` for details about conflicting transactions.

.. variable:: locktreeMaxMemory

  :default: 10% of :variable:`cacheSize`

Maximum amount of memory (in bytes) used by the document-level locking data structures.

.. variable:: logDir

  :default: same directory as ``dbpath``

Directory where the |TokuMX| transaction log (similar to the |MongoDB| durability journal) will go.

.. variable:: logFlushPeriod / journalCommitInterval

  :default: 100ms
  :values: 0ms - 300ms

How often to fsync the recovery log (like ``journalCommitInterval`` in basic |MongoDB|).

Unlike original |MongoDB|, a value of ``0`` means commit every operation.

.. warning::

  Committing every operation can be expensive.

.. variable:: pluginsDir

  :default: ``installdir/lib64/plugins``

The directory searched for plugins by :variable:`loadPlugin`.

Usually should not be modified.

.. variable:: rsMaintenance

  :default: false

Supported since 2.0.0

Starts the server in `maintenance mode <http://docs.mongodb.org/manual/reference/command/replSetMaintenance/>`_

Has no effect if the server is not a replica set member.

Used for :ref:`pitr_plugin`.

.. variable:: tmpDir

  :default: same directory as ``dbpath``

Directory where |TokuMX| will place temporary files used by the :ref:`bulk_loader` for building indexes.

.. variable:: txnMemLimit

  :default: ``1048576`` bytes (1MB)
  :values: up to 2097152 (2MB)

The amount, in bytes, of replication data a transaction stores in memory before spilling the data to disk.

This parameter only affects replica sets.

We recommend leaving the default unless there is some compelling reason to change this.

Advanced Parameters
===================

Several server parameters can be set with ``--setParameter`` on the command line.

.. variable:: compressBuffersBeforeEviction

  :default: true

Supported since 1.5.0

During partial eviction, we have the option to compress nonleaf node buffers rather than completely evicting them the first time we run partial eviction. This costs some CPU and doesn't reclaim as much space, but may avoid disk I/O later if the buffer is reused soon.

.. variable:: defaultCompression

Supported since 1.4.2

Sets the default :option:`compression` for new collections and indexes.

.. variable:: defaultFanout

Supported since 1.4.2

Sets the default :option:`fanout` for new collections and indexes.

.. variable:: defaultPageSize

Supported since 1.4.2

Sets the default :option:`pageSize` for new collections and indexes.

.. variable:: defaultReadPageSize

Supported since 1.4.2

Sets the default :option:`readPageSize` for new collections and indexes.

.. variable:: electionBackoffMillis

  :default: 1000 ms

Supported since 2.0.0

During a replication election, multiple members may be electable. To reduce the chance of multiple elections running concurrently, basic |MongoDB| secondaries sleep for a random amount of time up to 1000 ms.

|TokuMX| makes this duration configurable. Generally, this value should not be modified unless experiments show that concurrent elections often conflict and cause downtime. This can happen in replica sets with high-latency connections between members, especially in cross-datacenter scenarios. In that case, increasing :variable:`electionBackoffMillis` may reduce downtime due to concurrent elections.

.. variable:: forceWriteLocks

 :default:false

Supported since 1.5.0

If this setting is true, then when a client is waiting to acquire one of the per-database :ref:`metadata_locks`, all other clients that hold the read lock will abort and need to be retried. This can avoid stalls in some workloads, but requires that the application understand it may need to retry some operations.

This setting controls the default value for :command:`setWriteLockYielding` for each newly-created connection.

.. variable:: loaderCompressTmp

  :default: true

Supported since 1.4.0

Controls whether the :ref:`bulk_loader` compresses temporary files put in :variable:`tmpDir` with ``quicklz``.

.. variable:: migrateStartCloneLockTimeout

  :default: 60000 ms

Supported since 1.5.0

Controls how long a sharding chunk migration will wait to acquire a range lock on the chunk it intends to move.

.. variable:: migrateUniqueChecks

  :default:true

Supported since 1.4.0

Controls whether sharding chunk migrations do unique checks when inserting new data on the recipient shard.

This setting is ``true`` by default, which is the safest option, but if you know that any of the following are true, you can turn this off for a performance boost during chunk migrations:

* You are sharding with a :option:`primaryKey`.

* You are allowing |MongoDB| to assign ``_ids`` for your application.

* You know your ``_ids`` are all globally unique and are willing to accept the consequences if they aren't.

For a full discussion of this setting's meaning, see `What's new in TokuMX 1.4, Part 5: Faster chunk migrations <http://www.tokutek.com/2014/02/whats-new-in-tokumx-1-4-part-5-faster-chunk-migrations/>`_.

.. variable:: soTimeoutForReplLargeTxn

 :default: 600s

Supported since 1.5.1

Specifies the socket timeout of replication thread's connection while replicating a large transaction stored in ``local.oplog.refs``.

.. variable:: numCachetableBucketMutexes

  :default: 1000000

Supported since 1.3.2

Internal use only.

.. variable:: pkUniqueChecks

  :default: true

Supported since 1.5.0

Internal use only.

Getting and Setting Dynamic Parameters
======================================

Many server parameters can be set and viewed dynamically, using :ref:`parameter_commands`.

The new dynamic parameters in |TokuMX| are:

 * :command:`checkpointPeriod`
 * :command:`cleanerIterations`
 * :command:`cleanerPeriod`
 * :command:`compressBuffersBeforeEviction`
 * :command:`defaultCompression`
 * :command:`defaultFanout`
 * :command:`defaultPageSize`
 * :command:`defaultReadPageSize`
 * :command:`electionBackoffMillis`
 * :command:`fastUpdates`
 * :command:`fastUpdatesIgnoreErrors`
 * :command:`forceWriteLocks`
 * :command:`loaderCompressTmp`
 * :command:`loaderMaxMemory`
 * :command:`lockTimeout`
 * :command:`logFlushPeriodjournalCommitInterval`
 * :command:`migrateStartCloneLockTimeout`
 * :command:`migrateUniqueChecks`
 * :command:`soTimeoutForReplLargeTxn`
 * :command:`numCachetableBucketMutexes`
 * :command:`pkUniqueChecks`

Deprecated Parameters
=====================
Some parameters available in basic |MongoDB| have been deprecated in |TokuMX|.

* ``directoryperdb``

All data is in one directory.

* ``dur/journal``

Crash safety is always on.

* ``durOptions/journalOptions``

Currently, there are no options with respect to crash safety.

* ``nodur/nojournal``

Crash safety cannot be disabled.

* ``smallfiles``

|TokuMX| does not preallocate data files nearly as aggressively as |MongoDB|.

* ``noprealloc/nopreallocj``

We do not aggressively preallocate indexes or logs.

* ``repair``

We automatically recover indexes to a consistent state.

* ``upgrade``

We automatically upgrade indexes to the current format.

* ``nssize``

Our implementation of the namespace index is not fixed-size.

* ``oplogSize``

The oplog is no longer a capped collection.

* ``master``

Classic master/slave replication from |MongoDB| 1.x is not supported.

* ``slave``

Classic master/slave replication from |MongoDB| 1.x is not supported.

* ``pretouch``

Classic master/slave replication from |MongoDB| 1.x is not supported.

* ``replIndexPrefetch``

Replication does not require prefetching, and therefore no longer has prefetch threads.

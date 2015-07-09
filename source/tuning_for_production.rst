.. _tuning_for_production:

=======================
 Tuning for Production
=======================

In most cases the default options should be left in-place to run |TokuMX|, however it is a good idea to review some of the configuration parameters.

In this section, we'll describe some concerns for tuning |TokuMX| for production workloads, in terms of resource usage and choices to make for optimal efficiency.

Resource Usage
==============

|TokuMX| is unlike most databases in that, even when data is much larger than RAM, it can still be mostly CPU-bound on some workloads that would make other databases bound by disk seeks on rotating media, or disk bandwidth on SSDs. In general, TokuMX's resource usage is just different from that of other databases. This section serves as a starting point for understanding TokuMX's resource usage, for the four most commonly constrained resources: CPU, RAM, disk, and network.

CPU
---

CPU usage is attributed to:

Compression work
^^^^^^^^^^^^^^^^
When data blocks are written to disk (for checkpoint or when evicted while dirty, in the presence of memory pressure), they are first compressed, and compression is primarily CPU work. Decompression is generally much cheaper than compression, and is hardly noticeable next to the disk I/O done just before it, even on most SSDs.

Message application
^^^^^^^^^^^^^^^^^^^
High-volume update workloads tend to stress this subsystem, which updates the data in leaves with the result of applying deferred operations above them in the tree. Small updates to large documents can disproportionately stress this system, in that case it can help to break up a large document into smaller documents and rely on TokuMX's multi-document transactional semantics to read the same data consistently later.

Miscellaneous tasks
^^^^^^^^^^^^^^^^^^^
Tree searches, serialization and deserialization, sorting for aggregation, and other things common to most databases.

Building indexes
^^^^^^^^^^^^^^^^
Compressing intermediate files during bulk load of indexes (foreground ``ensureIndex`` operations as well as loading collections with :program:`mongorestore`).

RAM
---

RAM usage is configurable with the :variable:`cacheSize` parameter, and is attributed to:

Data cache
^^^^^^^^^^
Caching uncompressed user data in tree node blocks is the main use of RAM.

|TokuMX| doesn't distinguish here between data and indexes, data is just stored in a clustering primary key index.

Document-level locks
^^^^^^^^^^^^^^^^^^^^
:ref:`document-level_locks` are stored in a locktree. The locktree's size is dependent on the number of concurrent writers and the keys modified by those writers. Its maximum size is controlled by :variable:`locktreeMaxMemory`.

Building indexes
^^^^^^^^^^^^^^^^
Each running bulk load reserves an additional 100MB of RAM by default (:variable:`loaderMaxMemory`).

Miscellaneous data
^^^^^^^^^^^^^^^^^^
Transient data for cursors, transactions, intermediate aggregation results, thread stacks, etc. This is typically negligible except on extremely memory-constrained systems.

Disk
----
Disk usage is attributed to:

Queries
^^^^^^^
Reading data not in the cache to answer queries or to find existing documents for updates.

This can be sequential or random, depending on the queries, but the basic unit of I/O is 64KB before compression (:option:`readPageSize` in :ref:`collection_and_index_options`). In most workloads, especially read-heavy workloads, this is the primary source of disk utilization.  

Checkpoints and evictions
^^^^^^^^^^^^^^^^^^^^^^^^^
Writing dirty blocks for checkpoint, or when the cache is too full and something dirty needs to be evicted to make room for other data.

These are large writes—4MB before compression (:option:`pageSize` in :ref:`collection_and_index_options`)—that tend to appear as sequential I/O.

Logging
^^^^^^^
Writing and fsyncing the transaction log (similar to the journal in basic |MongoDB|), for any write operation.

These are frequent, small, sequential writes eligible for merging, and frequent fsyncs eligible for group commit, and usually show up as sequential I/O. The fsyncs can be easily absorbed by a battery-backed disk controller, since the I/O is sequential, and the log can be placed on a different device with the :variable:`logDir` server parameter.

Building indexes
^^^^^^^^^^^^^^^^
Writing and reading intermediate files to and from disk during bulk load. This I/O is all sequential, and can be placed on a different device with the :variable:`tmpDir` server parameter.

Network
-------
Network usage is almost identical to basic |MongoDB|, and is attributed to:

Replication
^^^^^^^^^^^^
Replicating the oplog to secondaries in a replica set.

:ref:`Some oplog entries are larger <miscellaneous_differences>` than in basic |MongoDB|, to support faster application on secondaries; these will cost more bandwidth.

Sharding
^^^^^^^^
Chunk migrations to other shards in a sharded cluster.

Clients
^^^^^^^
Sending and receiving data to and from applications and sharding routers.

Memory Allocation
=================
|TokuMX| will allocate 50% of the installed RAM for its own cache (:variable:`cacheSize`).

While this is optimal in most situations, there are cases where it may lead to memory over allocation. If the system tries to allocate more memory than is available, the machine will begin swapping and run much slower than normal.

The 2 most frequent cases when it is necessary to set the :variable:`cacheSize` to a value other than the default are:

* Using Direct I/O:

  The directio parameter enables Direct I/O for all data and index filesystem operations, which bypasses the kernel’s page cache. This parameter removes the need to leave extra space available for the page cache, so we suggest increasing the cacheSize parameter to 80% or more of main memory. Using the directio flag and increasing cacheSize often improves the performance of TokuMX.

* Running other memory heavy processes on the same server as TokuMX:

  In many cases the database process needs to share the system with other server processes like additional database instances, http server, application server, email server, monitoring systems and others. In order to properly configure TokuMX's memory consumption, it's important to understand how much free memory will be left and assign a sensible value. There is no fixed rule, but a conservative choice would be 50% of available RAM while all the other processes are running. If the result is under 2GB, you should consider moving some of the other processes to a different system or using a dedicated database server. As above, if you are using Direct I/O, this could instead be 80% or more of available RAM, while other processes are running.

.. note::
  :variable:`cacheSize` needs to be set before starting the server and cannot be changed while the server is running.

.. _bulk_loader:

Bulk Loader
===========

|TokuMX| includes a bulk loader that increases the throughput of initial loads into empty or non-existent collections.

The bulk loader is used automatically in the following scenarios:

* All foreground index builds.

* :program:`mongorestore` and :program:`mongoimport` operations that create collections (rather than insert into existing collections).

  For these tools, either the target collection must not exist beforehand, or the ``--drop`` option must be passed to the tool.

  In addition, for :program:`mongorestore`, the bulk loader is disabled if the ``--w`` option is specified greater than ``1``.

.. note:: 
  When loading large data sets, try to always load on a standalone server and then convert that server to a replica set after the load is finished (using a cold or hot backup to seed secondaries).
  This technique will avoid filling the oplog with the data from the load and eliminate the initial replication lag caused by these large loading activities.

.. Percona TokuMX documentation master file, created by
   sphinx-quickstart on Thu Jul  2 11:24:46 2015.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

.. _dochome:

==================================
 Percona TokuMX - Documentation
==================================

|TokuMX| is a highly scalable, zero-maintenance downtime database supporting the |MongoDB| v2.4 protocol and drivers that replaces all |MongoDB| storage with |Fractal Tree| indexes. |TokuMX| requires no changes to |MongoDB| applications or code. The main benefits of |TokuMX| are:

 * Improved single-threaded and multi-threaded performance
 * Compression
 * Fully ACID and MVCC transaction support
 * No maintenance or scheduled downtime necessary
 * Clustering key support for query acceleration
 * Better read scaling and reduced lag on replica sets
 * Low-impact migrations and accelerated range queries with clustering shard keys
 * Fast, read-free updates
 * Hot Backup
 * Point-in-time Recovery
 * Audit Logging
 * Reduced SSD wear
 * Geospatial Indexes
 * Includes all |MongoDB| v2.4 functionality except Text Search

Key Features
============

Performance
-----------
TokuMX enhances large databases (typically 50 GBs or larger) in the following ways:

 * Faster indexing: TokuMX speeds up indexing by 10x or more, so database scalability radically improves. With its significant improvement to insertion speed and indexing, TokuMX delivers faster query performance in live production systems when collections are too large for main memory. This makes TokuMX ideal for applications that must simultaneously query and update large volumes of rapidly arriving data (e.g., clickstream analytics).

 * Document Level Locking: TokuMX has document level locking for inserts, updates, and deletes, whereas MongoDB has DB level locking. This improves the performance of multithreaded read/write workloads even within a single collection, allowing a single server to scale out reads and writes.

 * Fast updates that can run 10-30x faster by avoiding unnecessary disk reads.

Compression
-----------
TokuMX transparently compresses all data on disk, and achieves up to a 20x reduction in HDD and flash storage requirements without impacting performance. It can dramatically reduce the number of servers needed to host a MongoDB database.

:ref:`transactions`
-------------------
|TokuMX| adds transactions with MVCC semantics and ACID reliability to any MongoDB application, making MongoDB suitable for a much wider range of applications. This includes:

 * Snapshot (MVCC) queries: Without application changes, queries in TokuMX return a consistent snapshot of the data. Queries will not see the effects of modifications that complete after the query starts, unlike in MongoDB where documents can be out of sync, duplicated, or missing due to interleaved updates.

 * Multi-statement transactions: TokuMX supports new commands for managing concurrent, isolated transactions that span multiple query and modification statements.

Note that multi-statement transactions are not available in sharded environments.

Zero Maintenance Downtime
-------------------------
TokuMX does not require routine maintenance to deal with fragmentation, as performance remains steady through heavy usage. Data does not ever need to be compacted, repaired, or re-indexed to restore performance.

Clustering Indexes
------------------
TokuMX provides an option to declare a secondary index as clustering to provide much better performance on a broader range of queries. A clustering key includes (or clusters) all document fields in a collection along with the key. As a result, one can efficiently retrieve any field when doing a range query on a clustering index, thereby dramatically reducing I/O.

Replication
-----------
In replica sets, TokuMX significantly reduces the I/O requirements for secondaries to apply changes from the primary. This leads to reduced lag and better read scaling on secondaries.

Note that TokuMX replication is not compatible with MongoDB, so you cannot run both in a replica set. However, we have also created a “Migration Toolkit” to assist evaluations and deployments for users with existing installations of basic MongoDB.

Sharding
--------
In sharded collections, range queries in TokuMX are optimized thanks to the use of a clustering index for the shard key, and migrations between TokuMX servers impose very low I/O overhead, making low-entropy keys good candidates for sharding.

In addition, chunk migrations (for cluster balancing) in TokuMX require significantly less I/O than in basic MongoDB. This means migrations typically have no impact on the application workload in TokuMX, which simplifies operations and data modeling.

:ref:`hot_backup`
-----------------
TokuMX's Hot Backup solution gives you backups you can trust, without downtime or complicated cluster management. This feature is only available in the Enterprise Edition.

:ref:`pitr_plugin`
----------------------
TokuMX's Point-in-time Recovery plugin empowers administrators to get a snapshot of the database at any point in the past, for disaster recovery, accountability, or historical analytics. This feature is only available in the Enterprise Edition.

:ref:`audit` Logging
--------------------
TokuMX's Audit Logging provides insight into the security of your cluster, providing feature parity with audit capabilities of MongoDB Enterprise Edition v2.6, with even more reliability. This feature is only available in the Enterprise Edition.

Reduced SSD Wear
----------------
TokuMX's fractal trees do fewer and bigger writes to the file system, thereby reducing wear on the SSD and extending its lifetime.

Geospatial Indexes
------------------
Starting with version 2.0, TokuMX includes full support for Geospatial Indexes.

MongoDB v2.4 Compatibility
--------------------------
TokuMX is a true drop-in replacement for MongoDB. All MongoDB v2.4 features are present except Text Search.

Full Table of Contents
=======================

.. toctree::
   :numbered:
   
   installation
   installation_from_packages
   server_parameters
   collection_index_options
   commands
   server_status
   concurrency
   transactions
   replication
   sharding
   tuning_for_production
   tokumx_migration
   partitioned_collections
   fast_updates
   hot_backup
   pitr_plugin
   audit
   errata
   faq
   glossary
   copyright
   trademark-policy

* :ref:`genindex`


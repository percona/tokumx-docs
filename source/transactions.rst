.. _transactions:

============
Transactions
============

While basic |MongoDB| supports transactional semantics only for a single document at a time, |TokuMX| provides a spectrum of stronger transactional guarantees, ranging from MVCC snapshot queries all the way through multi-statement serializable transactions.

.. warning:: 
 As you may have heard, in a distributed system like a |MongoDB| or |TokuMX| sharded cluster, transactions can be tricky. |TokuMX| provides some transactional guarantees to sharded clusters beyond basic |MongoDB|, but not everything available on a single |TokuMX| shard extends to a full cluster.
 
 As you read this document, know that everything here applies only to single servers and replica sets, not to sharded clusters. In :ref:`semantics_in_sharding` we discuss how |TokuMX| sharded clusters behave in terms of transactional semantics, and for that discussion it will be useful to understand the situation for a single shard.

.. _multi-document_atomicity:

Multi-document Atomicity
========================

In |MongoDB| and |TokuMX|, there are three typical write operations that affect multiple documents at once:

 * Batched inserts.
 * Updates with ``{multi: true}``.
 * Deletes (``remove``) with ``{justOne: false}``.

In basic |MongoDB|, if such an operation fails part of the way through for any reason, the operation is left partially completed. For example, if in an update affecting of ten documents, the sixth update fails due to a unique index constraint violation, then the operation will fail at that point, leaving the first five documents reflecting the effect of the update, and the documents after and including the sixth in their pre-update state.

In |TokuMX|, all operations are transparently wrapped in an `atomic transaction <https://en.wikipedia.org/wiki/Atomicity_(database_systems)>`_ on the server. The operation either completely succeeds or completely fails; there is no longer a "partial failure" result.

.. note:: 
 The only exception is a write to the profile collection (aborted transactions are still profiled).

In |TokuMX|, for the above example of a ten-document update, if any document in the middle fails due to a unique index constraint violation (or for any other reason), the entire transaction will be rolled back, and all documents will be restored to their pre-update state.

In addition, any query will either see all documents in their pre-update state, or will see all documents in their post-update state (read more below in :ref:`snapshot_isolation`).

Example:
Consider a collection with a unique index on ``{a: 1}`` containing these documents:

.. code-block:: javascript

  { "a" : 10 }
  { "a" : 20 }

Suppose we run the following update statement:

.. code-block:: javascript

  db.foo.update({}, {$set: {a: 30}}, {multi: true})

Under basic |MongoDB|, this will first update ``{a: 10}`` to be ``30``, then it will try to update ``{a: 20}`` but this will fail the uniqueness constraint, and we'll be left with:

.. code-block:: javascript

 { "a" : 20 }
 { "a" : 30 }
 
Under |TokuMX|, the first document will be updated, the second one will fail the uniqueness constraint, and then the transaction will abort, returning the first document to 10. This way, we'll be left in the original state and we can repair the update statement and try again:

.. code-block:: javascript

  { "a" : 10 }
  { "a" : 20 }

This behavior is automatically present in every write operation in |TokuMX|. Some users of basic |MongoDB| may do things like always use ``upsert`` instead of inserts to make those inserts idempotent so they can be retried—with |TokuMX|, those users can use normal batched inserts, and just retry the whole batch if it fails.

.. _snapshot_isolation:

Snapshot Isolation
==================

|MongoDB| and |TokuMX| support many flavors of read operations, including normal queries, aggregation, and mapreduce. In many cases, these operations read and return data from multiple documents.

In basic |MongoDB|, reads, like writes, are atomic on a per-document basis. So a query will never see a single document only partially updated, but if a query needs to look at multiple documents and some of them are affected by an update, it might see some of them before the update and some of them after the update.

This can sometimes cause strange behavior, like resources temporarily going "missing" or becoming "duplicated" because a query saw documents at two different logical points in time, with respect to an update.

In |TokuMX|, all reads (queries, aggregations, mapreduce, even reads done to perform updates) use `snapshot isolation <https://en.wikipedia.org/wiki/Snapshot_isolation>`_ leveraging `multi-version concurrency control <https://en.wikipedia.org/wiki/Multiversion_concurrency_control>`_ information.

When a query starts, it defines a point in time at which it will see the state of all documents it examines. Any writes that commit after this point in time—including those that create new documents—won't affect the results of the query. They'll effectively be invisible to the query.

Example:
Consider a collection containing these documents:

.. code-block:: javascript

  { "a" : 0 }
  { "a" : 1 }
  { "a" : 2 }
  { "a" : 3 }

Suppose we have two clients running concurrently. Client A runs a single ``find()`` on the collection, while Client B runs a ``remove`` and an ``insert``:

============== ================ ===================================================================
Client A       Client B         Notes
============== ================ ===================================================================
                                Client A's find() begins
Reads {a: 0}   Removes {a: 2}  
Reads {a: 1}   Inserts {a: 100}    
Reads {a: 2}                    This read occurs in TokuMX, but not in basic MongoDB.
Reads {a: 3}        
Reads {a: 100}                  This read does not occur in TokuMX, it does occur in basic MongoDB.
                                Client A's find() completes
============== ================ ===================================================================

Note that all of Client B's operations that happened after Client A's snapshot started were not reflected in the results Client A received from the query.
This behavior is automatically present in every read operation on a |TokuMX| server. This means that regardless of how long a query takes to execute, how many round trips it takes, or how many documents it touches, it will always see a consistent view of the data.

.. _multi-statement_transactions:

Multi-statement Transactions
============================

.. warning:: 
 Multi-statement transactions are currently not supported in sharded clusters.

In earlier sections, we have discussed some transactional semantics of |TokuMX| that apply transparently to all applications written for basic |MongoDB|: :ref:`multi-document_atomicity` and :ref:`snapshot_isolation`.

|TokuMX| also exposes explicit, multi-statement transactions to the application. These transactions provide semantics similar to single-statement transactions, but across multiple API calls, enabling more expressive transactions to be implemented in the application.

.. _isolation:

Isolation
---------

Multi-statement transactions in |TokuMX| support three different `isolation levels <https://en.wikipedia.org/wiki/Isolation_(database_systems)>`_: MVCC (the default), Serializable, and Read Uncommitted. The isolation level must be chosen at the beginning of the transaction, in :command:`beginTransaction`.

.. _mvcc:

MVCC
^^^^

The default isolation level, MVCC, is what is used by normal single-statement reads. These transactions give a guarantee equivalent to ``REPEATABLE READ`` in SQL databases.

Reads done in the context of an MVCC transaction use a snapshot as described in :ref:`snapshot_isolation` and take no :ref:`document-level_locks`.

.. note:: 
 Unlike some database systems that implement REPEATABLE READ isolation, ``MVCC`` transactions in |TokuMX| are immune to the "Phantom Read" phenomenon as described in SQL 92.

Writes done in the context of an transaction still take :ref:`document-level_locks` as in :ref:`serializable`. However, if the writes they perform depend on reads done previously, those transactions can cause lost updates; in most cases, transactions that perform writes should use :ref:`serializable` transactions.

.. _serializable:

Serializable
^^^^^^^^^^^^

The serializable isolation level is what is used by normal single-statement writes. These transactions give a guarantee equivalent to ``SERIALIZABLE`` in SQL databases.

All operations done in the context of a serializable transaction take :ref:`document-level_locks` on the documents they read or modify.

.. warning:: 
 Long-running serializable transactions that conflict on :ref:`document-level_locks` can cause deadlocks and lock wait timeout errors, which should be retried by the application. This can happen even if neither transaction is doing any writes. See :ref:`document-level_lock_conflicts` for more details.

Since reads done in the context of a serializable transaction take :ref:`document-level_locks`, serializable transactions that do writes are immune to the "Lost Update" problem. For this guarantee, in most cases, transactions that perform writes should be serializable.

.. _read_uncommitted:

Read Uncommitted
^^^^^^^^^^^^^^^^

The read uncommitted isolation level is typically unsuitable for most applications, but is provided for internal use (trivia: it is used to implement tailable cursors on the oplog). These transactions are equivalent to ``READ UNCOMMITTED`` in SQL databases.

Read uncommitted transactions are similar to :ref:`mvcc` transactions, except that instead of using a snapshot, reads can return values written later than when the transaction began, even including values yet uncommitted, that may abort in the future.

Writes done in the context of a read uncommitted transaction still take :ref:`document-level_locks`.

.. caution::
  Read uncommitted transactions are susceptible to a large number of bizarre read phenomena, similar to reads in basic |MongoDB|. Nearly all applications should never use read uncommitted transactions.
  Read uncommitted transactions should not be considered to be an optimization over MVCC transactions, the work done in either isolation level is nearly the same.

.. _drivers:

Drivers
-------

Since multi-statement transactions are exposed to the application, an important part of understanding how to use them is understanding the programming model.

Multi-statement transactions in |TokuMX| are created and rolled back or committed with :ref:`transaction_commands`, and during their lifetime they are associated with a single connection from the application to the database server.

Therefore, to use transactions, you need to be able to control the lifetime and thread affinity of your application's connections. Most standard |MongoDB| drivers support this type of feature, typically named something similar to `MongoClient.start_request <http://api.mongodb.org/python/current/api/pymongo/mongo_client.html#pymongo.mongo_client.MongoClient.start_request>`_ (in *Python*) or `MongoServer.RequestStart <http://api.mongodb.org/csharp/1.0/html/d751ceee-e249-f039-120e-fc9c1d440d60.htm>`_ (in *C#*).

When a thread in your application wants to execute a multi-statement transaction, it must reserve a connection for its exclusive use for the duration of the transaction, and it must use only that connection to perform operations comprising that transaction.

.. caution:: 
  Failure to properly manage connection-thread affinity while using multi-statement transactions can result in two possible failure modes:
    * If another thread uses your connection while you are executing a transaction, their operations will be included in your transaction, including the possible accidental commit or rollback of your transaction, or the accidental failure to begin a transaction while your transaction is live on your connection.

    * If you use another thread's connection while you are intending to execute a transaction, those operations performed on the other connection will not be part of your transaction and may be independently committed or aborted separate from your transaction.

.. important:: 

  Some language drivers, for example `node-mongodb-native <http://mongodb.github.io/node-mongodb-native/>`_, do not have a way to express this type of relationship between a connection and a thread (or in the case of node.js, an execution context). These drivers currently make it impossible to use TokuMX's multi-statement transactions in a meaningful way unless each process can be considered a single thread of execution, though there are some community efforts to add this capability.

.. _trans_commands:

Commands
--------

Multi-statement transactions are managed by running three commands over a connection: :command:`beginTransaction`, :ref:`commitTransaction`, and :ref:`rollbackTransaction`. See :ref:`transaction_commands` for more details.

.. _semantics_in_sharding:

Semantics in Sharding
=====================

|TokuMX| sharded clusters cannot provide some of the transactional semantics described above without making a big trade-off in performance. However, |TokuMX| sharded clusters can still provide some guarantees to the application that are stronger than what basic |MongoDB| provides. In this section, we'll describe how each of the above concepts works with sharding.

.. _atomicity_in_sharding:

Atomicity in Sharding
---------------------

In a sharded cluster, |TokuMX| provides atomicity on a per-shard basis. :ref:`multi-document_writes` that target a single shard are done atomically just as in a replica set. This is always the case for unsharded collections.

A batch of inserts to a sharded collection targets a single shard (and is therefore atomic) if all documents in the batch have the same shard key. Updates with ``{multi: true}`` and removes with ``{justOne: false}`` target a single shard (and are therefore atomic) if the query parameter contains an equality clause on the shard key.

Example:
Consider a sharded collection with the shard key {sk: 1}. The following operations always target a single shard:

.. code-block:: javascript

  db.foo.insert([{x: 1, sk: 42},
                {x: 2, sk: 42},
                {x: 3: sk: 42}])
  db.foo.update({sk: 42}, {$set: {isAnswer: true}}, {multi: true})
  db.foo.remove({sk: 42, x: {$gt: 10}}, false)

By contrast, the following operations do not necessarily target a single shard, and therefore may not be atomic:

.. code-block:: javascript
                              
  db.foo.insert([{x: 1, sk: 42},
                {x: 2, sk: 43},
                {x: 3: sk: 44}])
  db.foo.update({sk: {$gte: 43}}, {$set: {isAnswer: false}}, {multi: true})
  db.foo.remove({sk: {$lt: 42}, x: 10}, false)

In addition, `tag-aware sharding <http://docs.mongodb.org/manual/core/tag-aware-sharding/>`_ can be used to make more operations atomic. If you assign a chunk range to a tag which only has one shard, then all operations that affect only that range will also be atomic.

.. warning::
  There is an issue in :program:`mongos` where chunks can span multiple tag ranges. If this happens, it can mean that some documents may not reside on the shard they should according to tag assignments. This could destroy atomicity of operations on the affected tag ranges. To prevent this problem, you should `create splits <http://docs.mongodb.org/manual/tutorial/split-chunks-in-sharded-cluster/>`_ on your tag boundaries.

.. _query_behavior_in_sharding:

Query Behavior in Sharding
--------------------------

Like writes, reads in |TokuMX| sharded clusters have their normal transactional semantics on a single-shard basis. A query that only targets a single shard (see "`Query Isolation <http://docs.mongodb.org/manual/core/sharding-shard-key/#query-isolation>`_") gets the benefit of all of the :ref:`snapshot_isolation` benefits of a single server or replica set.

Example:
Consider a sharded collection with the shard key {sk: 1}. The following operations always target a single shard:

.. code-block:: javascript

  db.foo.find({sk: 42})
  db.foo.find({sk: 42, a: {$gt: 100, $lte: 200}})
  db.foo.aggregate([{$match: {sk: 42}},
        {$group: {_id: null, count: {$sum: 1}}}])

By contrast, the following operations do not necessarily target a single shard, and therefore may not be totally consistent snapshots:

.. code-block:: javascript

  db.foo.find({sk: {$gt: 42}})
  db.foo.find({a: {$gt: 100, $lte: 200}})
  db.foo.aggregate([{$match: {"author.last": "Leiserson"}},
        {$group: {_id: null, count: {$sum: 1}}}])

Queries that span multiple shards use one MVCC snapshot on each shard, but in the presence of writes that span multiple shards, the reads and writes may be done in an order that is not linearizable.

.. note::
  If all writes in your application happen to target a single shard, then queries that span multiple shards can exhibit some linearizability. Multiple such queries may not linearize in a coherent way, but each query individually, taken along with the universe of writes done to the cluster, will be linearizable, though this property is of limited value and is hard to reason about.

.. _concurrency:

============
Concurrency
============

Basic |MongoDB| supports reader concurrency but does not support writer concurrency or mixed reader/writer concurrency. |TokuMX| supports concurrency between multiple writers even within the same collection, as well as concurrency between mixed read/write workloads. This is accomplished with the use of :ref:`document-level_locks` in the Fractal Tree indexes.

In this section, we'll describe the concurrency model that |TokuMX| supports, including :ref:`metadata_locks` and :ref:`document-level_locks`.

.. _metadata_locks:

Metadata Locks
==============

Background
----------

Basic |MongoDB| has a per-database readers/writer lock that controls access to all the collections (data and indexes) in each database.

This lock is a readers/writer lock and is taken for writing (which excludes other readers and writers) for all operations that write to disk, be they collection or index creation or deletion, or just simple collection inserts, updates, and deletes.

|TokuMX| does not use the readers/writer locks for DML operations, i.e. normal collection reads and writes (``find``, ``aggregate``, ``insert``, ``update``, ``remove``, ``findAndModify``, etc.). Instead, |TokuMX| uses :ref:`document-level_locks` to control these operations.

.. _ddl_operations:

DDL Operations
--------------

|TokuMX| makes use of the per-database readers/writer lock from basic |MongoDB| to manage collection metadata. Collection metadata includes:

  * The existence of a collection or database.

  * The existence of indexes on a collection (including background indexes that are in progress).

  * :ref:`collection_and_index_options` that can be :ref:`modified dynamically <modifying_index_options>`.

  * Whether a given index is `multiKey <http://docs.mongodb.org/manual/core/index-multikey/>`_.

In order to change any of the above properties (e.g. to create or drop a collection or an index), the readers/writer lock for the appropriate database is taken for writing (exclusive lock). To protect normal DML operations against the drop of a collection or index, all DML operations hold the read lock for the appropriate database (shared lock).

Foreground index builds hold the write lock during the entire index build phase, which can take a long time and block other clients from accessing the database. Background index builds use the write lock briefly at the beginning and end, but most of the time spent building the index does not hold the database-level lock, allowing other operations to proceed.

.. warning:: 
  DML operations other than queries do not yield the database-level lock like they do in basic |MongoDB| (this is to provide :ref:`multi-document_atomicity`). Therefore, since the readers/writer lock is "`writer greedy <http://docs.mongodb.org/manual/faq/concurrency/#what-type-of-locking-does-mongodb-use>`_", if there are long-running DML operations (e.g. a large ``{multi: true}`` update), a DDL operation may block behind them, and all other DML operations will begin to block, waiting for the DDL operation to get its write lock.
  If your application includes many long-running DML operations, you may see stalls occur whenever there is a waiting DDL operation. To prevent this, you can use the :variable:`forceWriteLocks` parameter or :command:`setWriteLockYielding` command, which will cause DML operations to abort early if a DDL operation appears that wants the lock. If you use this feature, your application should be prepared to retry DML operations that abort in this way. This strategy will maintain :ref:`multi-document_atomicity`.

.. _visualizing_metadata_locks:

Visualizing Metadata Locks
--------------------------

The ``db.currentOp()`` command is a powerful tool for debugging stalls. By default, it returns a large amount of information—one object for every running client operation—which can be then filtered with additional client-side javascript.

It can also accept a BSON query object, similar to the query portion of a ``find()`` operation, to show just the operations whose information object matches the given query. The fields ``waitingForLock`` and ``locks`` can be used to inspect the state of metadata locks.

Find all operations currently waiting for a DBRead or GlobalRead lock:

.. code-block:: javascript

  db.currentOp({waitingForLock: true, 'locks.^': {$or: ['r', 'R']}})

Find the operation that currently has a GlobalWrite lock:

.. code-block:: javascript

  db.currentOp({waitingForLock: false, 'locks.^': 'W'})

Some system threads have this type of information as well, but are not shown by default. To see these system threads, run ``db.currentOp({$all: true})``. However, the ``$all`` flag cannot be used with a query object.

|TokuMX| adds the ``$allMatching`` flag (since 1.3.3) that accepts a query object, and also shows system threads like the ``$all`` flag.

Find all operations that currently have or want GlobalWrite lock, including background system threads:

.. code-block:: javascript

  db.currentOp({$allMatching: true, 'locks.^': 'W'})

.. _document-level_locks:

Document-level Locks
====================

|TokuMX| achieves write concurrency by employing document-level locking. Read concurrency is achieved without any document-level locking; reads provide consistency using MVCC information as discussed in :ref:`snapshot_isolation`.

Document-level locking means that locks are represented at the granularity of the document in a collection or key in an index. Document-level locking in |TokuMX| provides ``SERIALIZABLE`` isolation to writes, but operations only physically serialize—actually block each other—when they write to the same documents or index rows. These locks are held for the lifetime of the operation and are released when the operation’s underlying transaction commits or aborts.

Example:
Consider a collection with _id values ranging from 0 to 10000 and the following operation:

.. code-block:: javascript

  db.foo.update({_id: {$gte: 50, $lte: 5000}}, {$set: {c: 1 }}, {multi: true})

In basic |MongoDB|, this operation will hold a database-level write lock while documents in the range ``[50, 5000]`` are updated, but will periodically yield the lock to other readers and writers, allowing them to see partially-updated state.

In |TokuMX|, this operation will hold a database-level read lock (to protect it from database or collection drops, as described in :ref:`ddl_operations`) and lock only the documents in the range ``[50, 5000]``, never yielding those locks until the entire operation has completed. This means a concurrent update on {_id: 500} will need to wait until the original operation completes and releases its locks before it may proceed, but a concurrent update on ``{_id: 6000}`` can run without restriction.

.. _document-level_lock_conflicts:

Document-level Lock Conflicts
=============================

Operations that are waiting for :ref:`document-level_locks` will time out after a specified period (see :variable:`lockTimeout`) and return an error to the application, and should be retried.

If two operations need to modify the same two keys in an index but acquire those locks in different orders, this would create a deadlock. |TokuMX| indexes (including the :option:`primaryKey` index) include deadlock detection. If an operation needs to acquire a lock that would create a deadlock, it instead returns an error to the application (unblocking the other operation(s) involved in the potential deadlock) and should be retried.

.. _diagnosing_lock_conflicts:

Diagnosing Lock Conflicts
-------------------------

A large number of lock timeouts or deadlocks can be symptomatic of data modeling problems. |TokuMX| provides some tools for diagnosing the cause for excessive lock conflicts.

Server Logs
^^^^^^^^^^^

When a lock request times out or fails due to deadlock, some diagnostic information is written to the server's logfile:

Fields:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Meaning
   * - index   
     - string  
     - The index in which the lock conflict occurred (as database.collection.$indexName).
   * - requestingTxnid 
     - int 
     - The transaction id of the operation that aborted due to the lock conflict.
   * - blockingTxnid   
     - int 
     - The transaction id of the operation that was holding the lock on which the operations conflicted.
   * - bounds  
     - array   
     - Array of two keys that represent the start and end points of the conflicted lock's range. Often this is a point lock (on a single key), which is represented as a degenerate range with both endpoints equal.

.. note:: 
  If the blocking operation is still running, it will show up in ``db.currentOp()`` with the ``blockingTxnid`` as its ``rootTxnid``.

.. note:: 
  In ``bounds``, ``$primaryKey`` is a special field in the lock that describes the :option:`primaryKey` value |TokuMX| implicitly adds to each secondary index's key.

Example:

.. code-block:: text

  Mon Sep 16 11:40:33 [conn4] update test.a query: { a: 11.0 } update: { $set: { z: 11.0 } } nscanned:0 keyUpdates:0 exception: Lock not granted. Try restarting the transaction. code:16759 lockNotGranted: { index: "test.a.$a_1", requestingTxnid: 4398508, blockingTxnid: 4398504, bounds: [ { a: 11.0, $primaryKey: MinKey }, { a: 11.0, $primaryKey: ObjectId('5237155bf07c362d724fd883') } ] } locks(micros) r:4000419 4000ms

Here, we can see that an update to the collection ``test.a`` on ``{a: 11.0}`` with transaction id ``4398508`` failed to acquire the lock on that key in the index on ``{a: 1}`` because of the operation with transaction id ``4398504``.

Below, we'll see how to correlate this information with ``db.currentOp()`` information to figure out what the blocking operation was.

.. command:: db.showLiveTransactions()

:command:`db.showLiveTransactions()` lists all live transactions and the :ref:`document-level_locks` they currently hold. It returns a cursor, so ``db.showLiveTransactions().pretty()`` is also valid. Each element returned represents a live transaction, and includes a ``txnid`` and an array of ``rowLocks``.

Each element of ``rowLocks`` describes a lock held by the transaction:

Fields:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Meaning
   * - index   
     - string  
     - in which the lock is held (as ``database.collection.$indexName``)
   * - bounds  
     - array   
     - An array of two keys that represent the start and end points of the locked range

.. note:: 
  In ``bounds``, ``$primaryKey`` is a special field in the lock that describes the :option:`primaryKey` value |TokuMX| implicitly adds to each secondary index's key.

Example:

.. code-block:: javascript

  > db.showLiveTransactions().pretty()
  {
        "txnid" : 5048608,
        "rowLocks" : [
            {
                "index" : "sysbench.sbtest_11.$_id_",
                "bounds" : [
                    {
                        "_id" : NumberLong(8255)
                    },
                    {
                        "_id" : NumberLong(8255)
                    }
                ]
            },
            {
                "index" : "sysbench.sbtest_11.$k_1",
                "bounds" : [
                    {
                        "k" : NumberLong("3409118536147010569"),
                        "$primaryKey" : NumberLong(8255)
                    },
                    {
                        "k" : NumberLong("3409118536147010569"),
                        "$primaryKey" : NumberLong(8255)
                    }
                ]
             }
        ]
  }

In this example, we can see an operation which is using a secondary index to perform an update. It first queries the secondary index ``sysbench.sbtest_11.$k_1`` to find the row with ``{k: NumberLong("3409118536147010569")}`` (which grabs a lock), then uses that key to do another lookup in the :option:`primaryKey` called ``sysbench.sbtest_11.$_id_``, which also takes a lock on that key, where it does the update.

.. command:: db.showPendingLockRequests()

:command:`db.showPendingLockRequests()` lists all lock requests that are currently waiting behind another operation. It returns a cursor, so ``db.showPendingLockRequests().pretty()`` is also valid.

Each element has the same information and form as what's printed to the Server Logs. Refer to that section for a field reference. The entries in :command:`db.showPendingLockRequests()` also report the time the lock request began as started.

Example:

.. code-block:: javascript

  > db.showPendingLockRequests().pretty()
  {
        "index" : "stress_test0.coll.$_id_",
        "requestingTxnid" : 201536646,
        "blockingTxnid" : 201536637,
        "started" : ISODate("2013-09-17T18:03:18.805Z"),
        "bounds" : [
                {
                        "_id" : ObjectId("5238996659aac9d3335787a4")
                },
                {
                        "_id" : ObjectId("5238996659aac9d3335787a4")
                }
        ]
   }

In this example, we can see that operation with transaction id ``201536646`` is waiting for a point lock on an ``ObjectId`` in the :option:`primaryKey` index.

We can join this information with the information in ``db.currentOp()`` with a simple javascript snippet:

.. code-block:: javascript

  > db.showPendingLockRequests().map(function(obj) {
        obj.requestingOp = db.currentOp({rootTxnid: obj.requestingTxnid}).inprog[0];
        obj.blockingOp = db.currentOp({rootTxnid: obj.blockingTxnid}).inprog[0];
        return obj;
  })



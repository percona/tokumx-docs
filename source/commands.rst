.. _commands:

========
Commands
========
.. _new_commands:

New Commands
============

.. _transaction_commands:

Transaction Commands
--------------------

These commands are used to manage the lifetime of :ref:`multi-statement_transactions`. Please take note of the section on :ref:`drivers` when using these commands.

.. command:: beginTransaction

.. code-block:: javascript

  {
   beginTransaction: 1,
   isolation:        '<string>'
  }


Arguments:

========= ================= =============================================================================
Field     Type              Description
========= ================= =============================================================================
isolation string (optional) One of :ref:`mvcc` (default), :ref:`serializable`, or :ref:`read_uncommitted`
========= ================= =============================================================================

Begins a transaction associated with the current connection. Only one transaction may be live at a time on a connection.

Returns an error if there is already another live transaction for this connection.

Requires authentication (and authorization for write privileges) to create a :ref:`serializable` transaction.

In the :program:`mongo` shell, there is a helper function ``db.beginTransaction([isolation])`` that wraps this command.

Example:

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 1 }
  > db.beginTransaction()
  { "status" : "transaction began", "ok" : 1 }
  > db.foo.insert({_id : 2})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  > db.foo.insert({_id : 3})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  { "_id" : 3 }
  > db.rollbackTransaction()
  { "status" : "transaction rolled back", "ok" : 1 }
  > db.foo.find()
  { "_id" : 1 }

.. command:: commitTransaction

.. code-block:: javascript

  {
    commitTransaction: 1
  }

Commits the transaction associated with the current connection. This allows future queries to see this transaction's writes, logs this transaction's write operations to the oplog, and releases any :ref:`document-level_locks` held.

Returns an error if there is no live transaction for this connection.

In the :program:`mongo` shell, there is a helper function ``db.commitTransaction()`` that wraps this command.

Example:

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 1 }
  > db.beginTransaction()
  { "status" : "transaction began", "ok" : 1 }
  > db.foo.insert({_id : 2})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  > db.foo.insert({_id : 3})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  { "_id" : 3 }
  > db.commitTransaction()
  { "status" : "transaction committed", "ok" : 1 }
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  { "_id" : 3 }

.. command:: rollbackTransaction

.. code-block:: javascript

  {
    rollbackTransaction: 1
  }

Rolls back the transaction associated with the current connection. This undoes all of this transaction's writes and releases any :ref:`document-level_locks` held.

Returns an error if there is no live transaction for this connection.

In the :program:`mongo` shell, there is a helper function ``db.rollbackTransaction()`` that wraps this command.

Example:

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 1 }
  > db.beginTransaction()
  { "status" : "transaction began", "ok" : 1 }
  > db.foo.insert({_id : 2})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  > db.foo.insert({_id : 3})
  > db.foo.find()
  { "_id" : 1 }
  { "_id" : 2 }
  { "_id" : 3 }
  > db.rollbackTransaction()
  { "status" : "transaction rolled back", "ok" : 1 }
  > db.foo.find()
  { "_id" : 1 }

.. _loader_commands:

Loader Commands
---------------

These commands are used for controlling the :ref:`bulk_loader` to build collections and indexes. They are used transparently by :program:`mongorestore` and :program:`mongoimport` but can be used separately by clients as well.

The bulk loader commands must be used inside a :ref:`multi-statement_transactions`, and therefore cannot be used on a sharded cluster.

Example:
This is an example of using this API from the :program:`mongo` shell, using a :option:`primaryKey`, building one secondary index, and specifying some :ref:`collection_and_index_options` for both indexes.

.. code-block:: javascript

  > db.beginTransaction()
  { "status" : "transaction began", "ok" : 1 }
  > db.runCommand({beginLoad: 1,
  ...              ns:        'foo',
  ...              indexes:   [{key:          {x: 1, y: 1},
  ...                           ns:           'loader.foo',
  ...                           name:         'x_1_y_1',
  ...                           readPageSize: '8k'}],
  ...              options:   {compression: 'quicklz',
  ...                          primaryKey:  {a: 1, x: 1, _id: 1}}})
  { "status" : "load began", "ok" : true }
  > db.foo.insert([{a: 100, x: 'john', y: new Date()},
  ...              {a: 200, x: 'leif', y: new Date()},
  ...              {a: 300, x: 'zardosht', y: new Date()},
  ...              {a: 0, x: 'tim', y: new Date()}])
  > db.foo.insert({a: 400, x: 'bradley', y: new Date()})
  > db.runCommand({commitLoad: 1})
  { "status" : "load committed", "ok" : true }
  > db.commitTransaction()
  { "status" : "transaction committed", "ok" : 1 }
  > db.foo.getIndexes()
  [
      {
          "key" : {
              "a" : 1,
              "x" : 1,
              "_id" : 1
          },
          "unique" : true,
          "ns" : "loader.foo",
          "name" : "primaryKey",
          "clustering" : true,
          "compression" : "quicklz"
      },
      {
          "key" : {
              "_id" : 1
          },
          "unique" : true,
          "ns" : "loader.foo",
          "name" : "_id_",
          "compression" : "quicklz"
      },
      {
          "key" : {
              "x" : 1,
              "y" : 1
          },
          "ns" : "loader.foo",
          "name" : "x_1_y_1",
          "readPageSize" : "8k"
      }
  ]
  > db.foo.stats()
  {
          "ns" : "loadtest.foo",
          "count" : 5,
          "nindexes" : 3,
          "nindexesbeingbuilt" : 3,
          "size" : 432,
          "storageSize" : 16896,
          "totalIndexSize" : 558,
          "totalIndexStorageSize" : 33792,
          "indexDetails" : [
                  {
                          "name" : "primaryKey",
                          "count" : 5,
                          "size" : 432,
                          "avgObjSize" : 86.4,
                          "storageSize" : 16896,
                          "pageSize" : 4194304,
                          "readPageSize" : 65536,
                          "fanout" : 16,
                          "compression" : "quicklz",
                          "queries" : 0,
                          "nscanned" : 0,
                          "nscannedObjects" : 0,
                          "inserts" : 0,
                          "deletes" : 0
                  },
                  {
                          "name" : "_id_",
                          "count" : 5,
                          "size" : 271,
                          "avgObjSize" : 54.2,
                          "storageSize" : 16896,
                          "pageSize" : 4194304,
                          "readPageSize" : 65536,
                          "fanout" : 16,
                          "compression" : "quicklz",
                          "queries" : 0,
                          "nscanned" : 0,
                          "nscannedObjects" : 0,
                          "inserts" : 0,
                          "deletes" : 0
                  },
                  {
                          "name" : "x_1_y_1",
                          "count" : 5,
                          "size" : 287,
                          "avgObjSize" : 57.4,
                          "storageSize" : 16896,
                          "pageSize" : 4194304,
                          "readPageSize" : 8192,
                          "fanout" : 16,
                          "compression" : "zlib",
                          "queries" : 0,
                          "nscanned" : 0,
                          "nscannedObjects" : 0,
                          "inserts" : 0,
                          "deletes" : 0
                  }
          ],
          "ok" : 1
  }

.. command:: beginLoad

.. code-block:: javascript

  {
    beginLoad: 1,
    ns:        '<string>',
    indexes:   [<indexspec>, ...],
    options:   <document>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ns
     - string
     - The name of the collection to create (without the dbname. prefix).
   * - indexes	
     - array of documents	
     - An array of all indexes to create. Each element should be of the same form as the documents in the system.indexes collection.
   * - options	
     - document	
     - Creation options for the collection, as they would be specified to ``db.createCollection()``, described in :ref:`collection_options`.

Supported since 1.1.0

Creates the collection ``ns`` in a special bulk loading mode. In this mode, the connection that ran :command:`beginLoad` may send multiple insert operations, followed by either :command:`commitLoad` or :command:`abortLoad`. Other connections that try to use this collection will be rejected until the load is complete.

Example:

.. code-block:: javascript

  > db.beginTransaction()
  { "status" : "transaction began", "ok" : 1 }
  > db.runCommand({beginLoad: 1,
  ...              ns:        'foo',
  ...              indexes: [{key:          {x: 1, y: 1},
  ...                         ns:           'loader.foo',
  ...                         name:         'x_1_y_1',
  ...                         readPageSize: '8k'}],
  ...              options: {compression: 'quicklz',
  ...                        primaryKey:  {a: 1, x: 1, _id: 1}}})
  { "status" : "load began", "ok" : true }

.. command:: commitLoad

.. code-block:: javascript

  {
    commitLoad: 1
  }

Supported since 1.1.0

Commits the current bulk load in progress for this client connection. This includes the work of building all indexes for the collection, and it will block until that work is complete.

After this command returns, you should run :command:`commitTransaction` to make the collection visible to other client connections.

.. command:: abortLoad

.. code-block:: javascript

  {
    abortLoad: 1
  }

Supported since 1.1.0

Aborts the current bulk load in progress for this client connection. This removes the collection from the database, as if by ``db.collection.drop()`` and destroys all temporary state created by the loader.

If the client connection times out while a loader is active, the bulk load is automatically aborted.

.. _partitioned_collections_commands:

Partitioned Collections Commands
--------------------------------

.. command:: addPartition

.. code-block:: javascript

  {
    addPartition: '<collection>',
    newMax:       <document>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - addPartition	
     - string	
     - The collection to which to add a partition.
   * - newMax	
     - document (optional)	
     - Identifies what the maximum key of the current last partition should become before adding a new partition. This must be greater than any existing key in the collection. If newMax is not passed in, then the maximum key of the current last partition will be set to the key of the last element that currently exists in it.

Supported since 1.5.0

Add a partition to a :ref:`partitioned_collections`. This command is used for :ref:`adding_a_partition`.

In the :program:`mongo` shell, there is a helper function ``db.collection.addPartition([newMax])`` that wraps this command.

Example:

.. code-block:: javascript

  rs0:PRIMARY> db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(1),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:14:35.040Z")
          }
      ],
      "ok" : 1
  }
  rs0:PRIMARY> db.foo.insert({_id:10})
  rs0:PRIMARY> db.runCommand({addPartition: 'foo'})
  { "ok" : 1 }
  rs0:PRIMARY> db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(2),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "_id" : 10
              },
              "createTime" : ISODate("2014-06-17T21:14:35.040Z")
          },
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:15:59.156Z")
          }
      ],
      "ok" : 1
  }
  rs0:PRIMARY> db.runCommand({addPartition: 'foo', newMax: {_id : 20}})
  { "ok" : 1 }
  rs0:PRIMARY> db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(3),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "_id" : 10
              },
              "createTime" : ISODate("2014-06-17T21:14:35.040Z")
          },
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : 20
              },
              "createTime" : ISODate("2014-06-17T21:15:59.156Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:16:17.871Z")
          }
      ],
      "ok" : 1
  }

.. command:: dropPartition

.. code-block:: javascript

  {
    dropPartition: '<collection>',
    id:            <number>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - dropPartition	
     - string
     - The collection to get partition info from.
   * - id	
     - number (optional)	
     - The id of the partition to be dropped. Partition ids may be identified by running :command:`getPartitionInfo`. Note that while optional, either id or max must be present, but not both.
   * - max	
     - document (optional)	
     - Specifies the maximum partition key for dropping such that all partitions with documents less than or equal to max are dropped. Note that while optional, either id or max must be present, but not both.

Supported since 1.5.0

Drop the partition given a partition id. This command is used for :ref:`dropping_a_partition` of a :ref:`partitioned_collections`.

In the :program:`mongo` shell, there is a helper function ``db.collection.dropPartition([id])`` that wraps this command for the case where a partition id is specified. Similarly, there is also a helper function, ``db.collection.dropPartitionsLEQ([max])`` that wraps this command for the case where a max partition key is specified.

Example:

.. code-block:: javascript

  rs0:PRIMARY> db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(3),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "_id" : 10
              },
              "createTime" : ISODate("2014-06-17T21:04:39.241Z")
          },
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : 20
              },
              "createTime" : ISODate("2014-06-17T21:04:46.377Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:04:48.704Z")
          }
      ],
      "ok" : 1
  }
  rs0:PRIMARY> db.runCommand({dropPartition: 'foo', id: 0})
  { "ok" : 1 }
  rs0:PRIMARY> db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(2),
      "partitions" : [
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : 20
              },
              "createTime" : ISODate("2014-06-17T21:04:46.377Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:04:48.704Z")
          }
      ],
      "ok" : 1
  }

.. command:: getPartitionInfo

.. code-block:: javascript

  {
    getPartitionInfo: '<collection>'
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - getPartitionInfo	
     - string	
     - The collection to get partition info from.

Supported since 1.5.0

Retrieve the list of partitions for a partitioned collection. This command provides :ref:`information_about_partitioned_collections`.

In the :program:`mongo` shell, there is a helper function ``db.collection.getPartitionInfo()`` that wraps this command.

Example:

.. code-block:: javascript

  > db.runCommand({getPartitionInfo: 'foo'})
  {
      "numPartitions" : NumberLong(2),
      "partitions" : [
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : 20
              },
              "createTime" : ISODate("2014-06-17T21:04:46.377Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-17T21:04:48.704Z")
          }
      ],
      "ok" : 1
  }

.. _parameter_commands:

Parameter Commands
------------------

|TokuMX| provides the ability to view and change several of the :ref:`server_parameters` at runtime, through the commands :command:`getParameter` and :command:`setParameter`.

.. command:: getParameter

View the value of a server parameter at runtime.

.. code-block:: javascript

  {
    getParameter: 1,
    <option>: 1
  }

See also `getParameter <http://docs.mongodb.org/manual/reference/command/getParameter/>`_ in the |MongoDB| documentation.

In the |TokuMX| shell, there is also a wrapper function for :command:`getParameter`: ``db.getParameter(name)``.

Example:
The syntax to view the server parameter :variable:`checkpointPeriod` in the shell is:

.. code-block:: javascript
  
  db.getParameter('checkpointPeriod')

.. command:: setParameter

Modify a server parameter at runtime.

.. code-block:: javascript

  {
    setParameter: 1,
    <option>: <value>
  }

See also `setParameter <http://docs.mongodb.org/manual/reference/command/setParameter/>`_ in the |MongoDB| documentation.

In the |TokuMX| shell, there is also a wrapper function for :command:`setParameter`: ``db.setParameter(name, value)``.

Example:
The syntax to modify the server parameter :variable:`checkpointPeriod` to ``120`` in the shell is:

.. code-block:: javascript

  db.setParameter('checkpointPeriod', 120)

.. note::
  Modifying a parameter returns the pre-existing value.

.. _locking_commands:

Locking Commands
----------------

|TokuMX| has some commands for controlling and viewing the behavior of both :ref:`metadata_locks` and :ref:`document-level_locks`.

.. command:: setClientLockTimeout

.. code-block:: javascript

  {
    setClientLockTimeout: <number>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - setClientLockTimeout	
     - number	
     - New value for this client's lock timeout (in milliseconds).

Supported since 1.5.0

The :variable:`lockTimeout` (used for :ref:`document-level_locks`) can be changed for each individual client connection, if needed.  

Returns the old value for this connection.

Example:

.. code-block:: javascript

  > db.runCommand({setClientLockTimeout: 10000})
  { "was" : 4000, "ok" : 1 }

.. command:: setWriteLockYielding

.. code-block:: javascript

  {
    setWriteLockYielding: <boolean>
  }


Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - setWriteLockYielding	
     - Boolean	
     - New value for this client's write lock yielding setting.

Supported since 1.5.0

The :variable:`forceWriteLocks` setting can be controlled (for read locks) for each individual client connection, if needed. This affects the behavior of per-database :ref:`metadata_locks`.

The default value for each new connection is the same as the value of :variable:`forceWriteLocks`. If this setting is true for a particular connection, then that connection's read locks will yield to any pending write locks.

Example:

.. code-block:: javascript

  > db.runCommand({setWriteLockYielding: true})
  { "was" : false, "ok" : 1 }

.. command:: showLiveTransactions

.. code-block:: javascript

  {
    showLiveTransactions: 1,
    cursor:       <document>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - cursor	
     - document (optional)
     - If present, requests that the command return a cursor. The cursor allows more results to be returned. The cursor document may specify options that control the creation of the cursor document.

Supported since 1.2.1

Lists all live transactions, and the :ref:`document-level_locks` each one currently holds. The reported information is described in :command:`db.showLiveTransactions()`.

.. command:: showPendingLockRequests

.. code-block:: javascript

  {
    showPendingLockRequests: 1,
    cursor:                  <document>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - cursor	
     - document (optional)	
     - If present, requests that the command return a cursor. The cursor allows more results to be returned. The cursor document may specify options that control the creation of the cursor document.

Supported since 1.2.1

Lists all pending requests for :ref:`document-level_locks`. The reported information is described in :command:`db.showPendingLockRequests()`.

.. _replication_commands:

Replication Commands
--------------------

.. command:: replAddPartition

.. code-block:: javascript

  {
    replAddPartition: 1
  }

Supported since 1.4.0

Adds a partition to the ``oplog.rs`` and ``oplog.refs`` collection.

Returns an error if the current last partition has no oplog entries, and as a result, the adding of the partition fails.

Requires authentication and cluster admin write privileges.

.. command:: replGetExpireOplog

.. code-block:: javascript

  {
    replGetExpireOplog: 1
  }

Supported since 1.0.4

Retrieve the values of :variable:`expireOplogDays` and :variable:`expireOplogHours`.

Requires authentication and cluster admin read privileges.

Example:

.. code-block:: javascript

  rs0:PRIMARY> db.adminCommand('replGetExpireOplog')
  { "expireOplogDays" : 14, "expireOplogHours" : 0, "ok" : 1 }

.. command:: replSetExpireOplog

.. code-block:: javascript

  {
    replSetExpireOplog: 1,
    expireOplogDays:    <number>,
    expireOplogHours:   <number>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - expireOplogDays	
     - number	
     - Specifies the number of days to keep oplog data.
   * - expireOplogHours	
     - number	
     - Specifies the number of hours to keep oplog data.

Supported since 1.0.4

Set the amount of oplog data, in time, that is saved. Any oplog data older than the specified amount of time may be trimmed and therefore removed by a background thread.

Requires authentication and cluster admin write privileges.

Example:

.. code-block:: javascript

  rs0:PRIMARY> db.adminCommand({replSetExpireOplog: 1, expireOplogDays: 4, expireOplogHours: 0})
  { "ok" : 1 }
  rs0:PRIMARY> db.adminCommand('replGetExpireOplog')
  { "expireOplogDays" : 4, "expireOplogHours" : 0, "ok" : 1 }

.. command:: replTrimOplog

.. code-block:: javascript

  {
    replTrimOplog: 1,
    ts:            <date>
  }
  // or
  {
    replTrimOplog: 1,
    gtid:          <GTID>
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - ts	
     - ISODate
     - Timestamp to which the oplog is to be trimmed. Partitions known not to contain entries created after this timestamp.
   * - gtid	
     - BinData	
     - GTID to which the oplog is to be trimmed. Partitions known not to contain entries greater than this GTID are dropped. The GTID must be a valid 16 byte entry that gets saved in the _id field in the oplog.

.. note::
  Either ``ts`` or ``gtid`` must be passed in, but not both.

Supported since 1.4.0

Trim the oplog by dropping partitions up to the specified GTID or specified date. Trimming is performed by dropping partitions. Partitions that are known to not contain entries greater than the specified GTID of date are dropped.

Requires authentication and cluster admin write privileges.

Example using a date:

.. code-block:: javascript

  rs0:PRIMARY> rs.oplogPartitionInfo()
  {
      "numPartitions" : NumberLong(4),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAEeg==")
              },
              "createTime" : ISODate("2014-06-13T20:09:56.819Z")
          },
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFCg==")
              },
              "createTime" : ISODate("2014-06-14T20:10:01.847Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFmg==")
              },
              "createTime" : ISODate("2014-06-15T20:10:02.200Z")
          },
          {
              "_id" : NumberLong(3),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-16T20:10:02.543Z")
          }
      ],
      "ok" : 1
  }
  rs0:PRIMARY> db.adminCommand({replTrimOplog: 1, ts: ISODate('2014-06-14T20:10:01.847Z')})
  { "ok" : 1 }
  rs0:PRIMARY> rs.oplogPartitionInfo()
  {
      "numPartitions" : NumberLong(3),
      "partitions" : [
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFCg==")
              },
              "createTime" : ISODate("2014-06-14T20:10:01.847Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFmg==")
              },
              "createTime" : ISODate("2014-06-15T20:10:02.200Z")
          },
          {
              "_id" : NumberLong(3),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-16T20:10:02.543Z")
          }
      ],
      "ok" : 1
  }

Example using a GTID:

.. code-block:: javascript

  rs0:PRIMARY> rs.oplogPartitionInfo()
  {
      "numPartitions" : NumberLong(3),
      "partitions" : [
          {
              "_id" : NumberLong(1),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFCg==")
              },
              "createTime" : ISODate("2014-06-14T20:10:01.847Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFmg==")
              },
              "createTime" : ISODate("2014-06-15T20:10:02.200Z")
          },
          {
              "_id" : NumberLong(3),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-16T20:10:02.543Z")
          }
      ],
      "ok" : 1
  }
  rs0:PRIMARY> db.runCommand({replTrimOplog:1, gtid : BinData(0,"AAAAAAAAAAEAAAAAAAAFCg==")})
  { "ok" : 1 }
  rs0:PRIMARY> rs.oplogPartitionInfo()
  {
      "numPartitions" : NumberLong(2),
      "partitions" : [
          {
              "_id" : NumberLong(2),
              "max" : {
                  "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAFmg==")
              },
              "createTime" : ISODate("2014-06-15T20:10:02.200Z")
          },
          {
              "_id" : NumberLong(3),
              "max" : {
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-06-16T20:10:02.543Z")
          }
      ],
      "ok" : 1
  }

.. _plugin_commands:

Plugin Commands
----------------

Plugins are dynamically loadable modules that add extra functionality to |TokuMX|, for example, :ref:`hot_backup`. These commands are used to control which plugins are loaded into the server.

.. command:: listPlugins

.. code-block:: javascript

  {
    listPlugins: 1
  }

Supported since 1.1.0

Lists which plugins are currently loaded, and information about them.

Example:

.. code-block:: javascript

  > db.adminCommand('listPlugins')
  {
      "plugins" : [
          {
              "filename" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
              "fullpath" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
              "name" : "backup_plugin",
              "version" : "tokubackup 1.1 $Revision: 56100 $",
              "checksum" : "688f63cb3018caa3efb74ff829ee3568",
              "commands" : [
                  "backupStart",
                  "backupThrottle",
                  "backupStatus"
              ]
          }
      ],
      "ok" : 1
  }

.. command:: _loadPlugin

.. code-block:: javascript

  {
    loadPlugin: '<name>',
    checksum:   '<string>'
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - loadPlugin	
     - string	
     - Name of the plugin to be loaded (searches for ``libname.so``)
   * - checksum	
     - string (optional)	
     - Checksum of the plugin to verify (aborts loading if the checksum doesn't match).

Supported since 1.1.0

Loads a new plugin by name. The :variable:`pluginsDir` is searched for a file named ``libname.so``, and if such a file is found, it is loaded as a plugin.

If ``checksum`` is provided, the plugin's checksum is verified before it is loaded.

If the plugin is successfully loaded, its information is returned, just as would be reported by :command:`listPlugins`.

Example:

.. code-block:: javascript

  > db.adminCommand({loadPlugin: 'backup_plugin'})
  {
      "loaded" : {
          "filename" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
          "fullpath" : "/opt/tokumx/lib64/plugins/libbackup_plugin.so",
          "name" : "backup_plugin",
          "version" : "tokubackup 1.1 $Revision: 56100 $",
          "checksum" : "688f63cb3018caa3efb74ff829ee3568",
          "commands" : [
              "backupStart",
              "backupThrottle",
              "backupStatus"
          ]
      },
      "ok" : 1
  }

.. command:: unloadPlugin

.. code-block:: javascript

  {
    unloadPlugin: '<string>'
  }

Arguments:

.. list-table::
   :header-rows: 1

   * - Field
     - Type
     - Description
   * - unloadPlugin	
     - string	
     - Name of the plugin to unload (see :command:`loadPlugin`).

Supported since 1.1.0

Unloads the named plugin, removing its functionality from the server.

Example:

.. code-block:: javascript

  > db.adminCommand({unloadPlugin: 'backup_plugin'})
  { "ok" : 1 }

.. _hot_backup_commands:

Hot Backup Commands
===================
These commands are part of the Hot Backup component available only in TokuMX Enterprise Edition.

.. command:: backupStart

   :field: destination, the directory where the backup files will reside.
   :type: string


.. code-block:: javascript

  {
  backupStart: '<destination>'
  }

Runs a :ref:`hot_backup`. This copies the dbpath to destination online, and leaves the files with contents identical to what was committed to disk at the moment the :command:`backupStart` command returns.

.. note::
  For more information about how backup works, see :ref:`hot_backup`.

Returns an error if there is already another backup operation running.

The backup destination must be a directory that exists, and should not be a subdirectory of dbpath.

If a separate :variable:`logDir` is used from ``dbpath``, then destination will contain two directories, data (containing the contents of ``dbpath``) and log (containing the contents of :variable:`logDir`).

.. note::
 Since Hot Backup copies data recursively, if :variable:`logDir` is a subdirectory of dbpath, all data is copied directly in to destination.

Example:

.. code-block:: javascript

  > var d = new Date()
  > var month = (d.getMonth() < 9 ? '0' : '') + (d.getMonth() + 1)
  > var backupName = 'tokumx-' + d.getFullYear() + month + d.getDate()
  > db.runCommand({backupStart: '/mnt/backup/' + backupName})
  { "ok" : 1 }


.. command:: backupStatus

.. code-block:: javascript

  {
  backupStatus: 1
  }

Queries the :ref:`hot_backup` system for the status of a running backup operation, if one is running.

Returns an error if there is no hot backup operation in progress.

Example:

.. code-block:: javascript

  > db.runCommand('backupStatus')
  {
        "percent" : 22.522784769535065,
        "bytesDone" : NumberLong(16875520),
        "files" : {
                "done" : 5,
                "total" : 20
        },
        "current" : {
                "source" : "/var/lib/tokumx/log000000000004.tokulog27",
                "dest" : "/mnt/backup/tokumx-demo/log000000000004.tokulog27",
                "bytes" : {
                        "done" : NumberLong(16777216),
                        "total" : NumberLong(57805156)
                }
        },
        "ok" : 1
  }

.. command:: backupThrottle

  :field: rate
  :type: integer or string (bytes)

.. code-block:: javascript

  {
  backupThrottle: <rate>
  }

The rate (bytes per second) at which the Hot Backup system will use I/O to copy files, ignoring client write activity. May use "K/M/G" suffix as a string.

The :ref:`hot_backup` system uses I/O in two ways: for mirroring writes, and for copying files (see Concepts for more details). Mirrored writes must be completed immediately, but file copying can be slow.

This command controls how much I/O (in bytes per second) is used for file copying. By default, backups do not limit themselves this way, but throttling the backup operation can help reduce the impact on a running server.

Example:

.. code-block:: javascript

  > db.runCommand({backupThrottle: '10MB'})
  { "ok" : 1 }

.. _pitr_commands:

Point in Time Recovery Commands
===============================

This command is part of the :ref:`pitr_plugin` component available only in |TokuMX| Enterprise Edition.

.. command:: recoverToPoint

.. code-block:: javascript

  {
    recoverToPoint: 1,
    ts:             <date>
  }
  // or
  {
    recoverToPoint: 1,
    gtid:           <GTID>
  }

===== ======= =================================================
Field Type    Description
===== ======= =================================================
ts    ISODate Timestamp to which the server is to be recovered.
gtid  BinData GTID to which the server is to be recovered.
===== ======= =================================================

Supported since 2.0.0

Runs :ref:`pitr_plugin`. This syncs and applies all entries from another replica set member's oplog up to the provided timestamp or ``GTID``.

.. note::
  For more information about how point in time recovery works, see :ref:`pitr_plugin`.

The server must be a member of a replica set, and must be in `maintenance mode <http://docs.mongodb.org/manual/reference/command/replSetMaintenance/>`_. To bring up a server in maintenance mode (to make sure it doesn't sync anything immediately on startup), use the server parameter :variable:`rsMaintenance`.

.. warning::
  Do not run multiple instances of :variable:`recoverToPoint` concurrently.

Example:

.. code-block:: javascript

  rs0:RECOVERING> db.runCommand({recoverToPoint: 1, gtid: GTID(1, 152)})
  { "ok" : 1 }


.. _adminstrative_commands:

Administrative Commands
-----------------------

.. command:: checkpoint

.. code-block:: javascript

 {
   checkpoint: 1
 }
 
Forces |TokuMX| to run a checkpoint immediately, rather than waiting for the :variable:`checkpointPeriod` timer to expire.

.. command:: engineStatus

.. code-block:: javascript

  {
    engineStatus: 1
  }

Retrieves raw status information from the Fractal Tree indexing engine. This is generally for diagnostic/development use only. This information is aggregated and presented in a more user-friendly form in `db.serverStatus() <http://docs.mongodb.org/manual/reference/server-status/>`_, more details of TokuMX-specific information there is available in the :ref:`server_status` section.

.. _internal-only_commands:

Internal-only Commands
----------------------

.. command:: _collectionsExist

Supported since 1.1.0

Internal use only.

.. command:: _migrateStartCloneTransaction

Supported since 1.4.0

Internal use only.

.. command:: clonePartitionInfo

Supported since 1.5.0

Internal use only.

.. command:: logReplInfo

Supported since 1.3.2

Internal use only.

.. command:: replUndoOplogEntry

Supported since 1.3.2

Internal use only.

.. command:: showSizes

Supported since 1.5.0

Internal use only.

.. command:: updateSlave

Internal use only.

Deprecated Commands
===================

* clean

  Deprecated internal |MongoDB| command.

* cloneCollectionAsCapped
  
  Capped collections are deprecated in favor of :ref:`partitioned_collections`.

* closeAllDatabases

  Deprecated internal |MongoDB| command.

* collMod

  |TokuMX| Fractal Tree indexes do not suffer the fragmentation problems of MongoDB's data storage, so the ``powerOf2Sizes`` option is deprecated, as well as this command. TTL indexes are also deprecated in favor of :ref:`partitioned_collections`.

* compact

  |TokuMX| Fractal Tree indexes do not fragment nor do they corrupt themselves in the way that MongoDB's indexes do, so this command is unneeded.

* convertToCapped

  Capped collections are deprecated in favor of :ref:`partitioned_collections`.

* godinsert

  Deprecated internal |MongoDB| command.

* journalLatencyTest

  The |TokuMX| transaction log is different from the |MongoDB| journal and does not need this command.

* logRotate

  You should instead use ``SIGUSR1`` to rotate logs.

* repairDatabase

  |TokuMX| Fractal Tree indexes do not fragment nor do they corrupt themselves in the way that MongoDB's indexes do, so this command is unneeded.

* validate

  |TokuMX| Fractal Tree indexes are different from MongoDB's B-tree indexes and do not need to be validated the same way.



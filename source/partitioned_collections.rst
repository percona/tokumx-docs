.. _partitioned_collections:

=======================
Partitioned Collections
=======================

Partitioned collections are collections that are divided into a number of sub-collections, or partitions, such that each partition is its own set of files on disk. Conceptually, a partitioned collection is similar to a `partitioned tables <http://dev.mysql.com/doc/refman/5.5/en/partitioning.html>`_ in MySQL.

The primary advantage of partitioned collections is the ability to delete a partition efficiently and immediately reclaim its storage space in a manner similar to dropping a collection in TokuMX, or a database in basic MongoDB. All that's required is removing some files.

In all other ways, a partitioned collection operates logically the same as a normal collection, including complex queries with sort and aggregation. The server does the required work to merge the results from all partitions, so the partitioning is invisible from the client's perspective.

Use Cases
Time-series data is the most obvious use case for partitioned collections. The typical pattern is to insert the newest data in the most recent partition, and periodically drop the oldest partition once it has passed some expiry time.

Partitioned collections make excellent replacements for capped collections and TTL indexes. Both of those strategies incur individual queries and deletes for each document that expires, while partitioned collections delete data in large batches. As such, they are much faster than capped collections or collections with TTL indexes. At the same time, they are more flexible and easier to manage and predict than either capped collections or TTL indexed collections.

For example, suppose you have some data being generated over time (financial data is a common such source), of which you would like to keep only the last year's worth. With a capped collection, you need to analyze your insertion rate and guess the appropriate size for the collection, and this size is then hard to change later. With a partitioned collection, you just create a new partition each month and drop the oldest. In this way you have the most recent 12 or 13 months of data, but this can be adjusted in the application layer later.

If you have a use case for a TTL index on a collection, and all documents will have the same "time to live", you should instead strongly consider making that collection partitioned. A TTL index incurs frequent queries on that index and deletes documents one at a time, whereas a partitioned collection is much faster and lighter weight.

Currently, the ``oplog.rs`` and ``oplog.refs`` collections, both of which are used for replication, are partitioned collections.

Concepts
========

Like a capped collection, a partitioned collection must be declared partitioned when the collection is first created, and it must be created explicitly.

Partitioned collections are always divided according to the :option:`primaryKey`. Each partition holds documents less than a certain :option:`primaryKey` value, in this way each partition holds a distinct range of documents according to the :option:`primaryKey`.

Since the :option:`primaryKey` must end in ``{_id: 1}``, the partition endpoints must as well.

.. _creating_partitioned_collections:

Creating Partitioned Collections
================================

There are two ways to create a partitioned collection.

Use the `create <http://docs.mongodb.org/manual/reference/command/create/#dbcmd.create>`_ command, and add the option :option:`partitioned`.

Use the :program:`mongo` shell provided with |TokuMX|, and add partitioned to the options document of `db.createCollection() <http://docs.mongodb.org/manual/reference/method/db.createCollection/>`_.

Since partitioned collections use the :option:`primaryKey` as the partitioning key, most applications will want to pick a custom :option:`primaryKey`, which should also be done at create time.

Example:
Suppose we want to create a collection prices, that is partitioned by a time field.

In the :program:`mongo` shell, we can run:

.. code-block:: javascript

  > db.createCollection('prices', {primaryKey:  {time: 1, _id: 1},
                                   partitioned: true})

Alternatively, from another client, we could run this command:

.. code-block:: javascript

  {
      create:      "prices",
      primaryKey:  {time: 1, _id: 1},
      partitioned: true
  }

After creation, a new partitioned collection has one partition that will be used for all data. To break up the partitioned collection, use the commands explained in :ref:`adding_a_partition` and :ref:`dropping_a_partition`.

.. _information_about_partitioned_collections:

Information About Partitioned Collections
=========================================

|TokuMX| provides information about partitioned collections via the shell function ``db.collection.getPartitionInfo()``, which is a wrapper around the :command:`getPartitionInfo` command.

This command returns the number of partitions for the collection (``numPartitions``) and an array of objects (``partitions``), one describing each partition. For each partition, the object includes:

* ``_id``: An integer identifier for the partition, used by :command:`dropPartition` to drop the partition.

* ``max``: The maximum value (of the :option:`primaryKey` this partition may hold. A partition includes all documents with primary key less than or equal to its ``max``, and greater than the ``max`` for the previous partition.

* ``createTime``: The time this partition was created.

Example:
For a newly created partitioned collection, ``foo``, that has only one partition, :command:`getPartitionInfo` shows the following:

.. code-block:: javascript

  > db.foo.getPartitionInfo()
  {
      "numPartitions" : NumberLong(1),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "a" : { "$maxKey" : 1 },
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-04-03T02:03:15.542Z")
          }
      ],
      "ok" : 1
  }

Note there is one partition and the maximum value of that partition is a special value "maxKey", which is the greatest possible key.

After adding two partitions to this collection, :command:`getPartitionInfo` returns:

.. code-block:: javascript

  > db.foo.getPartitionInfo()
  {
      "numPartitions" : NumberLong(3),
      "partitions" : [
          {
              "_id" : NumberLong(0),
              "max" : {
                  "a" : 1000,
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-04-03T02:24:21.104Z")
          },
          {
              "_id" : NumberLong(1),
              "max" : {
                  "a" : 2000,
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-04-03T02:26:31.044Z")
          },
          {
              "_id" : NumberLong(2),
              "max" : {
                  "a" : { "$maxKey" : 1 },
                  "_id" : { "$maxKey" : 1 }
              },
              "createTime" : ISODate("2014-04-03T02:26:34.563Z")
          }
      ],
      "ok" : 1
  }

In this example, any document with ``a <= 1000`` will be found in the first partition (``_id`` 0), a document with ``a <= 2000`` will be in the second, and all other documents will be in the third.

Changes to db.collection.stats()
--------------------------------

Additionally, for a partitioned collection, the output of ``db.collection.stats()`` and the `collStats <http://docs.mongodb.org/manual/reference/command/collStats/>`_ command adds some information about individual partitions.

The result has an additional array ``partitions`` that contains essentially the `collStats <http://docs.mongodb.org/manual/reference/command/collStats/>`_ results for each individual partition, and the top-level information is an aggregation of the info for all partitions.

Example:

.. code-block:: javascript

  > db.foo.stats().partitions
  [
      {
          "count" : 1001,
          "nindexes" : 2,
          "nindexesbeingbuilt" : 2,
          "size" : 47047,
          "storageSize" : 65536,
          "totalIndexSize" : 34034,
          "totalIndexStorageSize" : 40960,
          "indexDetails" : [
              {
                  "name" : "primaryKey",
                  "count" : 1001,
                  "size" : 47047,
                  "avgObjSize" : 47,
                  "storageSize" : 65536,
                  "pageSize" : 4194304,
                  "readPageSize" : 65536,
                  "fanout" : 16,
                  "compression" : "zlib",
                  "queries" : 1,
                  "nscanned" : 0,
                  "nscannedObjects" : 0,
                  "inserts" : 1001,
                  "deletes" : 0
              },
              {
                  "name" : "_id_",
                  "count" : 1001,
                  "size" : 34034,
                  "avgObjSize" : 34,
                  "storageSize" : 40960,
                  "pageSize" : 4194304,
                  "readPageSize" : 65536,
                  "fanout" : 16,
                  "compression" : "zlib",
                  "queries" : 1,
                  "nscanned" : 0,
                  "nscannedObjects" : 0,
                  "inserts" : 1001,
                  "deletes" : 0
              }
          ]
      },
      {
          "count" : 999,
          "nindexes" : 2,
          "nindexesbeingbuilt" : 2,
          "size" : 46953,
          "storageSize" : 65536,
          "totalIndexSize" : 33966,
          "totalIndexStorageSize" : 40960,
          "indexDetails" : [
              {
                  "name" : "primaryKey",
                  "count" : 999,
                  "size" : 46953,
                  "avgObjSize" : 47,
                  "storageSize" : 65536,
                  "pageSize" : 4194304,
                  "readPageSize" : 65536,
                  "fanout" : 16,
                  "compression" : "zlib",
                  "queries" : 1,
                  "nscanned" : 0,
                  "nscannedObjects" : 0,
                  "inserts" : 999,
                  "deletes" : 0
              },
              {
                  "name" : "_id_",
                  "count" : 999,
                  "size" : 33966,
                  "avgObjSize" : 34,
                  "storageSize" : 40960,
                  "pageSize" : 4194304,
                  "readPageSize" : 65536,
                  "fanout" : 16,
                  "compression" : "zlib",
                  "queries" : 1,
                  "nscanned" : 0,
                  "nscannedObjects" : 0,
                  "inserts" : 999,
                  "deletes" : 0
              }
          ]
      }
  ]

.. _adding_a_partition:

Adding a Partition
==================

A new, empty partition may be added after the last partition (the one containing the largest documents according to :option:`primaryKey`). Existing partitions cannot be modified to create another partition. That is, an existing partition, including the first, cannot be "split".

Partitions are added with the ``db.collection.addPartition()`` shell wrapper, or the :command:`addPartition` command.

When adding a partition, the previously last partition is "capped" by assigning a new max value that must be larger than the key of any document already in the collection, and a new partition is created with a ``max`` value of that special ``"maxKey"`` value that is larger than any key.

.. warning:: 
  When querying a partitioned collection, the server maintains separate cursors on all existing partitions that are relevant to the query. Since adding and dropping partitions changes which partitions may be relevant, the act of adding or dropping a partition invalidates any existing cursors on the collection.

  This will return an error to any client which has not retrieved all results, and clients should be prepared to retry their queries in this case.

The ``max`` value chosen to "cap" the previous partition can be specified as the ``newMax`` parameter to :command:`addPartition`, or if not provided, the server will automatically use the largest key of any document currently in in the collection for that ``max`` value.

.. note:: 
  It is considered an error to add a partition without specifying the ``newMax`` parameter if the last partition is currently empty.

Example:
Consider a collection ``foo``, with two partitions, each containing one document:

.. code-block:: javascript

  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(0),
          "max" : {
              "a" : 1000,
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:18.705Z")
      },
      {
          "_id" : NumberLong(1),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:33.967Z")
      }
  ]
  > db.foo.find()
  { "_id" : 0, "a" : 500 }
  { "_id" : 1, "a" : 1500 }

Adding a partition with a default ``max``:

If we run :command:`addPartition` without providing ``newMax``, the server will choose the largest key, ``{a: 1500, _id: 1}`` to cap the second partition:

.. code-block:: javascript

  > db.foo.addPartition()
  { "ok" : 1 }
  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(0),
          "max" : {
              "a" : 1000,
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:09:20.109Z")
      },
      {
          "_id" : NumberLong(1),
          "max" : {
              "a" : 1500,
              "_id" : 1
          },
          "createTime" : ISODate("2014-04-03T03:09:24.680Z")
      },
      {
          "_id" : NumberLong(2),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:11:36.072Z")
      }
  ]

Adding a partition with a custom ``max``:

Instead, we can provide a different key to cap the second partition:

.. code-block:: javascript

  > db.foo.addPartition({a: 1600})
  { "ok" : 1 }
  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(0),
          "max" : {
              "a" : 1000,
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:18.705Z")
      },
      {
          "_id" : NumberLong(1),
          "max" : {
              "a" : 1600,
              "_id" : null
          },
          "createTime" : ISODate("2014-04-03T03:00:33.967Z")
      },
      {
          "_id" : NumberLong(2),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:05:34.067Z")
      }
  ]

.. note:: 
  A custom value for max must be a valid :option:`primaryKey` and:

    * Be greater than the max of all existing partitions except the last one (which is the one getting the new max).

    * Be greater than any :option:`primaryKey` that exists in the collection.

.. _dropping_a_partition:

Dropping a Partition
====================

Partitions may also be dropped. This deletes all documents in that partition's range by unlinking the underlying data and index files.

.. note:: 

  Any partition of a collection may be dropped. Most time-series applications using partitioned collections will drop the first partition, but this is not the only option.

  When dropping the last partition (holding the largest documents by :option:`primaryKey`) of a collection, the previous partition's max is changed to be the special "maxKey" element.

There are two methods for dropping partitions, using different invocations of the :command:`dropPartition` command:

* Drop a single partition by its ``_id`` (see :ref:`information_about_partitioned_collections`).

* Drop all partitions whose ``max`` value is less than or equal to some key.

.. warning:: 
  If a collection only has one partition, it is considered an error to drop that partition. Instead, simply drop the collection with ``db.collection.drop()``.

Example:

Consider a collection ``foo``, with two partitions, each containing one document (note the partitions' ``_ids``):

.. code-block:: javascript

  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(4),
          "max" : {
              "a" : 1000,
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:18.705Z")
      },
      {
          "_id" : NumberLong(6),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:33.967Z")
      }
  ]
  > db.foo.find()
  { "_id" : 0, "a" : 500 }
  { "_id" : 1, "a" : 1500 }

Dropping a partition by ``_id``:

To drop the first partition by ``_id``, use the :program:`mongo` shell:

.. code-block:: javascript

  > db.foo.dropPartition(4)
  { "ok" : 1 }
  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(6),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:33.967Z")
      }
  ]
  > db.foo.find()
  { "_id" : 1, "a" : 1500 }

This can be done in other clients by running the command ``{dropPartition: "foo", id: 4}``.

Dropping a partition by key:

To drop the first partition using a key, use the :program:`mongo` shell:

.. code-block:: javascript

  > db.foo.dropPartitionsLEQ({a: 1000, _id: MaxKey})
  { "ok" : 1 }
  > db.foo.getPartitionInfo().partitions
  [
      {
          "_id" : NumberLong(6),
          "max" : {
              "a" : { "$maxKey" : 1 },
              "_id" : { "$maxKey" : 1 }
          },
          "createTime" : ISODate("2014-04-03T03:00:33.967Z")
      }
  ]
  > db.foo.find()
  { "_id" : 1, "a" : 1500 }

This can be done in other clients by running the command ``{dropPartition: "foo", max: {a: 1000, _id: MaxKey}}``.

.. _secondary_indexes:

Secondary Indexes
=================

Partitioned collections support secondary indexes. Each partition maintains its own set of secondary indexes.

Queries that use secondary indexes on a partitioned collection may return results in a different order than expected, since the secondary index is partitioned. If the query contains a $sort clause, the query will merge results properly and honor the requirements of the ``$sort``.

If the :option:`primaryKey` can be used to exclude some partitions from consideration for a query, that will be done. A query that only needs to search one partition will be faster than one that searches all partitions.

Unique indexes other than the :option:`primaryKey` are not supported, because unique checks across partitions are not enforced.

Background indexing on partitioned collections is not supported.

.. _sharding_of_partitioned_collections:

Sharding of Partitioned Collections
===================================

Supported since 2.0.0

Beginning in version 2.0, |TokuMX| supports limited sharding of partitioned collections. Partitioned collections may be sharded subject to the following restrictions:

* The balancer should be disabled for the collection. Disable the balancer for the collection using `sh.disableBalancing() <http://docs.mongodb.org/manual/reference/method/sh.disableBalancing/#sh.disableBalancing>`_.

* Chunks should be migrated manually, via `sh.moveChunk() <http://docs.mongodb.org/manual/reference/command/moveChunk/>`_, preferably while empty.

* Chunks will not be automatically split by the server. It is best to manually pre-split chunks with `sh.splitAt() <http://docs.mongodb.org/manual/reference/method/sh.splitAt/#sh.splitAt>`_ and also move them, while the collection is empty.

The :command:`dropPartition` command will not accept a partition ``_id``. Instead, a maximum partition key must be used to drop partitions. Commands :command:`addPartition` and :command:`getPartitionInfo` work as expected through :program:`mongos`.

Such a collection has two special keys, the `shard key <http://docs.mongodb.org/manual/core/sharding-shard-key/>`_ and the partition key, which must be the :option:`primaryKey`. These must both be chosen at creation time.

.. tip:: 
  A collection that is sharded and partitioned should have a different shard key from its partition key. For best results, the partition key should be sequential (e.g. timestamp or ObjectID), and inserts should be spread out evenly according to the shard key. Sequential writes to the partition key allow old data to be dropped, and more random writes to the shard key help writes scale out to all shards.

  As it is recommended not to split or migrate non-empty chunks, it is usually a good idea to use a non-clustering shard key.

.. _creating_partition:

Creating
--------

To create a sharded and partitioned collection, use the `shardCollection <http://docs.mongodb.org/manual/reference/command/shardCollection/#dbcmd.shardCollection>`_ command to create a non-existent collection. This command accepts a new field, ``partitionKey``, which determines the :option:`primaryKey`, and therefore the partition key, for the collection.

.. note:: 
  As for unsharded partitioned collections, the partition key needs to be a valid :option:`primaryKey`, and as such must end in ``{_id: 1}``.

Example:

For example, to create a sharded and partitioned collection that shards the data on symbol id, ``{sid: 1}``, and partitions the data by a timestamp, ``{ts: 1}``, do the following:

.. code-block:: javascript

  mongos> db.adminCommand({shardCollection: "test.mycoll",
  ...                      key:             {sid: 1},
  ...                      partitionKey:    {ts: 1, _id: 1},
  ...                      clustering:      false})
  { "collectionsharded" : "test.mycoll", "ok" : 1 }
  mongos> sh.disableBalancing("test.mycoll")

Pre-splitting
-------------

After creating a partitioned and sharded collection, it is recommended to pre-split and move the chunks before inserting data or creating partitions.

Example:

To continue our example with a symbol id and timestamp, we'll create chunks for 100 distinct symbol ids and move them to each of 4 shards, while they're empty:

.. code-block:: javascript

  mongos> for (var i = 0; i <= 100; ++i) {
  ...         sh.splitAt("test.mycoll", {sid: i})
  ...     }
  { "ok" : 1 }
  mongos> for (var i = 0; i <= 100; ++i) {
  ...         if (i % 4 != 0) {
  ...             sh.moveChunk("test.mycoll", {sid: i},
  ...                          "shard000" + (i % 4))
  ...         }
  ...     }
  { "millis" : 48, "ok" : 1 }

Managing Partitions
-------------------

Adding and dropping partitions is done with the :command:`addPartition` and :command:`dropPartition` commands, just as described in :ref:`adding_a_partition` and :ref:`dropping_a_partition`. The :program:`mongos` router broadcasts these commands to all shards.

.. warning:: 
  Because shards' collection data are different, using :command:`addPartition` without specifying a custom ``newMax`` value will generate different partitions on each shard.

Since partitions may be different on each shard, dropping by partition ``_id`` is not supported. Instead, you must drop all partitions less than or equal to a given key, using either ``db.dropPartitionsLEQ()`` or the ``newMax`` field of the :command:`dropPartition` command.

Example:

.. code-block:: javascript

  mongos> db.mycoll.getPartitionInfo()
  {
      "raw" : {
          "test-sh01.tokutek.com:27018" : {
              "numPartitions" : NumberLong(3),
              "partitions" : [
                  {
                      "_id" : NumberLong(1),
                      "max" : {
                          "ts" : ISODate("2014-12-01T05:00:00Z"),
                          "_id" : ObjectId("541b374337c1928e3e7e5b2b")
                      },
                      "createTime" : ISODate("2014-09-18T19:45:32.482Z")
                  },
                  {
                      "_id" : NumberLong(2),
                      "max" : {
                          "ts" : ISODate("2014-12-21T05:00:00Z"),
                          "_id" : null
                      },
                      "createTime" : ISODate("2014-09-18T19:51:26.982Z")
                  },
                  {
                      "_id" : NumberLong(3),
                      "max" : {
                          "ts" : { "$maxKey" : 1 },
                          "_id" : { "$maxKey" : 1 }
                      },
                      "createTime" : ISODate("2014-09-18T19:54:59.247Z")
                  }
              ],
              "ok" : 1
          },
          "test-sh02.tokutek.com:27018" : {
              "numPartitions" : NumberLong(3),
              "partitions" : [
                  {
                      "_id" : NumberLong(1),
                      "max" : {
                          "ts" : ISODate("2014-11-01T04:00:00Z"),
                          "_id" : ObjectId("541b373d37c1928e3e7e5b2a")
                      },
                      "createTime" : ISODate("2014-09-18T19:45:32.482Z")
                  },
                  {
                      "_id" : NumberLong(2),
                      "max" : {
                          "ts" : ISODate("2014-12-21T05:00:00Z"),
                          "_id" : null
                     },
                      "createTime" : ISODate("2014-09-18T19:51:26.982Z")
                  },
                  {
                      "_id" : NumberLong(3),
                      "max" : {
                          "ts" : { "$maxKey" : 1 },
                          "_id" : { "$maxKey" : 1 }
                      },
                      "createTime" : ISODate("2014-09-18T19:54:59.247Z")
                  }
              ],
              "ok" : 1
          }
      },
      "ok" : 1
  }
  mongos> db.mycoll.dropPartitionsLEQ({ts: ISODate("2014-12-01T06:00:00Z")})
  {
      "raw" : {
          "lex2.tokutek.com:11111" : {
              "ok" : 1
          },
          "lex2.tokutek.com:22222" : {
              "ok" : 1
          }
      },
      "ok" : 1
  }

.. _draining_shards:

Draining Shards
===============

When draining a shard, the balancer is responsible for moving chunks off the draining shard. If, as recommended :ref:`above <creating_partition>`, the balancer is disabled for a sharded and partitioned collection, it will not move chunks of that collection even while draining a shard.

Therefore, when draining a shard, it is necessary to either re-enable the balancer (sh.enableBalancing('test.mycoll')), or move chunks of this collection manually.

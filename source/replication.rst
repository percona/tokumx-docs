.. _replication:

===========
Replication
===========

|TokuMX| implements the same replication concepts as MongoDB's replica sets. There are a few differences in implementation as described below, but most behavior is the same, including set membership, configuration and failover, write concern and read preference settings for clients, and most administrative commands.

.. note::
 TokuMX does not support the deprecated "master/slave" type of replication.

TokuMX's replication differs in several ways from basic |MongoDB|:

 * :ref:`atomicity`
 * :ref:`gtid_optime`
 * :ref:`write_concern`
 * :ref:`oplog_management`
 * :ref:`rollback`

In what follows, it will be important to recall that MongoDB's replication is done via what amounts to row-based replication via a system collection called the "oplog" (``local.oplog.rs``). Replication is asynchronous, though clients can wait for replication of each operation to complete using the write concern mechanism. Secondaries pull changes to the oplog into their own oplog (here, acting as a relay log), and then apply those changes to their own collections.

.. _atomicity:

Atomicity
=========

Background
----------
Basic |MongoDB| supports single-document atomicity in its data storage layer and also in its replication layer. A single client operation (such as an ``update`` with ``{multi: true}``) may be broken up into individual updates for each affected document, and the effects of the updates to each of these documents are all represented individually in the oplog, so they can be applied on an individual basis on the secondaries.

|TokuMX| aims to strengthen the transactional guarantees of |MongoDB|, and in this case to extend its :ref:`multi-document_atomicity` to replication.

.. _multi-document_entries:

Multi-document Entries
----------------------
Within a single |TokuMX| replica set, operations are :ref:`atomic <multi-document_atomicity>`. Just as this atomicity property applies on a single primary (for other clients connected to that primary), we want to provide this property on all machines in the replica set. Therefore, each operation must be applied atomically on each secondary in the set.

To achieve this, operations in the |TokuMX| oplog are arrays of individual document-level operations.

Example:
For example, consider an update operation that affects three documents:

.. code-block:: javascript

  > db.foo.insert([{_id: 1, x: 1}, {_id: 2, x: 10}, {_id: 3, x: 100}])
  > db.foo.update({_id: {$gte: 1, $lte: 3}}, {$inc: {x: 5}}, false, true)

In basic |MongoDB|, this update operation will be represented as three individual operations in the oplog, one for each affected document. However, in |TokuMX|, the operation in the oplog will contain an array of entries:

.. code-block:: javascript

  > use local
  switched to db local
  > db.oplog.rs.find().sort({_id: -1}).limit(1).pretty()
  {
      "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAAAw=="),
      "ts" : ISODate("2014-06-27T20:46:50.397Z"),
      "h" : NumberLong("7738476390427196552"),
      "a" : true,
      "ops" : [
          {
              "op" : "ur",
              "ns" : "test.foo",
              "pk" : {
                  "" : 1
              },
              "o" : {
                  "_id" : 1,
                  "x" : 1
              },
              "m" : {
                  "$inc" : {
                      "x" : 5
                  }
              }
          },
          {
              "op" : "ur",
              "ns" : "test.foo",
              "pk" : {
                  "" : 2
              },
              "o" : {
                  "_id" : 2,
                  "x" : 10
              },
              "m" : {
                  "$inc" : {
                      "x" : 5
                  }
              }
          },
          {
              "op" : "ur",
              "ns" : "test.foo",
              "pk" : {
                  "" : 3
              },
              "o" : {
                  "_id" : 3,
                  "x" : 100
              },
              "m" : {
                  "$inc" : {
                      "x" : 5
                  }
              }
          }
      ]
  }

Ignoring for now the way each entry differs from the update entries in basic |MongoDB|, the important feature here is that this single update operation creates a single document in the oplog, containing an array of subdocuments describing all the updates done by this operation.

When this document is encountered in the oplog on secondaries, all of its modifications will be applied atomically and will be made durable and also visible to other clients together.

Large Transactions
^^^^^^^^^^^^^^^^^^

Sometimes, a transaction will affect a large number of changes. In this case, it may be that all of the changes together in one oplog entry are larger than the 16MB BSON element size limit.

In this case, the operations are broken into multiple entries in a separate collection (``local.oplog.refs``), and on commit, an entry referencing all of them is written to the main oplog (``local.oplog.rs``).

Example:
Consider an update that modifies every document in the collection:

.. code-block:: javascript

 > db.foo.update({}, {$inc: {x: 1}}, false, true)

This will write a single entry to ``local.oplog.rs``:

.. code-block:: javascript

  > db.oplog.rs.find().sort({_id: -1}).limit(10).pretty()
  {
      "_id" : BinData(0,"AAAAAAAAAAoAAAAAAAAACg=="),
      "ts" : ISODate("2014-07-03T21:07:14.603Z"),
      "h" : NumberLong("1793255100492775595"),
      "a" : true,
      "ref" : ObjectId("53b5c6028e1112522e40f670")
  }

Instead of having an array of operations, this object has a reference to the ``local.oplog.refs`` collection, named ref. We can find some of the operations referenced by this entry:

.. code-block:: javascript

  > db.oplog.refs.find({'_id.oid': ObjectId("53b5c6028e1112522e40f670")}).limit(2)
  {
          "_id" : {
                  "oid" : ObjectId("53b5c6028e1112522e40f670"),
                  "seq" : NumberLong(3)
          },
          "ops" : [
                  {
                          "op" : "ur",
                          "ns" : "example.foo",
                          "pk" : {
                                  "" : ObjectId("53b5c5adad163158bce0e2f8")
                          },
                          "o" : {
                                  "_id" : ObjectId("53b5c5adad163158bce0e2f8"),
                                  "x" : 0,
                                  "str" : "aaa...a"
                          },
                          "m" : {
                                  "$inc" : {
                                          "x" : 1
                                  }
                          }
                  }
          ]
  }
  {
          "_id" : {
                  "oid" : ObjectId("53b5c6028e1112522e40f670"),
                  "seq" : NumberLong(5)
          },
          "ops" : [
                  {
                          "op" : "ur",
                          "ns" : "example.foo",
                          "pk" : {
                                  "" : ObjectId("53b5c5adad163158bce0e2f9")
                          },
                          "o" : {
                                  "_id" : ObjectId("53b5c5adad163158bce0e2f9"),
                                  "x" : 1,
                                  "str" : "aaa...a"
                          },
                          "m" : {
                                  "$inc" : {
                                          "x" : 1
                                  }
                          }
                  }
          ]
  }

In this case, the documents were artificially inflated with the ``str`` to force this behavior, but generally each entry in the ``local.oplog.refs`` collection can have its own array of operations, batching the client operation's work into chunks.

.. _gtid_optime:

GTID vs. OpTime
===============

Background
==========

In basic |MongoDB|, the DBWrite lock serializes all access to the oplog, which defines a strict order on write operations as they appear in the oplog, and thus on the order in which they are applied on secondaries. This ordering is what allows failover elections and rollback to function properly, but the serialization limits write concurrency, which is a fundamental goal for |TokuMX|.

Also in basic |MongoDB|, the unique identifier for each entry in the oplog is the OpTime, which is a 32-bit timestamp (with second resolution) and a 32-bit counter that gets reset to 1 each time the timestamp increases. It is well documented that due to clock skew, using wall-clock timestamps (as opposed to logical timestamps) can cause inconsistency and data loss in systems that use them to sequence operations, and MongoDB's replication is no exception.

In |TokuMX|, we no longer use the OpTime to identify oplog entries. We aim to solve both the concurrency and some of the data loss problems by using a different mechanism to identify entries, named the GTID, or "global transaction identifier."

Definition
==========

The GTID in TokuMX is a pair of 64-bit integers. The most significant integer is a sequence number that is incremented each time a new primary is elected. The least significant integer is a sequence number that is incremented each time an operation commits on the primary.

The GTID is stored as a 128-bit ``BinData`` value that can be interpreted directly as described above (two concatenated 64-bit integers).

The :program:`mongo` shell that ships with |TokuMX| has added functionality to manipulate GTIDs. Consider an oplog entry:

.. code-block:: javascript

  {
      "_id" : BinData(0,"AAAAAAAAAAEAAAAAAAAAAw=="),
      "ts" : ISODate("2014-06-27T20:46:50.397Z"),
      "h" : NumberLong("7738476390427196552"),
      "a" : true,
      "ops" : [
          {
              "op" : "ur",
              "ns" : "test.foo",
              "pk" : {
                  "" : 1
              },
              "o" : {
                  "_id" : 1,
                  "x" : 1
              },
              "m" : {
                  "$inc" : {
                      "x" : 5
                  }
              }
          }
      ]
  }

The shell function ``printGTID()`` can display this GTID in a readable form, and ``GTID()`` constructs a new GTID that can be used in queries:

Example:

.. code-block:: javascript

  > BinData(0,"AAAAAAAAAAEAAAAAAAAAAw==").printGTID()
  GTID(1, 3)
  > GTID(1, 4)
  BinData(0,"AAAAAAAAAAEAAAAAAAAABA==")
  > db.oplog.rs.find({_id: GTID(1, 4)})

Additionally, the methods ``GTIDPri()`` and ``GTIDSec()`` access the first and second number of a GTID:

Example:

.. code-block:: javascript

  > GTID(1,3).GTIDPri()
  1
  > BinData(0,"AAAAAAAAAAEAAAAAAAAAAw==").GTIDSec()
  3

Concurrency
-----------
One advantage of the GTID is improved write concurrency. Every write operation in |TokuMX| constitutes an :ref:`atomic transaction <multi-document_atomicity>`. Since these transactions are atomic, each one may constitute a large number of changes, but should either appear fully or not at all in the oplog. Operations' atomicity in replication is described in :ref:`atomicity`.

For these operations, the GTID plays an important role in write concurrency. Since operations are atomic, they can be large, and committing them to the oplog can be time consuming. If we required that operations were committed to the oplog sequentially, this would limit write concurrency for some workloads.

Instead of making operations serialize with each other when committing and writing to the oplog, TokuMX allows concurrent operations to write to the oplog concurrently, after allocating themselves a new GTID at commit time. Therefore, the operations may finish writing to the oplog out of GTID-order, but they can do their work concurrently.

Rather than ensure that operations write to the oplog in a strict order, we allow operations to commit concurrently, but push this ordering restriction to the secondaries. Secondaries are not allowed to read operations higher in GTID-order any operation that is not finished committing. For this purpose, the primary has a "GTID manager" that knows the GTID of the oldest operation that has not finished committing, and secondaries do not read this operation or any with a higher GTID.

This way, we ensure that every secondary has replicated an unbroken prefix of the primary's oplog, which makes elections and rollback possible, while allowing for concurrent operations to commit without blocking each other on the primary.

Safety
------
The GTID mechanism in |TokuMX| also provides additional safety in the face of network delays or other forms of clock skew, which are common in large clusters.

In basic |MongoDB|, if a machine's clock is sufficiently far ahead or behind (or messages get delayed sufficiently over the network), it can cause operations that were committed even with majority write concern to be rolled back. It can also cause operations that should not be represented in the cluster because they should have been rolled back to be present only on certain machines but not others.

TokuMX's replication application is not affected by clock skew, because system clocks are simply not used to identify transactions. The facts that that only one primary exists at a time, secondaries cannot accept writes, and the most-significant part of the GTID is incremented on every primary transition ensure that operations' commit order is identical to GTID order. That is, clock skew cannot cause a later operation to appear to come before an earlier operation in the oplog.

.. _write_concern:

Write Concern
=============

In |MongoDB|, when a write has its Write Concern satisfied (for ``w`` greater than 1), this means the write has been replicated to some secondaries, and all subsequent queries on those servers will show the result of the write.

In |TokuMX|, this is relaxed slightly. Secondaries report when they have replicated the oplog entry for a write, but don't wait until the effects have been applied to collections. So there is a small period of time after a ``getLastError`` returns, reporting that replication has completed to ``w`` machines, but a query immediately after this might not see the results of the write.

However, if the replica set fails over to such a secondary, that machine will guarantee the presence of the write after it is promoted to primary. In particular, beginning with the change to `Ark <http://www.tokutek.com/tag/explaining_ark/>`_ elections in |TokuMX| 2.0, a write which has been acknowledged with "majority" Write Concern is guaranteed to persist across all failovers.

In short, Write Concern in |TokuMX| guarantees safety of the write, but not visibility.

.. _oplog_management:

Oplog Management
================

In basic |MongoDB|, the oplog is a capped collection. Starting in |TokuMX| 1.4.0 the ``oplog`` and ``oplog.refs`` collections are :ref:`partitioned_collections`.

Periodically, the system creates a new partition and drops the oldest partition. The :variable:`expireOplogDays/expireOplogHours` parameter controls how much oplog data is retained. As a result, the command line parameter ``--oplogSize`` is deprecated.

If the time represented by these settings is less than a day, then a partition is added hourly. Otherwise, a partition is added daily.

.. note:: 
  On upgrade, existing oplogs will be converted to partitioned collections with the old data all in one partition and an empty partition after it for new data. After :variable:`expireOplogDays`, this large partition will be dropped.

  If, at the time of upgrade from |TokuMX| 1.3, the oplog is full, then as this large partition will not be dropped for another period of :variable:`expireOplogDays`, the oplog will effectively grow to twice as large as intended before it gets trimmed. In this case it is a good idea to manually trim the first partition sooner, before space becomes a problem but after the other partitions have accumulated a safe amount of data.

Information about the oplog is available in shell commands ``rs.oplogPartitionInfo()`` and ``rs.oplogRefsPartitionInfo()``. These provide the same information as the :command:`getPartitionInfo` for the ``oplog.rs`` and ``oplog.refs`` collections, plus some replication-specific information.

Example:

.. code-block:: javascript

  rs0:PRIMARY> rs.oplogPartitionInfo()
  {
     "numPartitions" : NumberLong(2),
     "partitions" : [
        {
           "id" : NumberLong(0),
           "max" : {
              "" : BinData(0,"AAAAAAAAAAEAAAAAAAAAAA==")
           },
           "createTime" : ISODate("2014-02-03T21:04:00.983Z")
        },
        {
           "id" : NumberLong(1),
           "max" : {
              "" : { "$maxKey" : 1 }
           },
           "createTime" : ISODate("2014-02-03T21:04:47.494Z")
        }
     ],
     "ok" : 1
  }
  rs0:PRIMARY> rs.oplogRefsPartitionInfo()
  {
     "numPartitions" : NumberLong(2),
     "partitions" : [
        {
           "id" : NumberLong(0),
           "max" : {
              "" : {
                 "oid" : ObjectId("52f004416ae5e77a3cbbbc4c"),
                 "seq" : NumberLong(3)
              }
           },
           "createTime" : ISODate("2014-02-03T21:04:01.109Z"),
           "maxRefGTID" : BinData(0,"AAAAAAAAAAEAAAAAAAAAAA==")
        },
        {
           "_id" : NumberLong(1),
           "max" : {
              "" : { "$maxKey" : 1 }
           },
           "createTime" : ISODate("2014-02-03T21:04:47.543Z")
        }
     ],
     "ok" : 1
  }

Adding and dropping oplog partitions must be done by the server, to ensure consistency between the ``oplog.rs`` and ``oplog.refs`` collections. To manually add and drop partitions of the oplog, there are new shell commands ``rs.addPartition()``, ``rs.trimToGTID()``, and ``rs.trimToTS()``.

.. note:: 
  Data must exist in the last partition in order to add a partition. Running ``rs.addPartition()`` when the last partition is empty will fail.

Adding a partition:

.. code-block:: javascript

  rs0:PRIMARY> rs.addPartition()

To drop partitions, you must use either ``rs.trimToGTID()`` or ``rs.trimToTS()``, using the GTID or timestamp from ``rs.oplogPartitionInfo()``.

Dropping a partition:
Using the example above for ``rs.oplogPartitionInfo()``, either of these commands will trim the oldest partition away:

.. code-block:: javascript
  
  rs0:PRIMARY> rs.trimToGTID(BinData(0,"AAAAAAAAAAEAAAAAAAAAAA=="))
  rs0:PRIMARY> rs.trimToTS(ISODate("2014-02-03T21:04:47.494Z"))
  
.. note:: 
  The GTID referenced is ``maxRefGTID`` of the first partition in ``rs.oplogRefsPartitionInfo()`` and max of the first partition of ``rs.oplogPartitionInfo()``. Similarly, the time passed to ``rs.trimToTS()`` is the ``createTime`` of the second partition referenced in ``rs.oplogPartitionInfo()``.

.. command:: db.getReplicationInfo()

The :command:`db.getReplicationInfo()` method gives you information about the state of the oplog on either a primary or secondary machine in a replica set. In |TokuMX|, it gives you slightly different information than in basic |MongoDB|. Mostly, this is because in |TokuMX|, the oplog isn't a capped collection with a fixed size on disk. Here are the differences:

``logSizeMB`` is no longer a single number. Instead, it is an object with two fields, ``uncompressed`` and ``compressed``, which show how much user data is in the oplog, and how much space that takes up on disk. Additionally, there are subdocuments ``oplog.rs`` and ``oplog.refs`` that show this same information, but broken down into sizes for the two collections that together store the oplog information.

Example:

.. code-block:: javascript

  test:PRIMARY> db.getReplicationInfo()
  {
     "logSizeMB" : {
        "uncompressed" : 556.4206256866455,
          "compressed" : 224,
          "oplog.rs" : {
             "uncompressed" : 356.61303424835205,
             "compressed" : 112
        },
        "oplog.refs" : {
           "uncompressed" : 199.80759143829346,
           "compressed" : 112
        }
     },
     "timeDiff" : 4802.052,
     "timeDiffHours" : 1.33,
     "tFirst" : "Thu Jan 23 2014 16:14:35 GMT-0500 (EST)",
     "tLast" : "Thu Jan 23 2014 17:34:37 GMT-0500 (EST)",
     "now" : "Thu Jan 23 2014 17:40:15 GMT-0500 (EST)"
  }

.. command:: db.printReplicationInfo()

The :command:`db.printReplicationInfo()` method prints some of the information from ``db.getReplicationInfo()`` with an explanation.

Example:

.. code-block:: javascript

  test:PRIMARY> db.printReplicationInfo()
  oplog user data size: 556.42MB
  oplog on-disk size: 224.00MB
  log length start to end: 4802.052secs (1.33hrs)
  oplog first event time: Thu Jan 23 2014 16:14:35 GMT-0500 (EST)
  oplog last event time: Thu Jan 23 2014 17:34:37 GMT-0500 (EST)
  now: Thu Jan 23 2014 17:35:12 GMT-0500 (EST)

.. command::  db.printSlaveReplicationInfo()

The :command:`db.printSlaveReplicationInfo()` method shows the position of each of the secondaries in a replica set. In |TokuMX|, it also prints the lag of each secondary behind the primary, in seconds.

Example:

.. code-block:: javascript

  test:PRIMARY> db.printSlaveReplicationInfo() source: localhost:27018
   syncedTo: Thu Jan 23 2014 17:34:37 GMT-0500 (EST)
       = 70 secs ago (0.02hrs), 0 secs behind primary

.. _rollback:

Rollback
========

Because TokuMX's implementation of replication is different than MongoDB's, the algorithms for rollback during replica set failover has changed. The purpose of rollback is still the same, to `revert write operations on a former primary when the member rejoins a replica set after a failover <http://docs.mongodb.org/manual/core/replica-set-rollbacks/#rollbacks-during-replica-set-failover>`_.

Collect Rollback Data
---------------------
Supported since 2.0.0

When a rollback occurs, |MongoDB| writes data to BSON files in the rollback/ folder. |TokuMX|, starting in version 2.0, instead writes the oplog entries that are rolled back to a collection, ``local.rollback.opdata``. Each entry in ``local.rollback.opdata`` stores information about an operation that was rolled back.

``local.rollback.opdata`` entry fields:

``_id.rid``

This value identifies all entries associated with a single rollback instance. In the example below, they all have value of 0. If another rollback occurs on this member, entries associated with it will have a different, higher ``_id.rid``. Therefore, all operations undone by a single rollback can be identified with a common ``_id.rid`` field.

``_id.seq``

A sequence number to distinguish different entries associated with a single rollback.

``gtid``

The :ref:`GTID <gtid_optime>` of the transaction containing this oplog entry. In the example below, note that the update was part of its own transaction, whereas the other three entries, all inserts, were part of another transaction.

``gtidString``

The GTID printed in string form to facilitate user inspection of the GTID. In the example below, the first entry, an update, is the only operation done with transaction ``GTID(1,15)``, while the three inserts were done within one transaction, ``GTID(1,14)``.

``time``

The timestamp of when the rollback algorithm processed that entry. This gives the user an idea of when the rollback occurred. All entries associated with a single rollback should have timestamps from around the same period.

``op``

The operation rolled back, as it would appear in an `oplog entry <multi-document_entries>`.

Example:

.. code-block:: javascript

  rs0:SECONDARY> db.rollback.opdata.find().pretty()
  {
      "_id" : {
          "rid" : 0,
          "seq" : 0
      },
      "gtid" : BinData(0,"AAAAAAAAAAEAAAAAAAAADw=="),
      "gtidString" : "GTID(1, 15)",
      "time" : ISODate("2014-09-10T15:16:36.619Z"),
      "op" : {
          "op" : "ur",
          "ns" : "test.foo",
          "pk" : {
              "" : 5
          },
          "m" : {
              "$inc" : {
                  "a" : 1
              }
          },
          "q" : {

          },
          "f" : 3
      }
  }
  {
      "_id" : {
          "rid" : 0,
          "seq" : 1
      },
      "gtid" : BinData(0,"AAAAAAAAAAEAAAAAAAAADg=="),
      "gtidString" : "GTID(1, 14)",
      "time" : ISODate("2014-09-10T15:16:36.619Z"),
      "op" : {
          "op" : "i",
          "ns" : "test.foo",
          "o" : {
              "_id" : 13,
              "a" : 130
          }
      }
  }
  {
      "_id" : {
          "rid" : 0,
          "seq" : 2
      },
      "gtid" : BinData(0,"AAAAAAAAAAEAAAAAAAAADg=="),
      "gtidString" : "GTID(1, 14)",
      "time" : ISODate("2014-09-10T15:16:36.619Z"),
      "op" : {
          "op" : "i",
          "ns" : "test.foo",
          "o" : {
              "_id" : 12,
              "a" : 120
          }
      }
  }
  {
      "_id" : {
          "rid" : 0,
          "seq" : 3
      },
      "gtid" : BinData(0,"AAAAAAAAAAEAAAAAAAAADg=="),
      "gtidString" : "GTID(1, 14)",
      "time" : ISODate("2014-09-10T15:16:36.619Z"),
      "op" : {
          "op" : "i",
          "ns" : "test.foo",
          "o" : {
              "_id" : 11,
              "a" : 110
          }
      }
  }

To discard the data saved in ``local.rollback.opdata``, the user may drop the collection. Future rollbacks will recreate the collection.

Limitations
-----------

In some cases, rollback may fail, causing the replica set to go to the FATAL state. In this case, you need to replace and/or recreate the secondary.

* If the amount of data to rollback goes back at least 30 minutes, rollback will fail. That is, the point being rolled back to was processed at least 30 minutes before the point the member is at now. Note that |TokuMX| does not have the 300MB limit |MongoDB| does. This 30 minute limit was inherited from |MongoDB|.

* Rollback of DDL operations (rename, create/drop collection/database, ``ensureIndex``, ``dropIndexes``) does not work. If a secondary encounters a DDL operation during rollback, rollback will fail.

* A component of the rollback algorithm is to play the oplog forward until the member running the rollback catches up to a certain point. This point is determined by the oplog position of the remote member being synced from. During this phase, if a collection that participated in the rollback is renamed, or is dropped and recreated, then rollback will fail.

.. _miscellaneous_differences:

Miscellaneous Differences
=========================

There are a few additional differences from basic |MongoDB| with respect to replication:

* The format of rows found in the oplog is no longer the same. Any application that depends on interpreting rows in the oplog will have to change. Please contact us with questions if this is the case. For an overview of the new oplog format, see :ref:`multi-document_entries`.

* Tailable cursors on secondaries may not work correctly. They may miss data in the capped collection, because data may not be applied in order on secondaries (operations are applied in commit order, rather than insert order). Use tailable cursors on capped collections only on primaries.

.. note:: 
  This issue does not exist for the oplog. As mentioned above, the oplog is not a capped collection.


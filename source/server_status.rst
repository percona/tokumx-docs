.. _server_status:

=============
Server Status
=============

In |TokuMX|, there are many changes to the information provided by `db.serverStatus() <http://docs.mongodb.org/manual/reference/server-status/>`_ to help users monitor running systems and diagnose issues.

New Sections
============

All of the new information in `db.serverStatus() <http://docs.mongodb.org/manual/reference/server-status/>`_ for |TokuMX| is in a new section named ft.

Unless otherwise noted, all counter-type metrics are reset to zero on server startup.

Example:

To see all of the new information below, use:

.. code-block:: javascript
 
  > db.serverStatus().ft

Each new subsection's information is described below.

Fsyncs
------

The |TokuMX| transaction log normally receives many small sequential writes, and periodic fsyncs. |TokuMX| tracks the number of and time spent doing fsyncs (across all files, though the transaction log is by far the most frequently synced file).

Example:

.. code-block:: javascript

  > db.serverStatus().ft.fsync
  {
      "count" : 772,
      "time" : 68707786
  }

.. variable:: fsync.count

The total number of times the database has flushed the operating system's file buffers to disk.

.. variable:: fsync.time

The total time, in microseconds, used to fsync to disk.

Transaction Log
---------------

The |TokuMX| transaction log normally receives many small sequential writes, and periodic fsyncs. |TokuMX| tracks the bytes written and number of writes to the log, as well as the time spent doing those writes.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.log
  {
      "count" : 2496,
      "time" : 0.17602924388888888,
      "bytes" : 254716357
  }

.. variable:: log.count

The number of individual log writes since the last server startup.

.. variable:: log.time

The total time, in seconds, used to perform log writes since the last server startup.

.. variable:: log.bytes

The number of bytes the logger has written to disk since the last server startup.

.. _cachetable:

Cachetable
----------

Unlike basic |MongoDB|, |TokuMX| manages the memory it uses directly. You should not rely on the ``mem`` section, instead, the :ref:`cachetable` section describes TokuMX's memory usage in detail.

In the following, it is important to note the distinction between "full" and "partial" cachetable operations. Index tree nodes in |TokuMX| have two parameters: :option:`pageSize` and :option:`readPageSize`. All writes are done in :option:`pageSize` chunks, but reads from disk can be done in :option:`pageSize` chunks, which are smaller (individual segments of the whole node). "Full" evictions and misses correspond to evicting or paging in the entire node, while "partial" evictions and misses correspond to evicting or paging in just one of the node's segments, which are generally preferred as they are less expensive.

|TokuMX| tracks the work done to evict old data from the cachetable when it reaches its memory limit (:variable:`cacheSize`), as "evictions". Clean evictions are usually cheap, but dirty evictions are much more expensive, as they require that data be written out before it can be released. If a page is clean, it can be partially evicted, which evicts only the lesser recently used parts of the page and does not cause a future full miss later on. Furthermore, |TokuMX| makes a distinction between leaves and nonleaves (internal tree nodes) when presenting eviction data. If too many nonleaf nodes are getting evicted, this is a sign that the workload can benefit from more memory.

The ``cachetable.evictions`` subdocument breaks down the evictions into these categories: partial vs. full misses, and nonleaf vs. leaf nodes. TokuMX tracks both the number of evictions of each type, and the amount of space reclaimed by those evictions. In addition, full evictions also track the time taken to serialize, compress, and write those nodes to disk.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.cachetable
  {
      "size" : {
          "current" : 1516207470,
          "writing" : 0,
          "limit" : 1771674009
      },
      "miss" : {
          "count" : 65470,
          "time" : 8.309003743333333,
          "full" : {
              "count" : 1018,
              "time" : 0.0017150680555555555
          },
          "partial" : {
              "count" : 64452,
              "time" : 8.307288675277778
          }
      },
      "evictions" : {
          "partial" : {
              "nonleaf" : {
                  "clean" : {
                      "count" : 1280,
                      "bytes" : 334317512
                  }
              },
              "leaf" : {
                  "clean" : {
                      "count" : 76871,
                      "bytes" : 6956659264
                  }
              }
          },
          "full" : {
              "nonleaf" : {
                  "clean" : {
                      "count" : 6,
                      "bytes" : 15362782
                  },
                  "dirty" : {
                      "count" : 0,
                      "bytes" : 0,
                      "time" : 0
                  }
              },
              "leaf" : {
                  "clean" : {
                      "count" : 452,
                      "bytes" : 387796936
                  },
                  "dirty" : {
                      "count" : 0,
                      "bytes" : 0,
                      "time" : 0
                  }
              }
          }
      }
  }

.. variable:: cachetable.size.current

Total size, in bytes, of how much of your uncompressed data is currently in the database's internal cache.

.. variable:: cachetable.size.writing

Total size, in bytes, of nodes that are currently queued up to be written to disk for eviction.

.. variable:: cachetable.size.limit

Total size, in bytes, of how much of your uncompressed data will fit in TokuMX’s internal cache (:option:`cacheSize`).

.. variable:: cachetable.miss.count

This is a count of how many times the application was unable to access your data in the internal cache (i.e., a cache miss). This metric is similar to MongoDB’s btree misses and page faults.

.. variable:: cachetable.miss.time
 
This is the total time, in microseconds, of how long the database has had to wait for a disk read to complete for a cache miss.

.. variable:: cachetable.miss.full.{count,time}

The "count" and "time" breakdown for "full misses", which are more expensive than partial misses.

.. variable:: cachetable.miss.partial.{count,time}

The "count" and "time" breakdown for "partial misses", which are less expensive than full misses.

.. variable:: cachetable.evictions.partial.{nonleaf,leaf}.clean.{count,bytes}

Number of partial evictions (of nonleaf and leaf nodes), and their total size in bytes.

.. variable:: cachetable.evictions.full.{nonleaf,leaf}.clean.{count,bytes}

Number of clean, full evictions (of nonleaf and leaf nodes), and their total size in bytes.

.. variable:: cachetable.evictions.full.{nonleaf,leaf}.dirty.{count,bytes,time}

Number of dirty, full evictions (of nonleaf and leaf nodes), and their total size in bytes, as well as the time spent serializing, compressing, and writing those nodes to disk.

.. _checkpoint:

Checkpoint
----------

TokuMX's main data structure, the Fractal Tree, usually writes most user data to disk during checkpoints, which are triggered once every 60 seconds by default (:option:`checkpointPeriod`, similar to ``--syncdelay`` or `storage.syncPeriodSecs <http://docs.mongodb.org/manual/reference/configuration-options/#storage.syncPeriodSecs>`_ in |MongoDB| 2.6.0).

During a checkpoint, all tree data that has changed is written durably to disk, and this allows the system to trim old data from the tail of the transaction log. The system reports timing information about checkpoints as well as the number of bytes written, in the :ref:`checkpoint` section of `db.serverStatus() <http://docs.mongodb.org/manual/reference/server-status/>`_.

A checkpoint is triggered ``60`` (:option:`checkpointPeriod`) after the previous checkpoint was triggered, or immediately after the last checkpoint if it took longer than ``60`` seconds. For example, if every checkpoint takes 6 seconds, there should be 54 seconds between checkpoints, and :variable:`checkpoint.time` should be about 10% of the total system uptime. Extremely long checkpoints can cause a system to back up over time; if checkpoints are taking too long it may mean that your system needs more I/O bandwidth for node writes and/or more CPU power for compression. Disk writes for checkpoint are tracked in ``checkpoint.write``.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.checkpoint
  {
      "count" : 13,
      "time" : 232,
      "lastBegin" : ISODate("2014-06-17T20:51:13Z"),
      "lastComplete" : {
          "begin" : ISODate("2014-06-17T20:51:13Z"),
          "end" : ISODate("2014-06-17T20:51:32Z"),
          "time" : 19
      },
      "begin" : {
          "time" : 1432
      },
      "write" : {
          "nonleaf" : {
              "count" : 682,
              "time" : 0.06575672722222221,
              "bytes" : {
                  "uncompressed" : 880481988,
                  "compressed" : 300794368
              }
          },
          "leaf" : {
              "count" : 942,
              "time" : 0.2565985083333333,
              "bytes" : {
                  "uncompressed" : 2014680388,
                  "compressed" : 857306624
              }
          }
      }
  }

.. variable:: checkpoint.count

Number of completed checkpoints.

.. variable:: checkpoint.time

Time (in seconds) spent doing checkpoints.

.. variable:: checkpoint.lastBegin

The begin timestamp of the most recently started (possibly in progress) checkpoint.

.. variable:: checkpoint.lastComplete.begin

The begin timestamp of the most recently completed checkpoint.

.. variable:: checkpoint.lastComplete.end

The end timestamp of the most recently completed checkpoint.

.. variable:: checkpoint.lastComplete.time

The time spent, in seconds, by the most recently completed checkpoint.

.. variable:: checkpoint.write.{nonleaf,leaf}.count

Number of nonleaf and leaf nodes written to disk during checkpoints.

.. variable:: checkpoint.write.{nonleaf,leaf}.time

Time spent, in seconds, writing nonleaf and leaf nodes to disk during checkpoints.

.. variable:: checkpoint.write.{nonleaf,leaf}.bytes.uncompressed

Total size of nonleaf and leaf nodes written to disk during checkpoints, before compression.

.. variable:: checkpoint.write.{nonleaf,leaf}.bytes.compressed

Total size of nonleaf and leaf nodes written to disk during checkpoints, after compression.

.. _serialize_time:

Serialize Time
--------------

For writes, the primary consumer of CPU time is usually compression. For in-memory queries it's usually tree searches, but for >RAM queries, decompression and deserialization can begin to impact performance. Serialization/deserialization and compression/decompression times are reported in :ref:`serialize_time`.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.serializeTime
  {
      "nonleaf" : {
          "serialize" : 1.650712856111111,
          "compress" : 67.41081527,
          "decompress" : 0.797666756111111,
          "deserialize" : 4.092646362777778
      },
      "leaf" : {
          "serialize" : 2.524351571111111,
          "compress" : 176.64170658944442,
          "decompress" : 8.835384991666666,
          "deserialize" : 1.52323294
      }
  }

.. variable:: serializeTime.{nonleaf,leaf}.serialize

Total time, in seconds, spent serializing nonleaf and leaf nodes before writing them to disk (for checkpoint or when evicted while dirty).

.. variable:: serializeTime.{nonleaf,leaf}.compress

Total time, in seconds, spent compressing nonleaf and leaf nodes before writing them to disk (for checkpoint or when evicted while dirty).

.. variable:: serializeTime.{nonleaf,leaf}.decompress

Total time, in seconds, spent decompressing nonleaf and leaf nodes and their partitions after reading them off disk.

.. variable:: serializeTime.{nonleaf,leaf}.deserialize

Total time, in seconds, spent deserializing nonleaf and leaf nodes and their partitions after reading them off disk.

.. _locktree:

Locktree
--------

|TokuMX| uses a locktree to implement :ref:`document-level_locks` for ``SERIALIZABLE`` transactions. The locktree's size is limited by :option:`locktreeMaxMemory`, and some statistics are reported in :ref:`locktree`.

Example:

.. code-block:: javascript

  db.serverStatus().ft.locktree
  {
      "size" : {
          "current" : 2660,
          "limit" : 161061273
      }
  }

.. variable:: locktree.size.current

Total size, in bytes, of memory the locktree is currently using.

.. variable:: locktree.size.limit

Maximum number of bytes that the locktree is allowed to use.

.. _compression_ratio:

Compression Ratio
-----------------

One of TokuMX's biggest features is compression. For a quick estimate of compression, |TokuMX| tracks all node writes that have been done while the server has been up, and reports the effective compression ratio for those writes in :ref:`compression_ratio`.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.compressionRatio
  {
      "leaf" : 9.303496342310673,
      "nonleaf" : 10.9380145184542696,
      "overall" : 9.5766049110996824
  }
  
.. variable:: compressionRatio.{leaf,nonleaf,overall}

For every node that is written out for checkpoint or eviction, |TokuMX| records the size of the node before and after compression. The reported values in :ref:`compression_ratio` are the ratios of those sizes, segregated into just leaves, just nonleaves, and the total ratio for all node types.

This may be different from the actual compression ratio, because it only tracks recently written nodes. However, it can be more accurate than what's provided in ``db.collection.stats()``, because that uses estimated values for the uncompressed size, whereas ``db.serverStatus().ft.compressionRatio`` uses exact values.

.. _fast_updates_status:

Fast Updates
------------

|TokuMX| tracks information about the possible usage of :ref:`fast_updates` so users can learn whether fast updates can benefit or are benefiting their application.

Example:

.. code-block:: javascript

  > db.serverStatus().metrics.fastUpdates
  {
      "errors" : NumberLong(0),
      "eligible" : {
          "primaryKey" : NumberLong(1000),
          "secondaryKey" : NumberLong(100)
      },
      "performed" : {
          "primaryKey" : NumberLong(0),
          "secondaryKey" : NumberLong(0),
          "slowOnSecondary" : NumberLong(0)
      }
  }

.. variable:: metrics.fastUpdates.errors

Number of fast updates performed that were later found to have an error and therefore resulted in a no-op. Note that a single update can be counted multiple times, so this number is not entirely accurate. An example that causes this number to increase is trying to increment a text field with a fast update.

.. variable:: metrics.fastUpdates.eligible.primaryKey

Number of updates performed with fast updates disabled where the query uniquely identified the primary key and the update was eligible to be fast. If fast updates were enabled, these updates would have occurred with no reads, resulting in possibly large performance gains. If this value represents a large portion of your updates, you may want to consider enabling fast updates for better performance.

.. variable:: metrics.fastUpdates.eligible.secondaryKey

Number of updates performed with fast updates disabled where a query was required to find the primary key, but the update was eligible to be fast without fetching the entire document. If fast updates were enabled, these updates may have occurred with fewer reads, possibly resulting in performance gains. If this value represents a large portion of your updates, you may want to consider enabling fast updates for better performance. Some applications will show an improvement in performance, but some applications might not. Therefore, you ought to first experiment with fast updates enabled to see how your application is affected.

.. variable:: metrics.fastUpdates.performed.primaryKey

Number of updates performed with fast updates enabled where the query uniquely identified the primary key. These updates occurred with no reads, resulting in possibly large performance gains.

.. variable:: metrics.fastUpdates.performed.secondaryKey

Number of updates performed with fast updates disabled where a query was required to find the primary key, but fetching the entire document was not required. These updates may have occurred with fewer reads, possibly resulting in performance gains.

.. variable:: metrics.fastUpdates.performed.slowOnSecondary

Number of updates performed on a secondary that required a query to find the primary key, before applying the update. Generally, this should not happen unless there is a mismatch in indexes between the primary and the secondary. If these updates are happening, the user ought to investigate why, because this may lead to secondary lag.

.. _alerts:

Alerts
------

|TokuMX| also tracks some anomalous events, which will appear in the :ref:`alerts` section if any such events are detected.

Example:

.. code-block:: javascript

  > db.serverStatus().ft.alerts
  {
      "panic code" : NumberLong(0),
      "panic string" : "",
      "filesystem status" : "OK",
      "locktreeRequestsPending" : 0,
      "checkpointFailures" : 0,
      "longWaitEvents" : {
          "logBufferWait" : 12,
          "fsync" : {
              "count" : 1,
              "time" : 1334887
          },
          "cachePressure" : {
              "count" : 0,
              "time" : 0
          },
          "checkpointBegin" : {
              "count" : 0,
              "time" : 0
          },
          "locktreeWait" : {
              "count" : 0,
              "time" : 0
          },
          "locktreeWaitEscalation" : {
              "count" : 0,
              "time" : 0
          }
      }
  }

.. variable:: alerts["panic code"] / alerts["panic string"]

Integer error code and error string if the engine is panicked. Usually indicates an impending crash.

.. variable:: alerts["filesystem status"]

Current status of the filesystem's free space, with respect to :variable:`fsRedzone`.

.. variable:: alerts.locktreeRequestsPending

The number of requests for :ref:`document-level_locks` in the locktree that are waiting for other requests to release their locks.

During normal operation of most workloads, there should be no requests pending. This is a good field to monitor, because if it is large, you may be doing large multi-document ``update`` or ``findAndModify`` operations, which tend to cause :ref:`document-level_lock_conflicts` and drastically reduce the potential concurrency of write operations.

.. variable:: alerts.checkpointFailures

Number of checkpoints that have failed for any reason.

.. variable:: alerts.longWaitEvents.logBufferWait

Number of times a writing client had to wait more than 100ms for access to the log buffer.

.. variable:: alerts.longWaitEvents.fsync.{count,time}

Same information as :variable:`fsync.count` and :variable:`fsync.time`, but only for fsync operations that took more than 1 second.

.. variable:: alerts.longWaitEvents.cachePressure.{count,time}

Number of times and the time spent (in microseconds) that a thread had to wait more than 1 second for evictions to create space in the cachetable for it to page in data it needed.

.. variable:: alerts.longWaitEvents.locktreeWait.{count,time}

Number of times and the time spent (in microseconds) that a thread had to wait more than 1 second to acquire a document-level lock in the locktree.

.. variable:: alerts.longWaitEvents.locktreeWaitEscalation.{count,time}

Number of times and the time spent (in microseconds) that a thread had to wait more than 1 second to acquire a document-level lock because the locktree was at the memory limit (:variable:`locktreeMaxMemory`) and needed to run escalation.

Deprecated Entries
==================

The following entries in `db.serverStatus() <http://docs.mongodb.org/manual/reference/server-status/>`_ do not make sense for TokuMX's storage engine and have been removed.

* ``backgroundFlushing``

* ``dur``

* ``indexCounters``

* ``recordStats``

* ``mem.mapped``

* ``mem.mappedWithJournal``

* ``record``

* ``repl.buffer.maxSizeBytes``

* ``preload``




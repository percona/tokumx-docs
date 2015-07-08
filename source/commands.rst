.. _commands:

========
Commands
========
.. _new_commands:

New Commands
============

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

========= ================= ============================================================================
Field     Type              Description
========= ================= ============================================================================
isolation string (optional) One of :ref:`mvcc` (default), :ref:`serializable`, or :ref:`readUncommitted`
========= ================= ============================================================================

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

Runs a Hot Backup. This copies the dbpath to destination online, and leaves the files with contents identical to what was committed to disk at the moment the backupStart command returns. 

.. note:: 
  For more information about how backup works, see :ref:`hot_backup`.

Returns an error if there is already another backup operation running.

The backup destination must be a directory that exists, and should not be a subdirectory of dbpath.

If a separate :variable:`logDir` is used from ``dbpath``, then destination will contain two directories, data (containing the contents of ``dbpath``) and log (containing the contents of :variable:`logDir`).

.. note::
 Since Hot Backup copies data recursively, if logDir is a subdirectory of dbpath, all data is copied directly in to destination.

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

Queries the Hot Backup system for the status of a running backup operation, if one is running.

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

The Hot Backup system uses I/O in two ways: for mirroring writes, and for copying files (see Concepts for more details). Mirrored writes must be completed immediately, but file copying can be slow.

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



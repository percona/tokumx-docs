.. _pitr_plugin:

======================
Point in Time Recovery
======================

Supported since 2.0.0

|TokuMX| Enterprise Edition includes Point in Time Recovery (PITR) for |TokuMX| clusters using replication. Point in Time Recovery uses a backup of a server and a running server with an oplog to sync the backup server to a specific point in time, or GTID in the oplog.

Overview
========
Point in Time Recovery is conceptually a simple process: it mimics normal replication syncing until it has applied all entries up to a specific GTID or timestamp.

As such, PITR can only move a server's state forward in time. Typical usage of PITR is to start a server from a backup before the desired point in time, then sync forward with PITR.

To reach a desired point in time with PITR, you need a backup at a point in time before the desired time, and servers with oplog data reaching back to that backup's point in time, continuously through the desired time. If there is a gap between the backup and the earliest oplog data, PITR will not work properly.

Loading the Plugin
==================
To use Enterprise Point in Time Recovery, ensure that the ``pitr_plugin`` plugin is loaded into the server. Before loading, :file:`libpitr_plugin.so` must be in the :variable:`pluginsDir`. The tarballs as :ref:`unpacked during installation <tarball_installation>` and the :ref:`deb and rpm packages <installation_from_packages>` set up these paths and settings correctly by default.

To load the PITR plugin into a running :program:`mongod` server, use the :variable:`loadPlugin` command.

Example:

.. code-block:: javascript

  > db.adminCommand({loadPlugin: 'pitr_plugin'})
  {
    "loaded" : {
        "filename" : "/opt/tokumx/lib64/plugins/libpitr_plugin.so",
        "fullpath" : "/opt/tokumx/lib64/plugins/libpitr_plugin.so",
        "name" : "pitr_plugin",
        "version" : "1.0.0",
        "checksum" : "547d645b109a3b66a97b5c8e0f72733f",
        "commands" : [
        "recoverToPoint"
        ]
     },
     "ok" : 1
   }
                                                                                
Autoloading the Plugin
======================
                
To automatically load the PITR plugin on server startup, use the :variable:`loadPlugin` server parameter:

On the command line:

.. code-block:: bash
                
  $ mongod --loadPlugin=pitr_plugin:547d645b109a3b66a97b5c8e0f72733f
                                                                                
In the config file:

.. code-block:: bash
                
  loadPlugin=pitr_plugin:547d645b109a3b66a97b5c8e0f72733f

.. note::
  The checksum will be different in different versions of |TokuMX|. You can discover the checksum for your plugin by loading it once with the :variable:`loadPlugin` command.

Basic Usage
===========

The Enterprise Point in Time Recovery plugin adds one commands to a running :program:`mongod`. For details, see the section on :ref:`pitr_commands`.

:variable:`recoverToPoint` runs Point in Time Recovery to the provided ``GTID`` or timestamp.

To run :variable:`recoverToPoint`, the server must be in `maintenance mode <http://docs.mongodb.org/manual/reference/command/replSetMaintenance/>`_. Among other things, this prevents normal replication oplog syncing, which allows Point in Time Recovery to do this instead.

To enter maintenance mode on startup, use the :variable:`rsMaintenance` server parameter.

After running Point in Time Recovery, remove the recovered member from the replica set by shutting it down and starting it up without the ``--replSet`` option.

.. warning::
  Leaving `maintenance mode <http://docs.mongodb.org/manual/reference/command/replSetMaintenance/>`_ will allow normal replication syncing to proceed, which will move the server forward past the point recovered to.

Example:

In this example, we have a three-member replica set, in which we have started the last member with :variable:`rsMaintenance` so that it initialized itself in maintenance mode. We use :variable:`recoverToPoint` to sync that member up to a specific ``GTID``, after which point we can shut it down, remove it from the set, and use it in that state.

.. code-block:: javascript

  rs0:RECOVERING> rs.status()
  {
    "set" : "rs0",
    "date" : ISODate("2014-09-26T20:58:59Z"),
    "myState" : 3,
    "members" : [
        {
            "_id" : 0,
            "name" : "rs0-db0.demo.tokutek.com:27017",
            "health" : 1,
            "state" : 1,
            "stateStr" : "PRIMARY",
            "uptime" : 11,
            "optimeDate" : ISODate("2014-09-26T20:58:34.564Z"),
            "lastGTID" : "GTID(1, 202)",
            "lastUnappliedGTID" : "GTID(1, 202)",
            "minLiveGTID" : "GTID(1, 203)",
            "minUnappliedGTID" : "GTID(1, 203)",
            "oplogVersion" : 4,
            "highestKnownPrimaryInReplSet" : 1,
            "lastHeartbeat" : ISODate("2014-09-26T20:58:58Z"),
            "lastHeartbeatRecv" : ISODate("2014-09-26T20:58:58Z"),
            "pingMs" : 0
        },
        {
            "_id" : 1,
            "name" : "rs0-db1.demo.tokutek.com:27017",
            "health" : 1,
            "state" : 2,
            "stateStr" : "SECONDARY",
            "uptime" : 11,
            "optimeDate" : ISODate("2014-09-26T20:58:34.564Z"),
            "lastGTID" : "GTID(1, 202)",
            "lastUnappliedGTID" : "GTID(1, 202)",
            "minLiveGTID" : "GTID(1, 203)",
            "minUnappliedGTID" : "GTID(1, 203)",
            "oplogVersion" : 4,
            "highestKnownPrimaryInReplSet" : 1,
            "lastHeartbeat" : ISODate("2014-09-26T20:58:58Z"),
            "lastHeartbeatRecv" : ISODate("2014-09-26T20:58:58Z"),
            "pingMs" : 0,
            "lastHeartbeatMessage" : "syncing to: rs0-db0.demo.tokutek.com:27017",
            "syncingTo" : "rs0-db0.demo.tokutek.com:27017"
        },
        {
            "_id" : 2,
            "name" : "rs0-db2.demo.tokutek.com:27017",
            "health" : 1,
            "state" : 3,
            "stateStr" : "RECOVERING",
            "uptime" : 11,
            "optimeDate" : ISODate("2014-09-26T20:57:26.989Z"),
            "lastGTID" : "GTID(1, 102)",
            "lastUnappliedGTID" : "GTID(1, 102)",
            "minLiveGTID" : "GTID(1, 103)",
            "minUnappliedGTID" : "GTID(1, 103)",
            "oplogVersion" : 4,
            "highestKnownPrimaryInReplSet" : 1,
            "maintenanceMode" : 1,
            "errmsg" : "initial sync done",
            "self" : true
        }
    ],
    "ok" : 1
  }
  rs0:RECOVERING> db.runCommand({loadPlugin: 'pitr_plugin'})
  {
      "loaded" : {
          "filename" : "/opt/tokumx/lib64/plugins/libpitr_plugin.so",
          "fullpath" : "/opt/tokumx/lib64/plugins/libpitr_plugin.so",
          "name" : "pitr_plugin",
          "version" : "1.0.0",
          "checksum" : "547d645b109a3b66a97b5c8e0f72733f",
          "commands" : [
                "recoverToPoint"
          ]
      },
      "ok" : 1
  }
  rs0:RECOVERING> db.runCommand({recoverToPoint: 1, gtid: GTID(1, 152)})
  { "ok" : 1 }
  rs0:RECOVERING> rs.status()
  {
      "set" : "rs0",
      "date" : ISODate("2014-09-26T20:59:37Z"),
      "myState" : 3,
      "members" : [
        {
              "_id" : 0,
              "name" : "rs0-db0.demo.tokutek.com:27017",
              "health" : 1,
              "state" : 1,
              "stateStr" : "PRIMARY",
              "uptime" : 49,
              "optimeDate" : ISODate("2014-09-26T20:58:34.564Z"),
              "lastGTID" : "GTID(1, 202)",
              "lastUnappliedGTID" : "GTID(1, 202)",
              "minLiveGTID" : "GTID(1, 203)",
              "minUnappliedGTID" : "GTID(1, 203)",
              "oplogVersion" : 4,
              "highestKnownPrimaryInReplSet" : 1,
              "lastHeartbeat" : ISODate("2014-09-26T20:59:36Z"),
              "lastHeartbeatRecv" : ISODate("2014-09-26T20:59:36Z"),
              "pingMs" : 0
        },
        {
              "_id" : 1,
              "name" : "rs0-db1.demo.tokutek.com:27017",
              "health" : 1,
              "state" : 2,
              "stateStr" : "SECONDARY",
              "uptime" : 49,
              "optimeDate" : ISODate("2014-09-26T20:58:34.564Z"),
              "lastGTID" : "GTID(1, 202)",
              "lastUnappliedGTID" : "GTID(1, 202)",
              "minLiveGTID" : "GTID(1, 203)",
              "minUnappliedGTID" : "GTID(1, 203)",
              "oplogVersion" : 4,
              "highestKnownPrimaryInReplSet" : 1,
              "lastHeartbeat" : ISODate("2014-09-26T20:59:36Z"),
              "lastHeartbeatRecv" : ISODate("2014-09-26T20:59:36Z"),
              "pingMs" : 0,
              "syncingTo" : "rs0-db0.demo.tokutek.com:27017"
        },
        {
              "_id" : 2,
              "name" : "rs0-db2.demo.tokutek.com:27017",
              "health" : 1,
              "state" : 3,
              "stateStr" : "RECOVERING",
              "uptime" : 49,
              "optimeDate" : ISODate("2014-09-26T20:58:09.637Z"),
              "lastGTID" : "GTID(1, 152)",
              "lastUnappliedGTID" : "GTID(1, 152)",
              "minLiveGTID" : "GTID(1, 153)",
              "minUnappliedGTID" : "GTID(1, 153)",
              "oplogVersion" : 4,
              "highestKnownPrimaryInReplSet" : 1,
              "maintenanceMode" : 1,
              "errmsg" : "syncing to: rs0-db0.tokutek.com:27017",
              "self" : true
        }
    ],
    "ok" : 1
  }

.. _sharding_and_backups:

Sharding and Backups
====================

Enterprise Point in Time Recovery can be used to help take an approximately consistent backup of a sharded cluster with less downtime than a perfectly consistent backup, if this is a desirable tradeoff.

See :ref:`using_pitr` for details on how to accomplish this.

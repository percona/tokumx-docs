.. _installation:

==============
 Installation
==============

|TokuMX| Community edition is available in :ref:`packages <installation_from_packages>` for :ref:`centos` and :ref:`debian_and_ubuntu`. 

For users of other distributions, Tokutek provides a binary version of |TokuMX|. The Tokutek distribution includes the |TokuMX| server (``mongod``), router (``mongos``), and client (``mongo``) for your system, as well as miscellaneous tools like those provided with |MongoDB|.

The installation of |TokuMX| from a tarball release file is similar to the installation of |MongoDB|. This process is described nicely in the `MongoDB Documentation <http://docs.mongodb.org/manual/tutorial/install-mongodb-on-linux/>`_.

System and Hardware Requirements
================================

 * Operating Systems: |TokuMX| is currently supported on 64-bit Linux only. |TokuMX| is tested on *CentOS* 5, 6 and 7, but is expected to work on other Linux distributions.

 * A virtual machine image is available for evaluations on Windows and Mac. There are also binaries available for recent versions of Mac OS X for development use only. Please contact us at support@tokutek.com for more information.

 * Memory: |TokuMX| requires at least 1GB of RAM but for best results, we recommend to run with at least 2GB of RAM.

 * Disk space and configuration: Please make sure to allocate enough disk space for data, indexes and logs. In our users' experience, |TokuMX| achieves up to 20x space savings on data and indexes over |MongoDB| due to high compression.

Downloading
===========

To download the binary tarball, visit `Percona's Download Page <https://www.percona.com/downloads/percona-tokumx/LATEST/>`_.

After downloading, optionally verify the MD5 checksum by comparing the output of :program:`md5sum` to the MD5 checksum from the download page:

.. code-block:: bash

  $ md5sum tokumx-2.0.0-linux-x86_64-main.tar.gz 
  d2f979f2c6bb0d47bf9d3fd615a4ad53  tokumx-2.0.0-linux-x86_64-main.tar.gz

.. _tarball_installation:

Installing the Tarball
======================
For maximum flexibility, we recommend unpacking the tarball to a unique location, and then creating symlinks to the binaries there. In this example, we'll unpack the tarball in :file:`/opt`.

1. Unpack the tarball

  Unpack the tarball to :file:`/opt`:

.. code-block:: bash

  $ tar xzf tokumx-2.0.0-linux-x86_64-main.tar.gz --directory /opt

2. Create symlinks

 Create symlinks to all the binaries in :file:`/usr/local/bin`:

.. code-block:: bash

   $ ln -snf /opt/tokumx-2.0.0-linux-x86_64/bin/* /usr/local/bin

3. Check your PATH

 Ensure that your PATH includes :file:`/usr/local/bin` to make sure you can run |TokuMX|. Make sure your shell finds the right binaries.

.. code-block:: bash

  $ which mongod
  /usr/local/bin/mongod
  $ readlink /usr/local/bin/mongod
  /opt/tokumx-2.0.0-linux-x86_64/bin/mongod

If not, add :file:`/usr/local/bin` to your PATH and make it persistent in your :file:`.bash_profile`:

.. code-block:: bash

  $ export PATH=/usr/local/bin:"$PATH"
  $ echo 'export PATH=/usr/local/bin:"$PATH"' >> $HOME/.bash_profile

.. note::
 If you unpack and use symlinks this way, the binaries will be accessible to all users in the system (including initscripts you may want to install later), and upgrades will be simpler, and can use the exact same steps above.

Running the Server
==================
|TokuMX| supports almost all of the command line options for basic |MongoDB|, along with many new options. The differences are described in Server Parameters.

To start the server in its default configuration, just run :program:`mongod` from its installation location. You can stop it with C-c.

If you have an existing :file:`/etc/mongodb.conf`, you can copy it to :file:`/etc/tokumx.conf` and run :program:`mongod --config /etc/tokumx.conf` to use its options.

.. warning::
  Be sure to change the dbpath and logpath options to avoid conflicting with any existing basic MongoDB data. Also, to avoid conflicting with a running server, either shut down basic MongoDB or change the port option.

To connect to the |TokuMX| server, use the :program:`mongo` program:

.. code-block:: bash

  $ mongo
  TokuMX mongo shell v2.0.0-mongodb-2.4.10
  connecting to: test
  Welcome to the TokuMX shell.
  For interactive help, type "help".
  For more comprehensive documentation, see
    http://docs.mongodb.org/
  and the TokuMX Users' Guide available at
    https://www.percona.com/doc/percona-tokumx/
  Questions? Try the support group
  http://groups.google.com/group/tokumx-user
  > db.serverBuildInfo().tokumxVersion
  2.0.0

.. _replacing_mongodb:

Replacing MongoDB
=================
.. note::
  If migrating from an existing |MongoDB| installation, copy any relevant configuration from :file:`/etc/mongodb.conf` to :file:`/etc/tokumx.conf`.

|TokuMX| is not file-format compatible with |MongoDB|, so you must export your data from your existing |MongoDB| installation and import the data into |TokuMX|. This can be accomplished by using :program:`mongodump` and :program:`mongorestore`.


This procedure is described in :ref:`tokumx_migration` along with advanced techniques for online (no-downtime) migrations of replica sets, and migrations of sharded clusters.

.. _upgrading_tokumx:

Upgrading TokuMX
================
Unless otherwise noted, |TokuMX| is file-format compatible with prior versions.

.. important::
  |TokuMX| 1.3.0 introduced |MongoDB| 2.4 compatibility, and as such you must follow the `Upgrade to MongoDB 2.4 <http://docs.mongodb.org/manual/release-notes/2.4-upgrade/>`_ procedure if upgrading a ``pre-1.3.0`` |TokuMX| environment to 1.3.0 or newer, or if upgrading from |MongoDB| 2.2 to |TokuMX| 1.3.0 or newer.

.. _single_server:

Single Server
-------------
To upgrade a single |TokuMX| server, you must cleanly shut down the old server before installing and starting the new binaries.

1. Shutdown the server

   Perform a clean shutdown of the old server. This can be done in two ways:

   1. Command line

      Add the ``--shutdown`` command line option to your normal :program:`mongod` options.

       
      .. code-block:: bash

         $ mongod --config /etc/tokumx.conf --shutdown
      
   2. mongo shell

      Use the db.shutdownServer() shell function.

      .. code-block:: bash

        > use admin
        > db.shutdownServer()

2. Install and restart

  Install the new |TokuMX| binaries, and restart :program:`mongod`.

.. warning::
  Some OS initscripts will time out if shutdown takes too long, and will then send a ``KILL`` signal, which will abort the clean shutdown. If this happens, the upgrade may fail. It is recommended to shut down the server manually, and avoid using init scripts.

.. _replica_set:

Replica Set
-----------
To upgrade a |TokuMX| replica set, you should upgrade each machine in the set one at a time, starting with arbiters, then secondaries, then finally upgrade the primary.

.. important::
  TokuMX 1.4.0 introduced new oplog types that cannot be processed by secondaries running TokuMX 1.3.x and lower. Therefore, if a member running 1.4.0 or greater becomes primary, it may write oplog entries that will not be processed by members with earlier versions. For this reason, you should always upgrade all secondary machines first.
  TokuMX 2.0.0 also introduced new oplog types that cannot be processed by secondaries running TokuMX 1.5.x and lower. Therefore, like when upgrading to 1.4.0, when upgrading to 2.0.0, be sure to upgrade all secondary machines first.

.. tip::
  You may wish to use `Replica Set priority <http://docs.mongodb.org/manual/reference/replica-configuration/#local.system.replset.members[n].priority>`_ to have one machine try to stay primary during the upgrade process.

1. Upgrade all secondaries
Upgrade each secondary in the set, one at a time, according to the instructions for :ref:`single_server`. Make sure to wait for each member to return to ``SECONDARY`` status (in ``rs.status()``) before upgrading the next member.

2. Step down the primary
Use ``rs.stepDown()`` to have the primary fail over to one of the newly upgraded secondaries.

3. Upgrade the primary
Upgrade the stepped down primary according to the instructions for :ref:`single_server`.

Sharded Cluster
---------------
1. Disable the balancer

  Disable the balancer using ``sh.setBalancerState(false)``.

2. Upgrade routers

  Upgrade all :program:`mongos` instances by shutting them down and restarting them with the new binaries.

3. Upgrade config servers

  Upgrade all :program:`mongod` config servers (upgrading the first server listed in the :program:`mongos --configdb` option last).

4. Upgrade shard servers

  Upgrade each shard according to the instructions for :ref:`replica_set`.

5. Re-enable the balancer

  Re-enable the balancer using ``sh.setBalancerState(true)``.



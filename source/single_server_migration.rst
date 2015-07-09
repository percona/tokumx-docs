.. _single_server_migration:

=============
Single Server
=============

This section explains how to do an offline data migration from basic |MongoDB| to |TokuMX|, for a single server, converting the existing data in |MongoDB| to |TokuMX|.

Instructions
============

1. Shut down basic |MongoDB|

   Shut down the existing MongoDB server to make sure you get a consistent backup.

2. Take a backup

   Back up the |MongoDB| database with :program:`mongodump`. You will need the :variable:`dbpath` from your command-line options or :file:`/etc/mongodb.conf` (this is often :file:`/var/lib/mongodb`), and you will need to choose a location for the backup (here, :file:`/var/lib/mongodb.backup`).

   .. code-block:: bash 

     $ sudo mongodump --dbpath /var/lib/mongodb --out /var/lib/mongodb.backup

3. Uninstall |MongoDB|

   Uninstall MongoDB. You can also remove the old ``dbpath`` since you have a backup.

4. Install |TokuMX|

   Install |TokuMX| :ref:`from tarballs <installation>` or :ref:`from packages <installation_from_packages>` on all machines. If your distribution's package automatically starts |TokuMX|, stop it for now.

   Take a moment to transfer any important configuration from your basic |MongoDB| configuration file (usually :file:`/etc/mongodb.conf`) to your |TokuMX| configuration file (usually :file:`/etc/tokumx.conf`).

5. Import the backup

   Import your backup with :program:`mongorestore`. You do not need to specify :variable:`dbpath`, it will connect to the running server by default.

   .. code-block:: bash

     $ mongorestore /var/lib/mongodb.backup

6. Verify the migration

   Connect with the :program:`mongo` command and verify that you're connected to |TokuMX| and your data has been migrated.

   .. code-block:: javascript

     > db.serverBuildInfo().tokumxVersion
     2.0.0
     > show dbs
     ...


Collection Options
==================

For some data sets, it may make sense to use some of TokuMX's :ref:`collection_and_index_options` for your migrated data.

You can use the new ``--defaultCompression``, ``--defaultPageSize``, and ``--defaultReadPageSize`` options to :program:`mongorestore` to change the settings used to create newly loaded collections and indexes.

.. tip::
  Advanced users can modify the :file:`metadata.json` of any data dump before loading it to get full control of the indexing options after loading with :program:`mongorestore`.

  You can give collections a :option:`primaryKey`, make secondary indexes clustering, even add or remove secondary indexes by editing that file.



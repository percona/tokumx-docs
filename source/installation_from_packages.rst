.. _installation_from_packages:

============================
 Installation from Packages
============================

.. _centos:

RHEL/CentOS
===========

|TokuMX| is available for *CentOS* 5, 6 and 7.

The packages come in three components:

 * :program:`tokumx-enterprise` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :program:`tokumx-enterprise-common` contains the Fractal Tree indexing libraries and some documentation.
 * :program:`tokumx-enterprise-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important:: 
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. Install the Percona repository

   You can install Percona yum repository by running the following command as a ``root`` user or with sudo:

   .. code-block:: bash

     yum install http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm

   You should see some output such as the following:

   .. code-block:: bash

     Retrieving http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
     Preparing...                ########################################### [100%]
        1:percona-release        ########################################### [100%]

.. note::

  *RHEL*/*Centos* 5 doesn't support installing the packages directly from the remote location so you'll need to download the package first and install it manually with :program:`rpm`:

    .. code-block:: bash

      wget http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
      rpm -ivH percona-release-0.1-3.noarch.rpm

2. Testing the repository

   Make sure packages are now available from the repository, by executing the following command:

   .. code-block:: bash

     yum list | grep tokumx

You should see output similar to the following:

   .. code-block:: bash

     ...
     libtokumx-enterprise.x86_64                 2.0.2-1.el6                  percona-release-x86_64
     libtokumx-enterprise-devel.x86_64           2.0.2-1.el6                  percona-release-x86_64
     tokumx-enterprise.x86_64                    2.0.2-1.el6                  percona-release-x86_64
     tokumx-enterprise-common.x86_64             2.0.2-1.el6                  percona-release-x86_64
     tokumx-enterprise-debuginfo.x86_64          2.0.2-1.el6                  percona-release-x86_64
     tokumx-enterprise-server.x86_64             2.0.2-1.el6                  percona-release-x86_64
     ...

3. Install the packages

   You can now install |Percona TokuMX| by running:

   .. code-block:: bash

     yum install tokumx-enterprise

.. tip:: 
  After installing, read the instructions for :ref:`upgrading_tokumx`.

.. note::
  To control the :program:`mongod` data server, use service:

  .. code-block:: bash
                                                        
    $ sudo service tokumx start
    $ sudo service tokumx restart
    $ sudo service tokumx stop

.. note::
  To enable |TokuMX| on boot, use :program:`chkconfig`:

  .. code-block:: bash
                                                        
    $ sudo chkconfig tokumx on
    $ sudo chkconfig tokumx off

.. _debian_and_ubuntu:

Debian and Ubuntu
=================

|TokuMX| ready-to-use packages are from the |Percona| repositories.  

Supported Releases:

* Debian:

 * 7.0 (wheezy)
 * 8.0 (jessie)

* Ubuntu:

 * 12.04LTS (precise)
 * 14.04LTS (trusty)
 * 14.10 (utopic)
 * 15.04 (vivid)

The packages come in four components:

 * :file:`tokumx-enterprise` is a metapackage that installs the full |TokuMX| distribution.
 * :file:`tokumx-enterprise-clients` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :file:`tokumx-enterprise-common` contains the Fractal Tree indexing libraries and some documentation.
 * :file:`tokumx-enterprise-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important:: 
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. Import the public key for the package management system

  *Debian* and *Ubuntu* packages from *Percona* are signed with the Percona's GPG key. Before using the repository, you should add the key to :program:`apt`. To do that, run the following commands as root or with sudo:

  .. code-block:: bash

    $ sudo apt-key adv --keyserver keys.gnupg.net --recv-keys 1C4CBDCDCD2EFD2A

  .. note::

     In case you're getting timeouts when using ``keys.gnupg.net`` as an alternative you can fetch the key from ``keyserver.ubuntu.com``.

2. Create the :program:`apt` source list for Percona's repository:

   You can create the source list and add the percona repository by running:

   .. code-block:: bash

   $ echo "deb http://repo.percona.com/apt "$(lsb_release -sc)" main" | sudo tee /etc/apt/sources.list.d/percona.list

   Additionally you can enable the source package repository by running:

   .. code-block:: bash

   $ echo "deb-src http://repo.percona.com/apt "$(lsb_release -sc)" main" | sudo tee -a /etc/apt/sources.list.d/percona.list

3. Remember to update the local cache:

   .. code-block:: bash

     $ sudo apt-get update

4. After that you can install the server package:

   .. code-block:: bash

     $ sudo apt-get install tokumx-enterprise


.. tip::
  After installing, read the instructions for :ref:`upgrading_tokumx`.

To control the :program:`mongod` data server, use :program:`service`:
  
.. code-block:: bash
    
  $ sudo service tokumx start
  $ sudo service tokumx restart
  $ sudo service tokumx stop

.. note:: 

  *Debian* 8.0 (jessie) and *Ubuntu* 15.04 (vivid) come with `systemd <http://freedesktop.org/wiki/Software/systemd/>`_ as the default system and service manager so you can invoke all the above commands with ``sytemctl`` instead of ``service``. Currently both are supported.

To enable |TokuMX| on boot, use :program:`chkconfig`:

.. code-block:: bash

  $ sudo chkconfig tokumx on
  $ sudo chkconfig tokumx off


.. _installation_from_packages:

============================
 Installation from Packages
============================

.. _centos:

CentOS
======
|TokuMX| is available for *CentOS* 5 and 6.

The packages come in three components:

 * :program:`tokumx` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :program:`tokumx-common` contains the Fractal Tree indexing libraries and some documentation.
 * :program:`tokumx-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important:: 
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. For a full installation, download all three packages from the download page.

  You should have these files in your current directory:

    * :file:`tokumx-2.0.0-2.el6.x86_64.rpm`
    * :file:`tokumx-common-2.0.0-2.el6.x86_64.rpm`
    * :file:`tokumx-server-2.0.0-2.el6.x86_64.rpm`

2. Install the downloaded packages with :program:`yum`.
                                                    
.. code-block:: bash
                                                                
  $ sudo yum install tokumx*.rpm

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

.. _fedora:

Fedora
======
|TokuMX| is available for *Fedora* 20.

The packages come in three components:

 * :file:`tokumx` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :file:`tokumx-common` contains the Fractal Tree indexing libraries and some documentation.
 * :file:`tokumx-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important:: 
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. For a full installation, download all three packages from the download page.

  You should have these files in your current directory:
 
    * :file:`tokumx-2.0.0-2.fc20.x86_64.rpm`
    * :file:`tokumx-common-2.0.0-2.fc20.x86_64.rpm`
    * :file:`tokumx-server-2.0.0-2.fc20.x86_64.rpm`

2. Install the downloaded packages with :program:`yum`.

.. code-block:: bash

  $ sudo yum install tokumx*.rpm

.. tip::
  After installing, read the instructions for :ref:`upgrading_tokumx`.

.. note::
  To control the :program:`mongod` data server, use :program:`systemctl`:

  .. code-block:: bash
                                                                                                                        
    $ sudo systemctl start tokumx
    $ sudo systemctl restart tokumx
    $ sudo systemctl stop tokumx
                                                                                                                
.. note::
  To enable TokuMX on boot, use systemctl:

  .. code-block:: bash
                                                                                                                  
    $ sudo systemctl enable tokumx
    $ sudo systemctl disable tokumx

.. _debian:

Debian
======
|TokuMX| is available for *Debian* 7.

The packages come in four components:

 * :file:`tokumx` is a metapackage that installs the full |TokuMX| distribution.
 * :file:`tokumx-clients` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :file:`tokumx-common` contains the Fractal Tree indexing libraries and some documentation.
 * :file:`tokumx-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important:: 
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. Add the Tokutek package signing key.
                                
.. code-block:: bash

  $ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key 505A7412

You can check that the fingerprint is correct:

.. code-block:: bash

  $ sudo apt-key finger 505A7412
  /etc/apt/trusted.gpg
  --------------------
  pub   2048R/505A7412 2014-01-27
  Key fingerprint = DA56 C65D 432E DAB1 F183  AA6F 70A4 E325 505A 7412
  uid                  Timothy Callaghan (Tokutek Key) <tim@tokutek.com>
  sub   2048R/46A1A9B9 2014-01-27

2. Add an entry for the |TokuMX| package repository for your *Debian* release (wheezy).

.. code-block:: bash
                                                                                                                
  $ echo "deb [arch=amd64] http://s3.amazonaws.com/tokumx-debs wheezy main" \
  | sudo tee /etc/apt/sources.list.d/tokumx.list

3. Update :program:`apt` and install |TokuMX|.

.. code-block:: bash
                                                                                                                                        
  $ sudo apt-get update
  $ sudo apt-get install tokumx

.. tip::
  After installing, read the instructions for :ref:`upgrading_tokumx`.

.. note::
  To control the :program:`mongod` data server, use :program:`service`:
  
  .. code-block:: bash
                                                                                                                                        
    $ sudo service tokumx start
    $ sudo service tokumx restart
    $ sudo service tokumx stop

.. note:: 
  To enable |TokuMX| on boot, use :program:`chkconfig`:

  .. code-block:: bash
                                                                                                                                        
    $ sudo chkconfig tokumx on
    $ sudo chkconfig tokumx off


.. _ubuntu:

Ubuntu
======
|TokuMX| is available for *Ubuntu* 12.04, 12.10, 13.04, 13.10, and 14.04.

The packages come in four components:

 * :file:`tokumx` is a metapackage that installs the full |TokuMX| distribution.
 * :file:`tokumx-clients` contains the :program:`mongo` client shell and various tools like :program:`mongoimport` and :program:`mongoexport`.
 * :file:`tokumx-common` contains the Fractal Tree indexing libraries and some documentation.
 * :file:`tokumx-server` contains the :program:`mongod` database server and the :program:`mongos` routing server.

.. important::
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. Add the Tokutek package signing key.

.. code-block:: bash

  $ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-key 505A7412

You can check that the fingerprint is correct:

.. code-block:: bash

  $ sudo apt-key finger 505A7412
  /etc/apt/trusted.gpg
  --------------------
  pub   2048R/505A7412 2014-01-27
  Key fingerprint = DA56 C65D 432E DAB1 F183  AA6F 70A4 E325 505A 7412
  uid                  Timothy Callaghan (Tokutek Key) <tim@tokutek.com>
  sub   2048R/46A1A9B9 2014-01-27

2. Add an entry for the |TokuMX| package repository for your *Ubuntu* release (precise, quantal, raring, saucy, or trusty).

.. code-block:: bash
                                                                                                                
  $ echo "deb [arch=amd64] http://s3.amazonaws.com/tokumx-debs $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/tokumx.list

3. Update :program:`apt` and install |TokuMX|.

.. code-block:: bash
                                                                                                                
  $ sudo apt-get update
  $ sudo apt-get install tokumx

.. tip::
  After installing, read the instructions for :ref:`upgrading_tokumx`.

.. note::
  To control the :program:`mongod` data server, use :program:`service`:

  .. code-block:: bash

    $ sudo service tokumx start
    $ sudo service tokumx restart
    $ sudo service tokumx stop

.. note:: 
  To enable |TokuMX| on boot, use :program:`update-rc.d`:

  .. code-block:: bash

    $ sudo update-rc.d tokumx defaults
    $ sudo update-rc.d tokumx remove

.. _arch:

Arch
====

|TokuMX| is available for *Arch Linux*.

.. important::
  Before starting, make sure you have read about :ref:`replacing_mongodb` and :ref:`tokumx_migration`. You may want to export your existing data out of basic |MongoDB| before installing |TokuMX|.

1. Add the |TokuMX| package repository to your :file:`/etc/pacman.conf`: 
                        
.. code-block:: none

  [tokumx]
  Server = https://s3.amazonaws.com/tokumx-archlinux/$arch

2. Fetch and locally sign the repository signing key:

.. code-block:: bash
                                        
  $ sudo pacman-key --recv-keys 505A7412 --keyserver keyserver.ubuntu.com
  $ sudo pacman-key --lsign-key 505A7412

You can verify the fingerprint by comparing it with this output:

.. code-block:: bash
                                        
  $ sudo pacman-key --finger 505A7412
  pub   2048R/505A7412 2014-01-27
  Key fingerprint = DA56 C65D 432E DAB1 F183  AA6F 70A4 E325 505A 7412
  uid       [  full  ] Timothy Callaghan (Tokutek Key) <tim@tokutek.com>
  sub   2048R/46A1A9B9 2014-01-27

3. Install the downloaded packages with :program:`pacman`.

.. code-block:: bash
                                        
  $ sudo pacman -Sy
  $ sudo pacman -S tokumx

.. tip:: 
  After installing, read the instructions for :ref:`upgrading_tokumx`.

.. note::
  To control the mongod data server, use systemctl:
  
  .. code-block:: bash
                                                                                                                
    $ sudo systemctl start tokumx
    $ sudo systemctl restart tokumx
    $ sudo systemctl stop tokumx

.. note:: 
  To enable |TokuMX| on boot, use :program:`systemctl`:

  .. code-block:: bash
                                                                                                                
    $ sudo systemctl enable tokumx
    $ sudo systemctl disable tokumx




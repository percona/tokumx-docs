.. _collection_and_index_options:

============================
Collection and Index Options
============================

.. _collection_options:

Collection Options
==================

In |TokuMX|, all collection data is stored in Fractal Tree indexes. Documents are stored in a :option:`primaryKey` whose index is, by default, ``{_id: 1}`` and the value stored is the BSON document. Secondary indexes store the user-specified key and use the :option:`primaryKey` to reference the full document.

The syntax for creating a new collection is unchanged:

.. code-block:: javascript

  db.createCollection('foo')

``db.createCollection()`` will create a collection using all of our default values. However, collections and indexes in |TokuMX| support several new options to control storage on disk.

These options can be mixed in the options BSON, for example:

.. code-block:: javascript

  db.createCollection('foo', {compression:  'quicklz',
        readPageSize: '16KB',
        primaryKey:   {ts: 1, _id: 1}})

In other drivers, these options can be set by adding parameters to the create command, for example in Java:

.. code-block:: java

  DBObject cmd = new BasicDBObject();
  cmd.put("create", "foo");
  cmd.put("compression", "quicklz");
  cmd.put("pageSize", 16*1024);
  DBObject primaryKeyObj = new BasicDBObject();
  primaryKeyObj.put("ts", 1);
  primaryKeyObj.put("_id", 1);
  cmd.put("primaryKey", primaryKeyObj);
  CommandResult result = db.command(cmd);

All collection options can be specified at create time, and index-type options affect the :option:`primaryKey` index. Those same index-type options can also be used on secondary indexes when they are created, see Index Options.

.. option:: primaryKey

  :default: ``{_id: 1}``

Supported since 1.4.0

The primary key used to store documents, and used by secondary indexes to reference those documents. This is always a :option:`clustering` index. See `What’s new in TokuMX 1.4, Part 1: Primary keys <http://www.tokutek.com/2014/02/whats-new-in-tokumx-1-4-part-1-primary-keys/>`_ for more information.

Setting the primary key can be useful when you know you want a clustering index but don't want to pay for the storage of an additional clustering index on ``_id``. You may also want to set the primary key if you want to use a :option:`partitioned` collection.

The primary key must end in ``{_id: 1}``.

Example:

.. code-block:: javascript

  db.createCollection('foo', {primaryKey: {ts: 1, _id: 1}})

.. option:: partitioned

  :default: false

Supported since 1.5.0

Specifies that this collection will be partitioned, according to the :option:`primaryKey`. See :ref:`partitioned_collections` for more details.

Example:

.. code-block:: javascript

  db.createCollection('foo', {partitioned: true
  primaryKey:  {ts: 1, _id: 1}})

.. variable:: compression

Specifies the index option :option:`compression` for the :option:`primaryKey`.

Example:

.. code-block:: javascript

  db.createCollection('foo', {compression: 'quicklz'})

.. variable:: pageSize

Specifies the index option :option:`pageSize` for the :option:`primaryKey`.

Example:
                            
.. code-block:: javascript

  db.createCollection('foo', {pageSize: '8MB'})

.. variable:: readPageSize

Specifies the index option :option:`readPageSize` for the :option:`primaryKey`.

Example:

.. code-block:: javascript

  db.createCollection('foo', {readPageSize: '64KB'})

.. option:: fanout

Specifies the index option fanout for the :option:`primaryKey`.

Example:

.. code-block:: javascript

  db.createCollection('foo', {fanout: 64})

.. _index_options:

Index Options
=============

Collection indexes are also stored in Fractal Tree indexes. Secondary indexes store the user-specified key and use the :option:`primaryKey` to reference the full document.

The syntax for creating a new index is unchanged:

.. code-block:: javascript

  db.foo.ensureIndex({x: 1})

``db.collection.ensureIndex()`` will create an index using all of our default values. Indexes in |TokuMX| support several new options to control storage on disk.

These options can be mixed in the options BSON, for example:

.. code-block:: javascript

  db.foo.ensureIndex({x: 1}, {compression:  'quicklz',
        readPageSize: '16KB'})

To control the options for the :option:`primaryKey` index, specify the below options as :ref:`collection_options`.

.. option:: clustering

  :default: false

If true, denotes that this index will be clustering.

Secondary indexes in basic |MongoDB| store the indexed fields and a pointer to the document. When a query is run using the secondary index, |MongoDB| uses the secondary index to find the pointer, or pointers, then uses those pointers in the secondary index to lookup the document from the heap-based data store.

|MongoDB| supports "covered" indexes. A covered index includes fields at the beginning of the index key to support lookups and range scans; the remainder of the key is defined for the values that need to be retrieved as part of the lookup. For example, an index on ``{x: 1, a: 1}`` "covers" the query ``db.foo.find({x: 30}, {x: 1, a: 1})``. Adding new fields to documents means that they are no longer covered by existing indexes, so you’ll need to drop and recreate them or build new indexes entirely, if you want to cover added fields.

Secondary indexes in |TokuMX| store the indexed fields and a copy of the :option:`primaryKey`, which is used—instead of the pointer in basic |MongoDB| to look up the full document. |TokuMX| also supports this same covering technique.

In addition, |TokuMX| offers clustering indexes. A clustering index, rather than storing a reference to the :option:`primaryKey`, instead stores another copy of the full document. This saves the I/O required to find the full document in the :option:`primaryKey` index. Essentially, a clustering index "covers" all queries that use that index.

.. important::
  Clustering indexes require storing a full additional copy of the document itself, which is generally not a problem when documents are compressible. Clustering indexes must also be maintained for any update on the collection, not just updates that affect the index's key.
  However, the I/O savings for queries that can use a clustering index is dramatic.

Example:

.. code-block:: javascript

  db.foo.ensureIndex({x: 1}, {clustering: true})

.. option:: compression

  :default: ``zlib``
  :calues: ``zlib``, ``quicklz``, ``lzma``, ``none``.

The compression method used on fractal tree nodes on disk.

Compression is generally a tradeoff between CPU cost and size on disk. Some compressors use more CPU to get a better compression factor. Others sacrifice compression factor to get faster compression and decompression speed.

The default, ``zlib``, is a balanced compression algorithm good for nearly all workloads. The ``lzma`` compression algorithm is very expensive but can get a better compression ratio for most data, and is therefore better suited to archival applications. The ``quicklz`` compression algorithm is typically faster than ``zlib`` but doesn't achieve as good a compression ratio for most data.

Example:
                            
.. code-block:: javascript

  db.createCollection('foo', {compression: 'quicklz'})
                   
.. option:: pageSize

  :default: '4MB'

The block size used to write out fractal tree nodes to disk, before compression.

Page size represents the size of the nodes in the Fractal Tree index (both internal nodes and leaf nodes). Our internal nodes store the pivots for the node, and a message buffer for each path down the tree, for fanout buffers per node.

Pages can be larger than this defined size if a document is larger than the size, this will not cause an error.

Since all nodes are compressed before writing they are generally much smaller when written to disk.

Example:
                            
.. code-block:: javascript

  db.foo.ensureIndex({x: 1}, {pageSize: '8MB'})

.. option:: readPageSize
                            
  :default: '64KB'

The block size used to read portions of a fractal tree node from disk for query, before compression.

Read page size represents the smallest portion of a leaf node that can be read from disk. A leaf node in a Fractal Tree index is actually a set of pivots and a series of "basement nodes", each of size at most :option:`readPageSize`, when uncompressed. These "basement nodes" can be read in and cached in-memory independently of other basement nodes.

Example:
                            
.. code-block:: javascript

  db.foo.ensureIndex({x: 1}, {readPageSize: '64KB'})

.. option:: fanout
                
  :default: 16
  :Minimum: 4
  
Supported since: 1.4.0

The maximum fanout of the fractal tree. A higher fanout favors read performance, a lower fanout favors write performance.

Example:

.. code-block:: javascript

  db.foo.ensureIndex({x: 1}, {fanout: 64})

.. _modifying_index_options:

Modifying Index Options
=======================

Index options (:option:`compression`, :option:`pageSize`, etc.) can be changed after the index is created.

This modifies the header so that all tree nodes written out after this point will use the new options; nodes that aren't changed won't see the new options until they are. This makes the modification instantaneous, but means the effect will be delayed. You can later force all nodes to be rewritten by optimizing the indexes with ``reIndex``.

The interface for modifying index options is ``db.collection.reIndex(index, options)``.

If ``index`` is not present, all indexes for that collection are affected. If ``index`` is present, it can be an index name as a string (e.g.``'a_1_b_1'``) or a key pattern as an object (e.g.``{a: 1, b: 1}``), or the string ``'*'`` to indicate all indexes.

If ``options`` is not present or is the empty object ``{}``, ``reIndex`` runs a "hot optimize" on the specified index(es). This causes all internal nodes to be flushed, and causes all tree nodes to be rewritten. If ``options`` is a non-empty object, it may have the fields :option:`compression`, :option:`pageSize`, :option:`readPageSize`, and :option:`fanout`, and it alters those attributes instead of running an optimize.

Examples:
Optimize all indexes:

.. code-block:: javascript

  db.collection.reIndex()
  // or
  db.collection.reIndex('*')

Optimize just the ``_id`` index:

.. code-block:: javascript

  db.collection.reIndex('_id_')
  // or
  db.collection.reIndex({_id: 1})

Change the compression method of all indexes to ``lzma``:

.. code-block:: javascript

  db.collection.reIndex('*', {compression: 'lzma'})

Change the compression method of the all but the _id index to 'quicklz' and the _id index to 'lzma', and force just the _id index to be converted to 'lzma' immediately:

.. code-block:: javascript

 db.collection.reIndex('*', {compression: 'quicklz'})
 db.collection.reIndex({_id: 1}, {compression: 'lzma'})
 db.collection.reIndex({_id: 1})

The ``reIndex`` command can be used directly by drivers, by running it as a normal command on the database containing the target collection, with a command object of the form

.. code-block:: javascript

  {reIndex: "collection_name", [index: [indexName|keyPattern|"*"]], [options: obj]}

where ``index`` and ``options`` are optional parameters, and if present, have the same meaning as above.

Caveats
=======

When creating a unique index, it is possible to add the option ``dropDups``. This is an arbitrarily destructive operation, so it was not implemented in |TokuMX|. Even if it were implemented, there is no way for |TokuMX| to be sure that it dropped the same documents that |MongoDB| would have dropped.

Therefore, |TokuMX| ignores the ``dropDups`` option. If there are duplicate entries while building a unique index, the index build will fail.

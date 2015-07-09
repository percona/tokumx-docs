.. _fast_updates:

============
Fast Updates
============

Supported since 2.0.0

Beginning in version 2.0, users can dramatically improve the performance of many of their updates, in some cases greater than 10x.

Typically, in many databases, updates are performed with a "read-modify-write" process, requiring an initial query to look up the document(s) to be modified. In a write-optimized database like |TokuMX|, the read phase is often much more expensive than the write phase. When applicable, fast updates skip the read phase, thereby improving performance.

The tradeoff with fast updates (detailed in :ref:`specification`) is that error cases become harder to detect. The number of documents updated, as reported in ``getLastError`` may be inaccurate, because fast updates are not performing the reads necessary to accurately report these values.

Benefiting from fast updates requires properly designing the query of your update. There are two ways:

* Make sure the query of the update uniquely identifies the :option:`primaryKey` of the document to be updated. So, if your collection's :option:`primaryKey` is ``{_id: 1}``, make sure your query includes an equality clause on ``_id``.

* Make sure the query is covered by a secondary key. That is, all the fields in the query must exist in some secondary key. For example, on a collection with a secondary key of ``{a: 1, b: 1}``, an update with the query ``{a: 100, b: 200}`` will see some benefit with fast updates. On the other hand, an update with the query ``{a: 100, c: 100}`` would not see benefits with fast updates.

.. tip:: 
  The first method, designing queries around the :option:`primaryKey`, will likely yield better performance gains than the second method, designing queries around secondary keys.

Overview
========

Fast updates use a different algorithm for completing their work:

* The :option:`primaryKey` that is affected by the update is identified, without fetching the entire document. This can be done in either of two ways:

  * The document's :option:`primaryKey` is uniquely identified by the update's query.

  * The document's :option:`primaryKey` is uniquely identified by a query performed in the system on some secondary key. If the query is fully satisfied by the fields indexed by a secondary key, this is sufficient. Since secondary indexes are usually smaller than the primary key index, these queries are faster than a query that needs to look up the full document.

  .. warning:: 
    If some fields are needed by the query and are not indexed by the secondary key, then the server must fetch the whole document, and the update is not eligible for the fast update optimization.

* Given a uniquely identified primary key, a write is performed to the system that describes the full query and the update to be performed.

.. note:: 
  Because |TokuMX| is a write optimized database, the actual update may be applied to the document sometime in the future. Therefore, checks such as whether the document exists, whether the query fully matches the document, or whether the update is valid are not done until later. For this reason, with fast updates, failures are silent. This is expanded upon in :ref:`specification`.

.. _specification:

Specification
=============

By default, updates use the standard read-modify-write algorithm. To enable fast updates, set :variable:`fastUpdates` to ``true``.

Because fast updates do not perform reads, several operations of an update that require a read may be deferred. These operations are:

* Accurately determining how many documents are affected by the update.

* Checking if the update application has errors. For example, having an update try to perform a ``$inc`` on a text field is an error that requires a read.

Because fast updates do not perform reads, they have the following consequences:

* Updates that normally return an error may not return an error. Instead, the update will silently no-op.

* Updates may not accurately report the number of documents updated in `getLastError.n <http://docs.mongodb.org/v2.4/reference/command/getLastError/#getLastError.n>`_ and `getLastError.updatedExisting <http://docs.mongodb.org/v2.4/reference/command/getLastError/#getLastError.updatedExisting>`_

Silent error example:

Without fast updates, this update has a type error:

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 0, "state" : "blah" }
  > db.foo.update({_id: 0}, {$inc: {state: 1} } )
  Cannot apply $inc modifier to non-number
  > db.foo.find()
  { "_id" : 0, "state" : "blah" }

With fast updates, the error is not discovered at update time. When the update is applied (during the find(), the error is detected, but is not returned to the client. The update is simply discarded.

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 0, "state" : "blah" }
  > db.foo.update({_id: 0}, {$inc: { state: 1} } )
  > db.foo.find()
  { "_id" : 0, "state" : "blah" }

Affected rows example:

Here is an example of an update that does not accurately return the number of rows updated. First, let's look at what happens with fast updates disabled:

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 0, "a" : 0, "b" : 0 }
  > db.foo.update({_id: 0, a: "wrong value"}, {$inc: {b: 1}})
  > db.runCommand("getLastError")
  {
      "updatedExisting" : false,
      "n" : 0,
      "connectionId" : 2,
      "err" : null,
      "ok" : 1
  }

With fast updates, the fact that the update's a field doesn't match the document isn't discovered until later. The update framework just knows that it sent an update to the document with ``{_id: 0}``

.. code-block:: javascript

  > db.foo.find()
  { "_id" : 0, "a" : 0, "b" : 0 }
  > db.foo.update({_id: 0, a: "wrong value"}, {$inc: {b: 1}})
  > db.runCommand("getLastError")
  {
      "updatedExisting" : true,
      "n" : 1,
      "connectionId" : 2,
      "err" : null,
      "ok" : 1
  }
  > db.foo.find()
  { "_id" : 0, "a" : 0, "b" : 0 }

Even though no update was actually applied, ``updatdeExisting`` and ``n`` are inaccurate.

.. _eligibility:

Eligibility
===========

In general, only updates that can modify the document by updating its image in the :option:`primaryKey` without modifying indexes can be fast. Modifying secondary indexes requires reading the full document to identify what the appropriate index entries are. As such, all updates are eligible, with the following exceptions:

* Updates that modify a secondary index. For example, with a secondary index of ``{a: 1}``, an update ``db.foo.update({_id: 0}, {$inc: {a: 1}})`` will not be fast.

* Collections that have a :option:`clustering` secondary index cannot perform fast updates. The reason is that maintaining the secondary index requires looking up the document.

* Updates that specify ``{upsert: true}``. Such an update needs to perform an existence query first, since if the document doesn't yet exist, it must modify secondary indexes.

  .. note::
    In future versions of TokuMX, upserts may possibly use the fast update optimization if there are no secondary indexes.

* Updates that replace a document entirely.

* Updates performed on capped collections.

Additionally, the following updates cannot currently be fast for implementation specific reasons:

* `Positional updates <http://docs.mongodb.org/manual/reference/operator/update/positional/>`_ in an array.

* Updates performed while a `background index <http://docs.mongodb.org/v2.4/tutorial/build-indexes-in-the-background/>`_ is being built.

Application Design
==================

There are some general guidelines for how to design an application to take the best advantage of fast updates.

The most important thing to do is to understand the semantics of fast updates, specifically, how the behaviors related to :ref:`error handling and getLastError <specification>` change. Make sure your application can tolerate these changes.

Once you've ensured your application can tolerate the semantics, the next thing is to do is see if your application, as is, can benefit from fast updates. Check ``db.serverStatus()`` for the :ref:`fast_updates_status` metrics and see how many possible updates tracked in ``db.serverStatus().metrics.fastUpdates.eligible`` would benefit. Based on these metrics, you may want to try just enabling fast updates on your server.

If you don't see many updates being counted under ``db.serverStatus().fastUpdates.eligible``, try to understand why your updates are not eligible. The reason must be one listed under :ref:`eligibility`. Try to make the changes necessary to make your updates eligible. If you wish to shard with a hashed ``_id`` field, make sure you run the ``shardCollection`` command with ``{clustering: false}``.

Lastly, try to design your schema such that queries used for updates uniquely identify the primary key. If your primary key is ``{_id: 1}``, try to make sure your update queries includes ``_id``. This makes your updates eligible to be run without a query, which results in the biggest possible gains. Your application may still benefit without this design pattern, but this is guaranteed to yield the best results.

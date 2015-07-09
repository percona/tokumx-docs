.. _sharding:

========
Sharding
========

Sharding in TokuMX works the same as sharding in MongoDB, and supports all the same user and admin commands and behaviors. The shard key index should, in almost all cases, be a clustering index, and this is the default when sharding a new collection.

TokuMX applications can often take advantage of a different sharding strategy to provide dramatic performance improvements over MongoDB, by leveraging clustering indexes to improve range queries and migrations together.

Range queries that can use the shard key index are great because they often can be satisfied by one or a small number of shards, avoiding broadcasting queries to all shards. In TokuMX, clustering indexes provide further range query optimization, so in a sharded collection that will support range queries on the shard key, it is recommended to make the shard key's index clustering.

Migrations in basic MongoDB have a big impact on performance. In basic MongoDB, users often need to pick a shard key in order to avoid migrations, which may prevent the use of a shard key that is good for the application's range queries.

In TokuMX, migrations, especially if the shard key is clustering, cause a much smaller impact on normal operations, so it is usually possible to choose the best shard key for your queries, and not worry about the migrations.

These two properties, when combined, mean that in TokuMX you can pick a shard key that gives you great range query performance, and take advantage of the clustering index to improve both migrations and range queries at the same time.

Creating Sharded Collections
When initially sharding a collection, TokuMX will create a clustering index on the shard key. If a non-clustering index already exists, it will issue a warning, and you must re-run the shardCollection command with {clustering: false} to override this if this is truly desired.

In TokuMX, the sh.shardCollection() wrapper and shardCollection command are somewhat expanded to support control over clustering indexes, and partitioned and sharded collections.

When possible, it is best to create a collection by calling sh.shardCollection(), rather than sharding an existing collection. This allows TokuMX to choose the best primaryKey for your shard key.

Creating a collection with range-based sharding:
By sharding a nonexistent collection, mongos will create a primaryKey that includes the shard key. This maximizes migration and range query performance with a clustering shard key index, without sacrificing disk space for an extra clustering index.

.. code-block:: javascript

  mongos> sh.shardCollection('test.coll', {a: 1, b: 1})
  { "collectionsharded" : "test.coll", "ok" : 1 }
  mongos> db.coll.getIndexes()
  [
      {
          "key" : {
              "a" : 1,
              "b" : 1,
              "_id" : 1
          },
          "unique" : true,
          "ns" : "test.coll",
          "name" : "primaryKey",
          "clustering" : true
      },
      {
          "key" : {
              "_id" : 1
          },
          "unique" : true,
          "ns" : "test.coll",
          "name" : "_id_"
      }
  ]

Creating a collection with hash-based sharding:
With hash-based sharding, range queries on the shard key don't make sense, and migrations hardly ever happen, so a clustering shard key is unnecessary. In this case, create the collection by passing clustering: false as the fourth parameter to sh.shardCollection(). The collection will be well pre-split to avoid any future migrations.

.. code-block:: javascript

  mongos> sh.shardCollection('test.coll', {a: 'hashed'}, false, false)
  { "collectionsharded" : "test.coll", "ok" : 1 }
  mongos> db.coll.getIndexes()
  [
      {
          "key" : {
              "_id" : 1
          },
          "unique" : true,
          "ns" : "test.coll",
          "name" : "_id_",
          "clustering" : true
      },
      {
          "key" : {
              "a" : "hashed"
          },
          "ns" : "test.coll",
          "name" : "a_hashed"
      }
  ]
  mongos> db.getSiblingDB('config').chunks.count({ns: 'test.coll'})
  1024

Without a clustering shard key, autosplitting can be expensive. It is often a good idea when using hash-based sharding with a non-clustering shard key to use --noAutoSplit on all mongos routers.
Without a clustering shard key, migrations are roughly as expensive as in basic MongoDB. With hash-based sharding, you can expect to never need to migrate a chunk, except when adding or removing shards. If this is required, be aware that migrations will not be as fast or low-impact as with a clustering shard key, and plan accordingly.

Optimizing Migrations
In basic MongoDB, migrations can have a strong impact on a running application. For this reason, many MongoDB administrators choose to schedule the balancer window to prevent the balancer from running during periods of peak application activity.

In TokuMX, since v1.4, migrations are far less intrusive, and can be made even cheaper with some planning. For an in-depth discussion of how this works, see What's new in TokuMX 1.4, Part 5: Faster chunk migrations. Here, we will only cover what tuning steps should be taken.

First, make sure all your servers are running v1.4 or later. If any aren't, servers will fall back to a compatibility implementation of migrations that uses upserts on the recipient shard, which may require random I/O to query for the existence of the document.

Next, ensure that your collection was created with a primaryKey that includes the shard key. This will minimize the I/O required to read the migrating chunk off the donor shard.

Finally, if your application allows it, turn off migrateUniqueChecks. This will further reduce I/O on the recipient shard by skipping unique checks on the _id field.

Turning off migrateUniqueChecks can be dangerous with certain data models, but if your shard key is included in your primaryKey as recommended above, this is perfectly safe.
On the command line:

.. code-block:: bash
 
  $ mongod --setParameter=migrateUniqueChecks=false

In the config file:

.. code-block:: text

  setParameter=migrateUniqueChecks=false

---
title: "Updating & Reindexing Search Schemas"
description: ""
project: "riak_kv"
project_version: "2.1.4"
menu:
  riak_kv-2.1.4:
    name: "Updating / Reindexing Search Schemas"
    identifier: "cluster_operations_search_schemas"
    weight: 117
    parent: "managing_cluster_operations"
toc: true
canonical_link: "https://docs.basho.com/riak/kv/latest/using/cluster-operations/update-reindex-search-schema"
---

<<Introduction>>

If you need to update a [search schema]()...

## Update the Search Schema

First, make updates to the schema file, which is typically found here `$RIAK_HOST/search/index/famous`.

For example, the following creates or updates an index called `famous` that uses the default schema:

```bash
curl -XPUT $RIAK_HOST/search/index/famous \ -H 'Content-Type: application/json' \ -d '{"schema":"_yz_default"}'
```

Then update the new schema within Riak KV by either:

* Performing a [rolling restart]() to trigger a Solr reload of the schema.
* Or starting a `riak attach` session and run:
  `yz_index:reload(<<"index_name">>).`

> **Note:**
>
> Please note the `riak attach` method used above is a blocking, cluster-wide call but it will trigger a live reload of the Solr schema. New data will now contain the updated fields but existing data will not have their indexes updated.

<<We recommend blank>>

## Reindex Existing Data

### Option 1

The first option is to simply rePUT all of your data.  You can do this by using a script or the [data migrator tool]() <<?>>.

### Option 2

1. Delete all of your Solr data from the index that you changed the schema on.  You do not need to delete the configurations.
2. Expire the YZ AAE trees by entering a `riak attach` session and running the following command:  `rpc:multicall([node() | nodes()], yz_entropy_mgr, expire_trees, []).`
3. Allow the trees to rebuild with the new index.  You can monitor the progress of this activity with `riak-admin search aae-status`

## AAE Tuning

By default search reindexing through AAE happens at the same rate as KV AAE. AAE trees are rebuilt at a rate of 1 tree per node per hour, and AAE trees are expired once every week. With these settings it likely will take a great deal of time for the index to rebuild after the trees expire.

The [Active Anti Entropy][0] settings are described in our documentation. Of particular interest to you I think will be the following settings in `riak.conf` format, shown with their default values:

```

anti_entropy.concurrency_limit = 2
anti_entropy.tree.build_limit.number = 1
anti_entropy.tree.build_limit.per_timespan = 1h
anti_entropy.tree.expiry = 1w
```

You can experiment with setting `anti_entropy.tree.build_limit.per_timespan` to something shorter, perhaps 15m and/or raising the `anti_entropy.tree.build_limit.number`. You can also raise the `anti_entropy.concurrency_limit` to, for example, 4. These changes should be made in `riak.conf` and do require a node restart to be applied. Please monitor the impact of these changes on your cluster's performance. We cannot currently adjust the AAE settings for KV and Search separately through `riak.conf`.  Changing these settings will allow the trees to rebuild faster and thus have your data reindexed faster; however, will place additional load on your cluster.

You can monitor the progress of the re-indexing with the command Ian mentioned, `riak-admin search aae_status`. Please be aware that until all partitions have been reindexed you may see inconsistent results due to the way the query plan is accessing different Solr backends that may not yet have all the data reindexed.
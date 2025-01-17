---
title: "Elasticsearch integration"
linkTitle: "Elasticsearch integration"
#weight:
date: 2021-07-12
tags:
  - search
  - elastic-search
description: >
  How to integrate Stroom with Elastic Search
---

## Introduction

Stroom v6.1 can pass data to Elasticsearch for indexing.
Indices created using this process (i.e. those containing a `StreamId` and `EventId` corresponding to a particular Stroom instance)
are searchable via a Stroom _dashboard_, much like a Stroom Lucene index.

This integration provides operators with the flexibility to utilise the additional capabilities of Elasticsearch,
(like clustering and replication) and expose indexed data for consumption by external analytic or processing tools.

This guide will take you through creating an Elasticsearch index, setting up an indexing pipeline, activating a stream processor and searching the indexed data in both Stroom and Elasticsearch.

### Assumptions

1. You have created an Elasticsearch cluster.
   For test purposes, you can quickly create a single-node cluster using Docker by following the steps in the {{< external-link "Elasticsearch Docs" "https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-dev-mode" >}}.
1. The Elasticsearch cluster is reachable via HTTP/S from all Stroom nodes participating in [stream processing]({{< relref "../../quick-start-guide/running.md" >}}).
1. Elasticsearch security is disabled.
1. You have a feed containing `Event` data.

### Key differences

1. Unlike with [Solr indexing]({{< relref "Solr.md" >}}), Elasticsearch field mappings are managed outside of Stroom, usually via the {{< external-link "REST API" "https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html#mappings" >}}.
1. Aside from creating the mandatory `StreamId` and `EventId` field mappings, explicitly defining mappings for other fields is optional.
   It is however, considered good practice to define these mappings, to ensure each field's data type is correctly parsed and represented.
   For text fields, it also pays to ensure that the {{< external-link "appropriate mapping parameters are used" "https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html" >}},
   in order to satisfy your search and analysis requirements - and meet system resource constraints.
1. Unlike both Solr and Lucene indexing, it is not necessary to mark a field as `stored` (i.e. storing its raw value in the inverted index).
   This is because Elasticsearch stores the content of the original document in the {{< external-link "_source field" "https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html" >}},
   which is retrieved when populating search results.
   Provided the `_source` field is enabled (as it is by default), a field is treated as `stored` in Stroom and its value doesn't need to be retrieved via an _extraction pipeline_.

## Indexing data

### Creating an index in Elasticsearch

The following cURL command creates an index named `stroom_test` in Elasticsearch cluster `http://localhost:9200` consisting of the following fields:

1. `StreamId` (mandatory, must be of data type `long`)
1. `EventId` (mandatory, must also be `long`)
1. `Name` (text). Uses the default analyzer, which tokenizes the text for matching on terms.
     `fielddata` is enabled, which allows for {{< external-link "aggregating on these terms" "https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html#fielddata-mapping-param" >}}.
1. `State` (keyword). Supports exact matching.

The created index consists of `5` shards.
Note that the shard count cannot be changed after index creation, without a reindex.
See {{< external-link "this guide" "https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html" >}} on shard sizing.

{{< command-line "user" "localhost" >}}
curl -X PUT "http://localhost:9200/stroom_test?pretty" -H 'Content-Type: application/json' -d'
(out)   {
(out)      "settings": {
(out)        "number_of_shards": 5
(out)      },
(out)      "mappings": {
(out)        "properties": {
(out)          "StreamId": {
(out)            "type": "long"
(out)          },
(out)          "EventId": {
(out)            "type": "long"
(out)          },
(out)          "Name": {
(out)            "type": "text",
(out)            "fielddata": true
(out)          },
(out)          "State": {
(out)            "type": "text",
(out)            "fielddata": true
(out)          }
(out)        }
(out)      }
(out)    }
(out)'
{{</ command-line >}}

After creating the index, you can add additional field mappings.
Note the {{< external-link "limitations" "https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html#updating-field-mappings" >}} in doing so, particularly the fact that it will not cause existing documents to be re-indexed.
It is worthwhile to test index mappings on a subset of data before committing to indexing a large event feed, to ensure the resulting search experience meets your requirements.

### Registering the index in Stroom

This step creates an _Elasticsearch Index_ in the _Stroom Tree_ and tells Stroom how to connect to your Elasticsearch cluster
and index. Note that this process needs to be repeated for each index you create.

#### Steps

1. Right-click on the folder in the _Explorer Tree_ where you wish to create the index
1. Select `New` / `Elasticsearch Index`
1. Enter a valid name for the index. It is a good idea to choose one that reflects either the feed name being indexed, or if
indexing multiple feeds, the nature of data they represent. 
1. In the index tab that just opened:
    1. Select the `Settings` tab
    1. Set the `Index` to the name of the index in Elasticsearch (e.g. `stroom_test` from the previous example)
    1. Set the `Connection URLs` to one or more Elasticsearch node URLs. If multiple, separate each URL with `,`.
       For example, a URL like `http://data-0.elastic:9200,http://data-1.elastic:9200` will balance requests to two data nodes
       within an Elasticsearch cluster. See [this document](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html) for guidance on node roles.
    1. Click `Test Connection`. If the connection succeeds, and the index is found, a dialog is shown indicating
    the test was successful. Otherwise, an error message is displayed.
    1. If the test succeeded, click the _save_ button in the top-left. The `Fields` tab will now be populated with fields from
    the Elasticsearch index.

{{% note %}}
The field mappings list is only updated when index settings are changed, or a Stroom indexing or search task begins.
The refresh button in the `Fields` tab does not have any effect.
{{% /note %}}

### Setting index retention

As with Solr indexing, index document retention is determined by defining a Stroom query.

Setting a retention query is optional and by default, documents will be retained in an index indefinitely.

It is recommended for indices containing events spanning long periods of time, that {{< external-link "Elasticsearch Index Lifecycle Management" "https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html" >}} 
be used instead. The capabilities provided, such as automatic rollover to warm or cold storage tiers, are well worth considering, especially in high-volume production clusters.

#### Considerations when implementing ILM

1. It is recommended that _[data streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html)_ are used when
indexing data. These allow easier rollover and work well with ILM policies. A _data stream_ is essentially a container for multiple date-based indices
and to a search client such as Stroom, appears and is searchable like a normal Elasticsearch index.
1. Use of _data streams_ requires that a `@timestamp` field of type `date` be defined for each document (instead of say, `EventTime`).
1. Implementing ILM policies requires careful capacity planning, including anticipating search and retention requirements.

### Creating an indexing pipeline

As with Lucene and Solr indexing pipelines, indexing data using Elasticsearch uses a pipeline filter.
This filter accepts `<record>` elements and for each, sends a document to Elasticsearch for indexing.

Each `<data>` element contained within a `<record>` sets the document field name and value. You should ensure the `name` attribute
of each `<data>` element exactly matches the mapping property of the Elasticsearch index you created.

#### Steps

1. Create a pipeline inheriting from the built-in `Indexing` template.
1. Modify the `xsltFilter` pipeline stage to output the correct `<records>` XML (see the [Quick-Start Guide]({{< relref "../../quick-start-guide/indexing.md" >}}).
1. Delete the default `indexingFilter` and in its place, create an `ElasticIndexingFilter` (see screenshot below).
1. Review and set the following properties:
    1. `batchSize` (default: `10,000`). Number of documents to send in a single request to the Elasticsearch {{< external-link "Bulk API" "https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html" >}}.
        Should usually be set to `1,000` or more.
        The higher the number, the more memory is required by both Stroom and Elasticsearch when sending or receiving the request.
    1. `index` (required). Set this to the target Elasticsearch index in the Stroom _Explorer Tree_.
    1. `refreshAfterEachBatch` (default: `false`). Refreshes the Elasticsearch index after each batch has finished processing.
       This makes any documents ingested in the batch available for searching.
       Unless search results are needed in near-real-time, it is recommended this be set to `false` and the index refresh interval be set to an appropriate value.
       See {{< external-link "this document" "https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-indexing-speed.html#_unset_or_increase_the_refresh_interval" >}} for guidance on optimising indexing performance.

{{< screenshot "HOWTOs/Elastic-Add-Pipeline-Filter.png" >}}Elasticsearch indexing filter{{< /screenshot >}}

### Creating and activating a stream processor

Follow the steps as in [this guide]({{< relref "../../quick-start-guide/indexing.md" >}}).

### Checking data has been indexed

Query Elasticsearch, checking the fields you expect are there, and of the correct data type:

The following query displays five results:
{{< command-line "user" "localhost" >}}
curl -X GET "http://localhost:9200/stroom_test/_search?size=5" 
{{</ command-line >}}

You can also get an exact document count, to ensure this matches the number of events you are expecting:

{{< command-line "user" "localhost" >}}
curl -X GET "http://localhost:9200/stroom_test/_count"
{{</ command-line >}}

For more information, see the Elasticsearch {{< external-link "Search API documentation" "https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html" >}}.


### Reindexing data

By default, the original document values are stored in an Elasticsearch index and may be used later on to re-index data (such as when a change is made to field mappings).
This is done via the {{< external-link "Reindex API" "https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html" >}}.
Provided these values have not changed, it would likely be more efficient to use this API to perform a re-index,
instead of processing data from scratch using a Stroom stream processor.

On the other hand, if the content of documents being output to Elasticsearch has changed, the Elasticsearch index will need to be re-created
and the stream re-processed. Examples of where this would be required include:

1. A new field is added to the indexing filter, which previously didn't exist. That field needs to be searchable for all historical
events.
1. A field is renamed
1. A field data type is changed

If a field is omitted from the indexing translation, there is no need for a re-index, unless you wish to reclaim the space
occupied by that field.


#### Reindexing using a pipeline processor

1. Delete the index. While it is possible to {{< external-link "delete by query" "https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html" >}}, it is more efficient to drop the index.
   Additionally, deleting by query doesn't actually remove data from disk, until segments are merged.
   {{< command-line "user" "localhost" >}}
   curl -X DELETE "http://localhost:9200/stroom_test"
   {{</ command-line >}}
1. Re-create the index (as shown earlier)
1. Create a new pipeline processor to index the documents


## Searching

Once indexed in Elasticsearch, you can search either using the _Stroom Dashboard_ user interface, or directly against
the Elasticsearch cluster.

The advantage of using Stroom to search is that it allows access to the raw source data (i.e. it is not limited to what's stored in the index).
It can also use _extraction pipelines_ to enrich search results for export in a table.

Elasticsearch on the other hand, provides a rich{{< external-link "Search REST API" "https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html" >}} 
with powerful [aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html) that can be
used to generate reports and discover patterns and anomalies. It can also be readily queried using third-party tools.


### Stroom

See the [Dashboard]({{< relref "../../quick-start-guide/dashboard.md" >}}) page in the Quick-Start Guide.

Instead of selecting a Lucene index, set the target _data source_ to the desired Elasticsearch index in the Stroom _Explorer Tree_.

Once the target _data source_ has been set, the Dashboard can be used as with a Lucene or Solr index _data source_.


### Elasticsearch

Elasticsearch queries can be performed directly against the cluster using the {{< external-link "Search API" "https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html" >}}.

Alternatively, there are tools that make search and discovery easier and more intuitive, like {{< external-link "Kibana" "https://www.elastic.co/kibana" >}}.


## Security

It is important to note that Elasticsearch data is not encrypted at rest, unless this feature is enabled and the relevant {{< external-link "licensing tier" "https://www.elastic.co/subscriptions" >}} is purchased.
Therefore, appropriate measures should be taken to control access to Elasticsearch user data at the file level.

For production clusters, the Elasticsearch {{< external-link "security guidelines" "https://www.elastic.co/guide/en/elasticsearch/reference/current/elasticsearch-security.html" >}} should be followed, in order to control access and ensure requests are audited.

You might want to consider implementing {{< external-link "role-based access control" "https://www.elastic.co/guide/en/kibana/current/development-security.html" >}} 
to prevent unauthorised users of the native Elasticsearch API or tools like Kibana, from creating, modifying or deleting data within sensitive indices.

---
title: Elasticsearch Reference阅读笔记
date: 2017-02-03 11:28:01
tags: [Elasticsearch, 一起读文档]
categories: Elasticsearch
link_title: elasticsearch_reference_notes
toc_number: false
---
花了几天把[Elasticsearch的官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/index.html)读了一遍，随手记一些关键的笔记。

<!-- more -->
# 1. Getting Started
## 1.1 Elasticsearch
Elasticsearch is a highly scalable open-source full-text search and analytics engine. It allows you to store, search, and analyze big volumes of data quickly and in near real time. It is generally used as the underlying engine/technology that powers applications that have complex search features and requirements.

## 1.2 Sharding is important for two primary reasons:
It allows you to horizontally split/scale your content volume
It allows you to distribute and parallelize operations across shards (potentially on multiple nodes) thus increasing performance/throughput 

## 1.3 Replication is important for two primary reasons:
It provides high availability in case a shard/node fails. For this reason, it is important to note that a replica shard is never allocated on the same node as the original/primary shard that it was copied from.
It allows you to scale out your search volume/throughput since searches can be executed on all replicas in parallel.

## 1.4 Cluster Health
    curl 'localhost:9200/_cat/health?v'

    curl 'localhost:9200/_cat/nodes?v'

## 1.5 List All Indices
    curl 'localhost:9200/_cat/indices?v'

## 1.6 Create an Index
    curl -XPUT 'localhost:9200/customer?pretty'

## 1.7 Index and Query a Document
    curl -XPUT 'localhost:9200/customer/external/1?pretty' -d '
    {
      "name": "John Doe"
    }'

    curl -XGET 'localhost:9200/customer/external/1?pretty'

## 1.8 Delete an Index
    curl -XDELETE 'localhost:9200/customer?pretty'

## 1.9 Updating Documents
Note though that Elasticsearch does not actually do in-place updates under the hood. Whenever we do an update, Elasticsearch deletes the old document and then indexes a new document with the update applied to it in one shot.

## 1.10 Deleting Documents
    curl -XDELETE 'localhost:9200/customer/external/2?pretty'

## 1.11 Batch Processing
In addition to being able to index, update, and delete individual documents, Elasticsearch also provides the ability to perform any of the above operations in batches using the _bulk API. This functionality is important in that it provides a very efficient mechanism to do multiple operations as fast as possible with as little network roundtrips as possible.

The bulk API executes all the actions sequentially and in order. If a single action fails for whatever reason, it will continue to process the remainder of the actions after it. When the bulk API returns, it will provide a status for each action (in the same order it was sent in) so that you can check if a specific action failed or not.

## 1.12 The Search API
It is important to understand that once you get your search results back, Elasticsearch is completely done with the request and does not maintain any kind of server-side resources or open cursors into your results. This is in stark contrast to many other platforms such as SQL wherein you may initially get a partial subset of your query results up-front and then you have to continuously go back to the server if you want to fetch (or page through) the rest of the results using some kind of stateful server-side cursor.

## 1.13 Executing Filters
In the previous section, we skipped over a little detail called the document score (_score field in the search results). The score is a numeric value that is a relative measure of how well the document matches the search query that we specified. The higher the score, the more relevant the document is, the lower the score, the less relevant the document is.

But queries do not always need to produce scores, in particular when they are only used for "filtering" the document set. Elasticsearch detects these situations and automatically optimizes query execution in order not to compute useless scores.

## 1.14 Executing Aggregations
Aggregations provide the ability to group and extract statistics from your data. The easiest way to think about aggregations is by roughly equating it to the SQL GROUP BY and the SQL aggregate functions. In Elasticsearch, you have the ability to execute searches returning hits and at the same time return aggregated results separate from the hits all in one response. This is very powerful and efficient in the sense that you can run queries and multiple aggregations and get the results back of both (or either) operations in one shot avoiding network roundtrips using a concise and simplified API.

# 2. Setup
## 2.1 Environment Variables
Most times it is better to leave the default JAVA_OPTS as they are, and use the ES_JAVA_OPTS environment variable in order to set / change JVM settings or arguments.

The ES_HEAP_SIZE environment variable allows to set the heap memory that will be allocated to elasticsearch java process. It will allocate the same value to both min and max values, though those can be set explicitly (not recommended) by setting ES_MIN_MEM (defaults to 256m), and ES_MAX_MEM (defaults to 1g).

It is recommended to set the min and max memory to the same value, and enable mlockall.

## 2.2 File Descriptors
Make sure to increase the number of open files descriptors on the machine (or for the user running elasticsearch). Setting it to 32k or even 64k is recommended.

In order to test how many open files the process can open, start it with -Des.max-open-files set to true. This will print the number of open files the process can open on startup.

Alternatively, you can retrieve the max_file_descriptors for each node using the Nodes Info API, with:

    curl localhost:9200/_nodes/stats/process?pretty

## 2.3 Virtual memory
Elasticsearch uses a hybrid mmapfs / niofs directory by default to store its indices. The default operating system limits on mmap counts is likely to be too low, which may result in out of memory exceptions. On Linux, you can increase the limits by running the following command as root:

    sysctl -w vm.max_map_count=262144

To set this value permanently, update the vm.max_map_count setting in /etc/sysctl.conf.

## 2.4 Memory Settings
Most operating systems try to use as much memory as possible for file system caches and eagerly swap out unused application memory, possibly resulting in the elasticsearch process being swapped. Swapping is very bad for performance and for node stability, so it should be avoided at all costs.

There are three options: Disable swap、Configure swappiness、mlockall

## 2.5 Elasticsearch Settings
elasticsearch configuration files can be found under ES_HOME/config folder. The folder comes with two files, the elasticsearch.yml for configuring Elasticsearch different modules, and logging.yml for configuring the Elasticsearch logging.

The configuration format is YAML.

## 2.6 Directory Layout
zip and tar.gz
|Type |	Description| Location |
|:--- |:-----------|:---------|
home  |Home of elasticsearch installation|{extract.path}
bin   |Binary scripts including elasticsearch to start a node|{extract.path}/bin
conf  |Configuration files elasticsearch.yml and logging.yml|{extract.path}/config
data  |The location of the data files of each index / shard allocated on the node|{extract.path}/data
logs  |Log files location|{extract.path}/logs
plugins|Plugin files location. Each plugin will be contained in a subdirectory|{extract.path}/plugins
repo  |Shared file system repository locations.|Not configured
script|Location of script files.|{extract.path}/config/scripts

# 3 Breaking changes (skipped)

# 4 API Conventions (skipped)

# 5 Document APIs
## 5.1 Index API
The index API adds or updates a typed JSON document in a specific index, making it searchable. The following example inserts the JSON document into the "twitter" index, under a type called "tweet" with an id of 1:

    curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
        "user" : "kimchy",
        "post_date" : "2009-11-15T14:12:12",
        "message" : "trying out Elasticsearch"
    }'

## 5.2 Automatic Index Creation
The index operation automatically creates an index if it has not been created before (check out the create index API for manually creating an index), and also automatically creates a dynamic type mapping for the specific type if one has not yet been created (check out the put mapping API for manually creating a type mapping).

## 5.3 Versioning
Each indexed document is given a version number. The associated version number is returned as part of the response to the index API request. The index API optionally allows for optimistic concurrency control when the version parameter is specified. This will control the version of the document the operation is intended to be executed against. A good example of a use case for versioning is performing a transactional read-then-update. Specifying a version from the document initially read ensures no changes have happened in the meantime (when reading in order to update, it is recommended to set preference to _primary). For example:

    curl -XPUT 'localhost:9200/twitter/tweet/1?version=2' -d '{
        "message" : "elasticsearch now has versioning support, double cool!"
    }'

## 5.4 Operation Type
The index operation also accepts an op_type that can be used to force a create operation, allowing for "put-if-absent" behavior. When create is used, the index operation will fail if a document by that id already exists in the index.

Here is an example of using the op_type parameter:

    curl -XPUT 'http://localhost:9200/twitter/tweet/1?op_type=create' -d '{
        "user" : "kimchy",
        "post_date" : "2009-11-15T14:12:12",
        "message" : "trying out Elasticsearch"
    }'

## 5.5 Routing
By default, shard placement — or routing — is controlled by using a hash of the document’s id value. For more explicit control, the value fed into the hash function used by the router can be directly specified on a per-operation basis using the routing parameter. For example:

    curl -XPOST 'http://localhost:9200/twitter/tweet?routing=kimchy' -d '{
        "user" : "kimchy",
        "post_date" : "2009-11-15T14:12:12",
        "message" : "trying out Elasticsearch"
    }'

## 5.6 Parents & Children （******适合什么场景?）
A child document can be indexed by specifying its parent when indexing. For example:

    curl -XPUT localhost:9200/blogs/blog_tag/1122?parent=1111 -d '{
        "tag" : "something"
    }'

When indexing a child document, the routing value is automatically set to be the same as its parent, unless the routing value is explicitly specified using the routing parameter.

## 5.7 Distributed
The index operation is directed to the primary shard based on its route (see the Routing section above) and performed on the actual node containing this shard. After the primary shard completes the operation, if needed, the update is distributed to applicable replicas.

## 5.8 Write Consistency
To prevent writes from taking place on the "wrong" side of a network partition, by default, index operations only succeed if a quorum (>replicas/2+1) of active shards are available. 

## 5.9 Write Consistency (******如果index设置为1个主分片，两个复制分片。当两个复制分片都不可用的时候index在主分片是否成功？复制分片可用了是否能复制？)
To prevent writes from taking place on the "wrong" side of a network partition, by default, index operations only succeed if a quorum (>replicas/2+1) of active shards are available. 

The index operation only returns after all active shards within the replication group have indexed the document (sync replication).

## 5.10 Refresh
To refresh the shard (not the whole index) immediately after the operation occurs, so that the document appears in search results immediately, the refresh parameter can be set to true. Setting this option to true should ONLY be done after careful thought and verification that it does not lead to poor performance, both from an indexing and a search standpoint. Note, getting a document using the get API is completely realtime and doesn’t require a refresh.

## 5.11 Get API
The get API allows to get a typed JSON document from the index based on its id. The following example gets a JSON document from an index called twitter, under a type called tweet, with id valued 1:

    curl -XGET 'http://localhost:9200/twitter/tweet/1'

## 5.12 Preference
Controls a preference of which shard replicas to execute the get request on. By default, the operation is randomized between the shard replicas.

The preference can be set to:

- _primary: The operation will go and be executed only on the primary shards.
- _local: The operation will prefer to be executed on a local allocated shard if possible.
- Custom (string) value: A custom value will be used to guarantee that the same shards will be used for the same custom value. This can help with "jumping values" when hitting different shards in different refresh states. A sample value can be something like the web session id, or the user name.

## 5.13 Delete API
The delete API allows to delete a typed JSON document from a specific index based on its id. The following example deletes the JSON document from an index called twitter, under a type called tweet, with id valued 1:

    curl -XDELETE 'http://localhost:9200/twitter/tweet/1'

The delete operation gets hashed into a specific shard id. It then gets redirected into the primary shard within that id group, and replicated (if needed) to shard replicas within that id group.


## 5.14 Update API
The update API allows to update a document based on a script provided. The operation gets the document (collocated with the shard) from the index, runs the script (with optional script language and parameters), and index back the result (also allows to delete, or ignore the operation). It uses versioning to make sure no updates have happened during the "get" and "reindex".

Note, this operation still means full reindex of the document, it just removes some network roundtrips and reduces chances of version conflicts between the get and the index. The _source field needs to be enabled for this feature to work.

## 5.15 Update By Query API (new and should still be considered experimental)
The simplest usage of _update_by_query just performs an update on every document in the index without changing the source. This is useful to pick up a new property or some other online mapping change. Here is the API:

    curl -XPOST 'localhost:9200/twitter/_update_by_query?conflicts=proceed'

All update and query failures cause the _update_by_query to abort and are returned in the failures of the response. The updates that have been performed still stick. In other words, the process is not rolled back, only aborted.

## 5.16 Multi Get API
Multi GET API allows to get multiple documents based on an index, type (optional) and id (and possibly routing). The response includes a docs array with all the fetched documents, each element similar in structure to a document provided by the get API. Here is an example:

    curl 'localhost:9200/_mget' -d '{
        "docs" : [
            {
                "_index" : "test",
                "_type" : "type",
                "_id" : "1"
            },
            {
                "_index" : "test",
                "_type" : "type",
                "_id" : "2"
            }
        ]
    }'

## 5.17 Bulk API
The bulk API makes it possible to perform many index/delete operations in a single API call. This can greatly increase the indexing speed.

The REST API endpoint is /_bulk, and it expects the following JSON structure:

    action_and_meta_data\n
    optional_source\n
    action_and_meta_data\n
    optional_source\n
    ....
    action_and_meta_data\n
    optional_source\n

NOTE: the final line of data must end with a newline character \n.

The possible actions are index, create, delete and update. index and create expect a source on the next line, and have the same semantics as the op_type parameter to the standard index API (i.e. create will fail if a document with the same index and type exists already, whereas index will add or replace a document as necessary). delete does not expect a source on the following line, and has the same semantics as the standard delete API. update expects that the partial doc, upsert and script and its options are specified on the next line.

Here is an example of a correct sequence of bulk commands:

    { "index" : { "_index" : "test", "_type" : "type1", "_id" : "1" } }
    { "field1" : "value1" }
    { "delete" : { "_index" : "test", "_type" : "type1", "_id" : "2" } }
    { "create" : { "_index" : "test", "_type" : "type1", "_id" : "3" } }
    { "field1" : "value3" }
    { "update" : {"_id" : "1", "_type" : "type1", "_index" : "index1"} }
    { "doc" : {"field2" : "value2"} }

The endpoints are /_bulk, /{index}/_bulk, and {index}/{type}/_bulk. When the index or the index/type are provided, they will be used by default on bulk items that don’t provide them explicitly.

A note on the format. The idea here is to make processing of this as fast as possible. As some of the actions will be redirected to other shards on other nodes, only action_meta_data is parsed on the receiving node side.

## 5.18 Reindex API (new and should still be considered experimental)

The most basic form of _reindex just copies documents from one index to another. This will copy documents from the twitter index into the new_twitter index:

    POST /_reindex
    {
      "source": {
        "index": "twitter"
      },
      "dest": {
        "index": "new_twitter"
      }
    }

You can limit the documents by adding a type to the source or by adding a query. This will only copy tweet's made by kimchy into new_twitter:

    POST /_reindex
    {
      "source": {
        "index": "twitter",
        "type": "tweet",
        "query": {
          "term": {
            "user": "kimchy"
          }
        }
      },
      "dest": {
        "index": "new_twitter"
      }
    }
    
## 5.19 Term Vectors
Returns information and statistics on terms in the fields of a particular document. The document could be stored in the index or artificially provided by the user. Term vectors are realtime by default, not near realtime. This can be changed by setting realtime parameter to false.

    curl -XGET 'http://localhost:9200/twitter/tweet/1/_termvectors?pretty=true'

Three types of values can be requested: term information, term statistics and field statistics. By default, all term information and field statistics are returned for all fields but no term statistics.

Term information
- term frequency in the field (always returned)
- term positions (positions : true)
- start and end offsets (offsets : true)
- term payloads (payloads : true), as base64 encoded bytes

Term statistics
- total term frequency (how often a term occurs in all documents)
- document frequency (the number of documents containing the current term)

> Setting term_statistics to true (default is false) will return term statistics. By default these values are not returned since term statistics can have a serious performance impact.

Field statistics
- document count (how many documents contain this field)
- sum of document frequencies (the sum of document frequencies for all terms in this field)
- sum of total term frequencies (the sum of total term frequencies of each term in this field)

The term and field statistics are not accurate. Deleted documents are not taken into account. The information is only retrieved for the shard the requested document resides in, unless dfs is set to true. The term and field statistics are therefore only useful as relative measures whereas the absolute numbers have no meaning in this context. By default, when requesting term vectors of artificial documents, a shard to get the statistics from is randomly selected.

See more examples: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/docs-termvectors.html#_behaviour

# 6 Search APIs
## 6.1 Search
The search API allows you to execute a search query and get back search hits that match the query. The query can either be provided using a simple query string as a parameter, or using a request body.

All search APIs can be applied across multiple types within an index, and across multiple indices with support for the multi index syntax. 

## 6.2 URI Search
A search request can be executed purely using a URI by providing request parameters. Not all search options are exposed when executing a search using this mode, but it can be handy for quick "curl tests". Here is an example:

    curl -XGET 'http://localhost:9200/twitter/tweet/_search?q=user:kimchy'

## 6.3 Request Body Search
The search request can be executed with a search DSL, which includes the Query DSL, within its body. Here is an example:

    curl -XGET 'http://localhost:9200/twitter/tweet/_search' -d '{
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }'

## 6.4 Query
The query element within the search request body allows to define a query using the [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/query-dsl.html).

    {
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }

## 6.5 From / Size
Pagination of results can be done by using the from and size parameters.

    {
        "from" : 0, "size" : 10,
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }

Note that from + size can not be more than the index.max_result_window index setting which defaults to 10,000. See the [Scroll](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/search-request-scroll.html) API for more efficient ways to do deep scrolling.

## 6.6 Sort (******多个字段的排序规则是怎么样的？)
Allows to add one or more sort on specific fields. Each sort can be reversed as well. The sort is defined on a per field level, with special field name for _score to sort by score, and _doc to sort by index order.

    {
        "sort" : [
            { "post_date" : {"order" : "asc"}},
            "user",
            { "name" : "desc" },
            { "age" : "desc" },
            "_score"
        ],
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }

The sort values for each document returned are also returned as part of the response.
The order option can have the following values:

- asc: Sort in ascending order
- desc: Sort in descending order

The order defaults to desc when sorting on the _score, and defaults to asc when sorting on anything else.

## 6.7 Sort mode option
Elasticsearch supports sorting by array or multi-valued fields. The mode option controls what array value is picked for sorting the document it belongs to. The mode option can have the following values:

- min: Pick the lowest value.
- max: Pick the highest value.
- sum: Use the sum of all values as sort value. Only applicable for number based array fields.
- avg: Use the average of all values as sort value. Only applicable for number based array fields.
- median: Use the median of all values as sort value. Only applicable for number based array fields.

## 6.8 Missing Values
The missing parameter specifies how docs which are missing the field should be treated: The missing value can be set to _last, _first, or a custom value (that will be used for missing docs as the sort value). For example:

    {
        "sort" : [
            { "price" : {"missing" : "_last"} },
        ],
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }

## 6.9 Script Based Sorting
Allow to sort based on custom scripts, here is an example:

    {
        "query" : {
            ....
        },
        "sort" : {
            "_script" : {
                "type" : "number",
                "script" : {
                    "inline": "doc['field_name'].value * factor",
                    "params" : {
                        "factor" : 1.1
                    }
                },
                "order" : "asc"
            }
        }
    }
    
## 6.10 Memory Considerations
When sorting, the relevant sorted field values are loaded into memory. This means that per shard, there should be enough memory to contain them. For string based types, the field sorted on should not be analyzed / tokenized. For numeric types, if possible, it is recommended to explicitly set the type to narrower types (like short, integer and float).

## 6.11 Source filtering
Allows to control how the _source field is returned with every hit.

By default operations return the contents of the _source field unless you have used the fields parameter or if the _source field is disabled.

- To disable _source retrieval set to false.
- The _source also accepts one or more wildcard patterns to control what parts of the _source should be returned.
- Finally, for complete control, you can specify both include and exclude patterns.

## 6.12 Fields
> The fields parameter is about fields that are explicitly marked as stored in the mapping, which is off by default and generally not recommended. Use source filtering instead to select subsets of the original source document to be returned.

## 6.13 Script Fields
Allows to return a script evaluation (based on different fields) for each hit, for example:

    {
        "query" : {
            ...
        },
        "script_fields" : {
            "test1" : {
                "script" : "_source.obj1.obj2"
            },
            "test2" : {
                "script" : {
                    "inline": "doc['my_field_name'].value * factor",
                    "params" : {
                        "factor"  : 2.0
                    }
                }
            }
        }
    }

Note the _source keyword here to navigate the json-like model.

It’s important to understand the difference between doc['my_field'].value and _source.my_field. The first, using the doc keyword, will cause the terms for that field to be loaded to memory (cached), which will result in faster execution, but more memory consumption. Also, the doc[...] notation only allows for simple valued fields (can’t return a json object from it) and make sense only on non-analyzed or single term based fields.

The _source on the other hand causes the source to be loaded, parsed, and then only the relevant part of the json is returned.

## 6.14 Field Data Fields (******需要了解stored和fielddata的概念)
Allows to return the field data representation of a field for each hit, for example:

    {
        "query" : {
            ...
        },
        "fielddata_fields" : ["test1", "test2"]
    }
    
Field data fields can work on fields that are not stored.

It’s important to understand that using the fielddata_fields parameter will cause the terms for that field to be loaded to memory (cached), which will result in more memory consumption.

## 6.15 Post filter
The post_filter is applied to the search hits at the very end of a search request, after aggregations have already been calculated.

    {
      "query": {
        "bool": {
          "filter": {
            "term": {
              "brandName": "vans"
            }
          }
        }
      },
      "aggs": {
        "colors": {
          "terms": {
            "field": "colorNames"
          }
        },
        "color_red": {
          "filter": {
            "term": {
              "colorNames": "红色"
            }
          },
          "aggs": {
            "smallSorts": {
              "terms": {
                "field": "smallSort"
              }
            }
          }
        }
      },
      "post_filter": {
        "term": {
          "colorNames": "红色"
        }
      }
    }

	
- The main query now finds all products by vans, regardless of color.
- The colors agg returns popular colors by vans.
- The color_red agg limits the small sort sub-aggregation to red vans products.
- Finally, the post_filter removes colors other than red from the search hits.

## 6.16 Highlighting
Allows to highlight search results on one or more fields. The implementation uses either the lucene highlighter, fast-vector-highlighter or postings-highlighter. The following is an example of the search request body:

    {
        "query" : {...},
        "highlight" : {
            "fields" : {
                "content" : {}
            }
        }
    }

### 6.16.1 Plain highlighter
The default choice of highlighter is of type plain and uses the Lucene highlighter. It tries hard to reflect the query matching logic in terms of understanding word importance and any word positioning criteria in phrase queries.

### 6.16.2 Postings highlighter
If index_options is set to offsets in the mapping the postings highlighter will be used instead of the plain highlighter. The postings highlighter:

- Is faster since it doesn’t require to reanalyze the text to be highlighted: the larger the documents the better the performance gain should be
- Requires less disk space than term_vectors, needed for the fast vector highlighter
- Breaks the text into sentences and highlights them. Plays really well with natural languages, not as well with - fields containing for instance html markup
- Treats the document as the whole corpus, and scores individual sentences as if they were documents in this corpus, using the BM25 algorithm

### 6.16.3 Fast vector highlighter
If term_vector information is provided by setting term_vector to with_positions_offsets in the mapping then the fast vector highlighter will be used instead of the plain highlighter. The fast vector highlighter:

- Is faster especially for large fields (> 1MB)
- Can be customized with boundary_chars, boundary_max_scan, and fragment_offset (see below)
- Requires setting term_vector to with_positions_offsets which increases the size of the index
- Can combine matches from multiple fields into one result. See matched_fields
- Can assign different weights to matches at different positions allowing for things like phrase matches being - sorted above term matches when highlighting a Boosting Query that boosts phrase matches over term matches

## 6.17 Rescoring
Rescoring can help to improve precision by reordering just the top (eg 100 - 500) documents returned by the query and post_filter phases, using a secondary (usually more costly) algorithm, instead of applying the costly algorithm to all documents in the index.

A rescore request is executed on each shard before it returns its results to be sorted by the node handling the overall search request.

Currently the rescore API has only one implementation: the query rescorer, which uses a query to tweak the scoring. In the future, alternative rescorers may be made available, for example, a pair-wise rescorer.

By default the scores from the original query and the rescore query are combined linearly to produce the final _score for each document. The relative importance of the original query and of the rescore query can be controlled with the query_weight and rescore_query_weight respectively. Both default to 1.

    {
      "query": {
        "match": {
          "productName.productName_ansj": {
            "operator": "or",
            "query": "连帽 套装",
            "type": "boolean"
          }
        }
      },
      "_source": [
        "productName"
      ],
      "rescore": {
        "window_size": 50,
        "query": {
          "rescore_query": {
            "match": {
              "productName.productName_ansj": {
                "query": "连帽 套装",
                "type": "phrase",
                "slop": 2
              }
            }
          },
          "query_weight": 0.7,
          "rescore_query_weight": 1.2
        }
      }
    }

Score Mode
- total:Add the original score and the rescore query score. The default.
- multiply: Multiply the original score by the rescore query score. Useful for function query rescores.
- avg: Average the original score and the rescore query score.
- max: Take the max of original score and the rescore query score.
- min: Take the min of the original score and the rescore query score.

## 6.18 Search Type
There are different execution paths that can be done when executing a distributed search. The distributed search operation needs to be scattered to all the relevant shards and then all the results are gathered back. When doing scatter/gather type execution, there are several ways to do that, specifically with search engines.

One of the questions when executing a distributed search is how many results to retrieve from each shard. For example, if we have 10 shards, the 1st shard might hold the most relevant results from 0 till 10, with other shards results ranking below it. For this reason, when executing a request, we will need to get results from 0 till 10 from all shards, sort them, and then return the results if we want to ensure correct results.

Another question, which relates to the search engine, is the fact that each shard stands on its own. When a query is executed on a specific shard, it does not take into account term frequencies and other search engine information from the other shards. If we want to support accurate ranking, we would need to first gather the term frequencies from all shards to calculate global term frequencies, then execute the query on each shard using these global frequencies.

Also, because of the need to sort the results, getting back a large document set, or even scrolling it, while maintaining the correct sorting behavior can be a very expensive operation. For large result set scrolling, it is best to sort by _doc if the order in which documents are returned is not important.

Elasticsearch is very flexible and allows to control the type of search to execute on a per search request basis. The type can be configured by setting the search_type parameter in the query string. The types are:

### 6.18.1 Query Then Fetch(query_then_fetch)
The request is processed in two phases. In the first phase, the query is forwarded to all involved shards. Each shard executes the search and generates a sorted list of results, local to that shard. Each shard returns just enough information to the coordinating node to allow it merge and re-sort the shard level results into a globally sorted set of results, of maximum length size.

During the second phase, the coordinating node requests the document content (and highlighted snippets, if any) from only the relevant shards.

Note: This is the default setting, if you do not specify a search_type in your request.

### 6.18.2 Dfs, Query Then Fetch(dfs_query_then_fetch)
Same as "Query Then Fetch", except for an initial scatter phase which goes and computes the distributed term frequencies for more accurate scoring.

### 6.18.3 Count (Deprecated in 2.0.0-beta1)
### 6.18.4 Scan (Deprecated in 2.1.0)

## 6.19 Scroll
While a search request returns a single “page” of results, the scroll API can be used to retrieve large numbers of results (or even all results) from a single search request, in much the same way as you would use a cursor on a traditional database.

Scrolling is not intended for real time user requests, but rather for processing large amounts of data, e.g. in order to reindex the contents of one index into a new index with a different configuration.

## 6.20 Preference
Controls a preference of which shard replicas to execute the search request on. By default, the operation is randomized between the shard replicas.

The preference is a query string parameter which can be set to:

- _primary: The operation will go and be executed only on the primary shards.
- _primary_first: The operation will go and be executed on the primary shard, and if not available (failover), will execute on other shards.
- _replica: The operation will go and be executed only on a replica shard.
- _replica_first: The operation will go and be executed only on a replica shard, and if not available (failover), will execute on other shards.
- _local: The operation will prefer to be executed on a local allocated shard if possible.
- _only_node:xyz: Restricts the search to execute only on a node with the provided node id (xyz in this case).
- _prefer_node:xyz: Prefers execution on the node with the provided node id (xyz in this case) if applicable.
- _shards:2,3: Restricts the operation to the specified shards. (2 and 3 in this case). This preference can be combined with other preferences but it has to appear first: _shards:2,3;_primary
- _only_nodes: Restricts the operation to nodes specified in node specification https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html
- Custom (string) value: A custom value will be used to guarantee that the same shards will be used for the same custom value. This can help with "jumping values" when hitting different shards in different refresh states. A sample value can be something like the web session id, or the user name.

## 6.21 Explain
Enables explanation for each hit on how its score was computed.

    {
        "explain": true,
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }


## 6.22 Version
Returns a version for each search hit.

    {
        "version": true,
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }

## 6.23 Index Boost
Allows to configure different boost level per index when searching across more than one indices. This is very handy when hits coming from one index matter more than hits coming from another index (think social graph where each user has an index).

    {
        "indices_boost" : {
            "index1" : 1.4,
            "index2" : 1.3
        }
    }

## 6.24 Inner hits (******Skipped: 和parent/child有关系，后续一起学习)

## 6.25 Search Template
The /_search/template endpoint allows to use the mustache language to pre render search requests, before they are executed and fill existing templates with template parameters.

    GET /_search/template
    {
        "inline" : {
          "query": { "match" : { "{{my_field}}" : "{{my_value}}" } },
          "size" : "{{my_size}}"
        },
        "params" : {
            "my_field" : "foo",
            "my_value" : "bar",
            "my_size" : 5
        }
    }

## 6.26 Search Shards API
The search shards api returns the indices and shards that a search request would be executed against. This can give useful feedback for working out issues or planning optimizations with routing and shard preferences.

    curl -XGET 'localhost:9200/twitter/_search_shards'
    
## 6.27 Suggesters (Skipped)
The suggest feature suggests similar looking terms based on a provided text by using a suggester. Parts of the suggest feature are still under development.

## 6.28 Count API
The count API allows to easily execute a query and get the number of matches for that query. It can be executed across one or more indices and across one or more types. The query can either be provided using a simple query string as a parameter, or using the Query DSL defined within the request body. Here is an example:

    curl -XGET 'http://localhost:9200/twitter/tweet/_count?q=user:kimchy'

    curl -XGET 'http://localhost:9200/twitter/tweet/_count' -d '
    {
        "query" : {
            "term" : { "user" : "kimchy" }
        }
    }'
    
## 6.29 Validate API
The validate API allows a user to validate a potentially expensive query without executing it. The following example shows how it can be used:

    curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
        "user" : "kimchy",
        "post_date" : "2009-11-15T14:12:12",
        "message" : "trying out Elasticsearch"
    }'
When the query is valid, the response contains valid:true:

    curl -XGET 'http://localhost:9200/twitter/_validate/query?q=user:foo'
    
    {"valid":true,"_shards":{"total":1,"successful":1,"failed":0}}
    
## 6.30 Explain API
The explain api computes a score explanation for a query and a specific document. This can give useful feedback whether a document matches or didn’t match a specific query.

    curl -XGET 'localhost:9200/twitter/tweet/1/_explain' -d '{
          "query" : {
            "term" : { "message" : "search" }
          }
    }'

## 6.30 Profile API (experimental and may be changed or removed)
The Profile API provides detailed timing information about the execution of individual components in a query. It gives the user insight into how queries are executed at a low level so that the user can understand why certain queries are slow, and take steps to improve their slow queries.

The output from the Profile API is very verbose, especially for complicated queries executed across many shards. Pretty-printing the response is recommended to help understand the output.

    curl -XGET 'localhost:9200/_search' -d '{
      "profile": true,
      "query" : {
        "match" : { "message" : "search test" }
      }
    }

## 6.31 Field stats API (experimental and may be changed or removed)
The field stats api allows one to find statistical properties of a field without executing a search, but looking up measurements that are natively available in the Lucene index. This can be useful to explore a dataset which you don’t know much about. For example, this allows creating a histogram aggregation with meaningful intervals based on the min/max range of values.

The field stats api by defaults executes on all indices, but can execute on specific indices too.

All indices:

    curl -XGET "http://localhost:9200/_field_stats?fields=rating"
Specific indices:

    curl -XGET "http://localhost:9200/index1,index2/_field_stats?fields=rating"

# 7 Aggregations
## 7.1 Aggregations
The aggregations framework helps provide aggregated data based on a search query. It is based on simple building blocks called aggregations, that can be composed in order to build complex summaries of the data.

An aggregation can be seen as a unit-of-work that builds analytic information over a set of documents. The context of the execution defines what this document set is (e.g. a top-level aggregation executes within the context of the executed query/filters of the search request).

There are many different types of aggregations, each with its own purpose and output. To better understand these types, it is often easier to break them into three main families:

- Bucketing: A family of aggregations that build buckets, where each bucket is associated with a key and a document criterion. When the aggregation is executed, all the buckets criteria are evaluated on every document in the context and when a criterion matches, the document is considered to "fall in" the relevant bucket. By the end of the aggregation process, we’ll end up with a list of buckets - each one with a set of documents that "belong" to it.
- Metric: Aggregations that keep track and compute metrics over a set of documents.
- Pipeline: Aggregations that aggregate the output of other aggregations and their associated metrics

The interesting part comes next. Since each bucket effectively defines a document set (all documents belonging to the bucket), one can potentially associate aggregations on the bucket level, and those will execute within the context of that bucket. This is where the real power of aggregations kicks in: aggregations can be nested!

## 7.2 Structuring Aggregations
The following snippet captures the basic structure of aggregations:

    "aggregations" : {
        "<aggregation_name>" : {
            "<aggregation_type>" : {
                <aggregation_body>
            }
            [,"meta" : {  [<meta_data_body>] } ]?
            [,"aggregations" : { [<sub_aggregation>]+ } ]?
        }
        [,"<aggregation_name_2>" : { ... } ]*
    }

## 7.3 Metrics Aggregations
The aggregations in this family compute metrics based on values extracted in one way or another from the documents that are being aggregated. The values are typically extracted from the fields of the document (using the field data), but can also be generated using scripts.

Numeric metrics aggregations are a special type of metrics aggregation which output numeric values. Some aggregations output a single numeric metric (e.g. avg) and are called single-value numeric metrics aggregation, others generate multiple metrics (e.g. stats) and are called multi-value numeric metrics aggregation. The distinction between single-value and multi-value numeric metrics aggregations plays a role when these aggregations serve as direct sub-aggregations of some bucket aggregations (some bucket aggregations enable you to sort the returned buckets based on the numeric metrics in each bucket).

### 7.3.1 Avg Aggregation
A single-value metrics aggregation that computes the average of numeric values that are extracted from the aggregated documents. These values can be extracted either from specific numeric fields in the documents, or be generated by a provided script.

    {
      "size": 0,
      "query": {
        "match_all": {}
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "salesPrice"
          }
        }
      }
    }
    
    "aggregations": {
        "avg_price": {
            "value": 428.51063644785825
        }
    }

### 7.3.2 Cardinality Aggregation
A single-value metrics aggregation that calculates an approximate count of distinct values. Values can be extracted either from specific fields in the document or generated by a script.

    {
      "size": 0,
      "query": {
        "match_all": {}
      },
      "aggs": {
        "brand_count": {
          "cardinality": {
            "field": "brandId"
          }
        }
      }
    }

    "aggregations": {
        "brand_count": {
            "value": 1186
        }
    }

### 7.3.3 Stats Aggregation
A multi-value metrics aggregation that computes stats over numeric values extracted from the aggregated documents. These values can be extracted either from specific numeric fields in the documents, or be generated by a provided script.

The stats that are returned consist of: min, max, sum, count and avg.

    {
      "size": 0,
      "query": {
        "match_all": {}
      },
      "aggs": {
        "price_stat": {
          "stats": {
            "field": "salesPrice"
          }
        }
      }
    }
    
    "aggregations": {
        "price_stat": {
            "count": 221275,
            "min": 0,
            "max": 131231,
            "avg": 428.51063644785825,
            "sum": 94818691.07999983
        }
    }
    
### 7.3.4 Extended Stats Aggregation
A multi-value metrics aggregation that computes stats over numeric values extracted from the aggregated documents. These values can be extracted either from specific numeric fields in the documents, or be generated by a provided script.

The extended_stats aggregations is an extended version of the stats aggregation, where additional metrics are added such as sum_of_squares, variance, std_deviation and std_deviation_bounds.

    {
      "size": 0,
      "query": {
        "match_all": {}
      },
      "aggs": {
        "price_stat": {
          "extended_stats": {
            "field": "salesPrice"
          }
        }
      }
    }

    "aggregations": {
        "price_stat": {
            "count": 221275,
            "min": 0,
            "max": 131231,
            "avg": 428.51063644785825,
            "sum": 94818691.07999983,
            "sum_of_squares": 118950750156.63016,
            "variance": 353948.4012870255,
            "std_deviation": 594.9356278514723,
            "std_deviation_bounds": {
            "upper": 1618.3818921508027,
            "lower": -761.3606192550864
            }
        }
    }
    
### 7.3.5 Geo Bounds Aggregation (Skipped)
### 7.3.6 Geo Centroid Aggregation (Skipped)

### 7.3.7 Max Aggregation
A single-value metrics aggregation that keeps track and returns the maximum value among the numeric values extracted from the aggregated documents.

    "aggs": {
        "max_price": {
            "max": {
                "field": "salesPrice"
            }
        }
    }
  
### 7.3.8 Min Aggregation
A single-value metrics aggregation that keeps track and returns the minimum value among numeric values extracted from the aggregated documents.

    "aggs": {
        "min_price": {
          "min": {
            "field": "salesPrice"
          }
        }
      }

### 7.3.9 Percentiles Aggregation
A multi-value metrics aggregation that calculates one or more percentiles over numeric values extracted from the aggregated documents. 

Percentiles show the point at which a certain percentage of observed values occur. For example, the 95th percentile is the value which is greater than 95% of the observed values.

When a range of percentiles are retrieved, they can be used to estimate the data distribution and determine if the data is skewed, bimodal, etc.

    "aggs": {
        "price_outlier": {
          "percentiles": {
            "field": "salesPrice"
          }
        }
      }
  
    "aggregations": {
        "price_outlier": {
            "values": {
                "1.0": 19,
                "5.0": 49.049088235742005,
                "25.0": 148.8903318997934,
                "50.0": 288.33201291736634,
                "75.0": 521.2972145384141,
                "95.0": 1286.9096656603726,
                "99.0": 2497.931283641535
            }
        }
    }
    
### 7.3.10 Percentile Ranks Aggregation
A multi-value metrics aggregation that calculates one or more percentile ranks over numeric values extracted from the aggregated documents.
    
Percentile rank show the percentage of observed values which are below certain value. For example, if a value is greater than or equal to 95% of the observed values it is said to be at the 95th percentile rank.

    "aggs": {
        "price_outlier": {
          "percentile_ranks": {
            "field": "salesPrice",
            "values": [
              200,
              500
            ]
          }
        }
      }
      
    "aggregations": {
        "price_outlier": {
            "values": {
                "200.0": 37.906112721751086,
                "500.0": 74.407593883831
            }
        }
    }
    
### 7.3.11 Scripted Metric Aggregation (experimental and may be changed or removed)
A metric aggregation that executes using scripts to provide a metric output.

See a detailed example: https://www.elastic.co/guide/en/elasticsearch/reference/2.3/search-aggregations-metrics-scripted-metric-aggregation.html

### 7.3.12 Sum Aggregation
A single-value metrics aggregation that sums up numeric values that are extracted from the aggregated documents.
    
    {
      "query": {
        "term": {
          "brandName": "vans"
        }
      },
      "aggs": {
        "salesNum_total": {
          "sum": {
            "field": "salesNum"
          }
        }
      }
    }
        
    "aggregations": {
        "salesNum_total": {
            "value": 253365
        }
    }
    
### 7.3.13 Top hits Aggregation
A top_hits metric aggregator keeps track of the most relevant document being aggregated. This aggregator is intended to be used as a sub aggregator, so that the top matching documents can be aggregated per bucket.

The top_hits aggregator can effectively be used to group result sets by certain fields via a bucket aggregator. One or more bucket aggregators determines by which properties a result set get sliced into.

Options:
- from: The offset from the first result you want to fetch.
- size: The maximum number of top matching hits to return per bucket. By default the top three matching hits are returned.
- sort: How the top matching hits should be sorted. By default the hits are sorted by the score of the main query.   

### 7.3.14 Value Count Aggregation
A single-value metrics aggregation that counts the number of values that are extracted from the aggregated documents. Typically, this aggregator will be used in conjunction with other single-value aggregations. For example, when computing the avg one might be interested in the number of values the average is computed over.

## 7.4 Bucket Aggregations
Bucket aggregations don’t calculate metrics over fields like the metrics aggregations do, but instead, they create buckets of documents. Each bucket is associated with a criterion (depending on the aggregation type) which determines whether or not a document in the current context "falls" into it. In other words, the buckets effectively define document sets. In addition to the buckets themselves, the bucket aggregations also compute and return the number of documents that "fell into" each bucket.

Bucket aggregations, as opposed to metrics aggregations, can hold sub-aggregations. These sub-aggregations will be aggregated for the buckets created by their "parent" bucket aggregation.

There are different bucket aggregators, each with a different "bucketing" strategy. Some define a single bucket, some define fixed number of multiple buckets, and others dynamically create the buckets during the aggregation process.

### 7.4.1 Children Aggregation
A special single bucket aggregation that enables aggregating from buckets on parent document types to buckets on child documents.

### 7.4.2 Histogram Aggregation
A multi-bucket values source based aggregation that can be applied on numeric values extracted from the documents. It dynamically builds fixed size (a.k.a. interval) buckets over the values. For example, if the documents have a field that holds a price (numeric), we can configure this aggregation to dynamically build buckets with interval 5 (in case of price it may represent $5). When the aggregation executes, the price field of every document will be evaluated and will be rounded down to its closest bucket - for example, if the price is 32 and the bucket size is 5 then the rounding will yield 30 and thus the document will "fall" into the bucket that is associated with the key 30.

From the rounding function above it can be seen that the intervals themselves must be integers.

    {
      "size": 0,
      "query": {
        "term": {
          "brandName": "vans"
        }
      },
      "aggs": {
        "prices": {
          "histogram": {
            "field": "salesPrice",
            "interval": 200
          }
        }
      }
    }
    
    "aggregations": {
        "prices": {
        "buckets": [
            {
            "key": 0,
            "doc_count": 838
            }
            ,
            {
            "key": 200,
            "doc_count": 1123
            }
            ,
            {
            "key": 400,
            "doc_count": 804
            }
            ,
            {
            "key": 600,
            "doc_count": 283
            }
            ,
            {
            "key": 800,
            "doc_count": 64
            }
            ,
            {
            "key": 1000,
            "doc_count": 16
            }
            ,
            {
            "key": 1200,
            "doc_count": 18
            }
            ,
            {
            "key": 1400,
            "doc_count": 8
            }
            ,
            {
            "key": 1600,
            "doc_count": 7
            }
            ]
        }
    }

### 7.4.3 Date Histogram Aggregation
A multi-bucket aggregation similar to the histogram except it can only be applied on date values. Since dates are represented in elasticsearch internally as long values, it is possible to use the normal histogram on dates as well, though accuracy will be compromised. The reason for this is in the fact that time based intervals are not fixed (think of leap years and on the number of days in a month). For this reason, we need special support for time based data. From a functionality perspective, this histogram supports the same features as the normal histogram. The main difference is that the interval can be specified by date/time expressions.

Requesting bucket intervals of a month.

    {
        "aggs" : {
            "articles_over_time" : {
                "date_histogram" : {
                    "field" : "date",
                    "interval" : "month"
                }
            }
        }
    }

### 7.4.4 Range Aggregation
A multi-bucket value source based aggregation that enables the user to define a set of ranges - each representing a bucket. During the aggregation process, the values extracted from each document will be checked against each bucket range and "bucket" the relevant/matching document. Note that this aggregation includes the from value and excludes the to value for each range.

    {
      "size": 0,
      "query": {
        "term": {
          "brandName": "vans"
        }
      },
      "aggs": {
        "price_ranges": {
          "range": {
            "field": "salesPrice",
            "ranges": [
              {
                "to": 200
              },
              {
                "from": 200,
                "to": 500
              },
              {
                "from": 500
              }
            ]
          }
        }
      }
    }
    
    "aggregations": {
        "price_ranges": {
            "buckets": [
                {
                    "key": "*-200.0",
                    "to": 200,
                    "to_as_string": "200.0",
                    "doc_count": 838
                }
                ,
                {
                    "key": "200.0-500.0",
                    "from": 200,
                    "from_as_string": "200.0",
                    "to": 500,
                    "to_as_string": "500.0",
                    "doc_count": 1594
                }
                ,
                {
                    "key": "500.0-*",
                    "from": 500,
                    "from_as_string": "500.0",
                    "doc_count": 729
                }
            ]
        }
    }

### 7.4.5 Date Range Aggregation
A range aggregation that is dedicated for date values. The main difference between this aggregation and the normal range aggregation is that the from and to values can be expressed in Date Math expressions, and it is also possible to specify a date format by which the from and to response fields will be returned. Note that this aggregation includes the from value and excludes the to value for each range.

    {
        "aggs": {
            "range": {
                "date_range": {
                    "field": "date",
                    "format": "MM-yyy",
                    "ranges": [
                        { "to": "now-10M/M" }, 
                        { "from": "now-10M/M" } 
                    ]
                }
            }
        }
    }

### 7.4.6 Filter Aggregation
Defines a single bucket of all the documents in the current document set context that match a specified filter. Often this will be used to narrow down the current aggregation context to a specific set of documents.

    {
      "size": 0,
      "query": {
        "term": {
          "brandName": "vans"
        }
      },
      "aggs": {
        "red_products": {
          "filter": {
            "term": {
              "colorNames": "红色"
            }
          },
          "aggs": {
            "avg_price": {
              "avg": {
                "field": "salesPrice"
              }
            }
          }
        }
      }
    }

### 7.4.7 Filters Aggregation
Defines a multi bucket aggregation where each bucket is associated with a filter. Each bucket will collect all documents that match its associated filter.

    {
      "aggs" : {
        "messages" : {
          "filters" : {
            "filters" : {
              "errors" :   { "term" : { "body" : "error"   }},
              "warnings" : { "term" : { "body" : "warning" }}
            }
          },
          "aggs" : {
            "monthly" : {
              "histogram" : {
                "field" : "timestamp",
                "interval" : "1M"
              }
            }
          }
        }
      }
    }

In the above example, we analyze log messages. The aggregation will build two collection (buckets) of log messages - one for all those containing an error, and another for all those containing a warning. And for each of these buckets it will break them down by month.Response:

      "aggs" : {
        "messages" : {
          "buckets" : {
            "errors" : {
              "doc_count" : 34,
              "monthly" : {
                "buckets" : [
                  ... // the histogram monthly breakdown
                ]
              }
            },
            "warnings" : {
              "doc_count" : 439,
              "monthly" : {
                "buckets" : [
                   ... // the histogram monthly breakdown
                ]
              }
            }
          }
        }
      }


### 7.4.8 Geo Distance Aggregation (Skipped)
### 7.4.9 GeoHash grid Aggregation (Skipped)

### 7.4.10 Global Aggregation
Defines a single bucket of all the documents within the search execution context. This context is defined by the indices and the document types you’re searching on, but is not influenced by the search query itself.

### 7.4.11 IPv4 Range Aggregation
Just like the dedicated date range aggregation, there is also a dedicated range aggregation for IPv4 typed fields:

### 7.4.12 Missing Aggregation
A field data based single bucket aggregation, that creates a bucket of all documents in the current document set context that are missing a field value (effectively, missing a field or having the configured NULL value set). This aggregator will often be used in conjunction with other field data bucket aggregators (such as ranges) to return information for all the documents that could not be placed in any of the other buckets due to missing field data values.

### 7.4.13 Nested Aggregation
A special single bucket aggregation that enables aggregating nested documents.

### 7.4.14 Reverse nested Aggregation
A special single bucket aggregation that enables aggregating on parent docs from nested documents. Effectively this aggregation can break out of the nested block structure and link to other nested structures or the root document, which allows nesting other aggregations that aren’t part of the nested object in a nested aggregation.

### 7.4.15 Significant Terms Aggregation
An aggregation that returns interesting or unusual occurrences of terms in a set.

> Warning: The significant_terms aggregation can be very heavy when run on large indices. Work is in progress to provide more lightweight sampling techniques. As a result, the API for this feature may change in backwards incompatible ways.

Example use cases:

- Suggesting "H5N1" when users search for "bird flu" in text
- Identifying the merchant that is the "common point of compromise" from the transaction history of credit card owners reporting loss
- Suggesting keywords relating to stock symbol $ATI for an automated news classifier
- Spotting the fraudulent doctor who is diagnosing more than his fair share of whiplash injuries
- Spotting the tire manufacturer who has a disproportionate number of blow-outs

    {
      "query": {
        "terms": {
          "smallSort": [
            "牛仔裤"
          ]
        }
      },
      "aggregations": {
        "significantColors": {
          "significant_terms": {
            "field": "colorNames"
          }
        }
      }
    }

    "aggregations": {
        "significantColors": {
            "doc_count": 7365,
            "buckets": [
                {
                    "key": "蓝色",
                    "doc_count": 4750,
                    "score": 1.9775784118031037,
                    "bg_count": 35656
                }
                ,
                {
                    "key": "蓝",
                    "doc_count": 1287,
                    "score": 0.35538801144606,
                    "bg_count": 12949
                }
                ,
                {
                    "key": "原色",
                    "doc_count": 39,
                    "score": 0.09168423890725523,
                    "bg_count": 65
                }
                ,
                {
                    "key": "浅蓝色",
                    "doc_count": 79,
                    "score": 0.031059308568117887,
                    "bg_count": 619
                }
                ,
                {
                    "key": "水洗",
                    "doc_count": 10,
                    "score": 0.030522422208204166,
                    "bg_count": 13
                }
                ,
                {
                    "key": "深蓝色",
                    "doc_count": 131,
                    "score": 0.024955048079471253,
                    "bg_count": 1664
                }
            ]
        }
    }

GINO: 为什么深蓝色排在最后？
在所有的商品(总数为224778)中，共有1664件商品为深蓝色；但是对于牛仔裤(总数为7365)，只有131件牛仔裤为深蓝色，因此认为他们之间的关联度很低，就不是很推荐深蓝色的牛仔裤。再试试看品牌推荐的效果：

    {
      "query": {
        "terms": {
          "smallSort": [
            "牛仔裤"
          ]
        }
      },
      "aggregations": {
        "significantBrands": {
          "significant_terms": {
            "field": "brandNameEn"
          }
        }
      }
    }

    "aggregations": {
        "significantBrands": {
            "doc_count": 7365,
            "buckets": [
            {
                "key": "xinfeiyang",
                "doc_count": 1061,
                "score": 4.179762739096525,
                "bg_count": 1079
            }
            ,
            {
                "key": "lee",
                "doc_count": 432,
                "score": 0.5581216486055066,
                "bg_count": 1254
            }
            ,
            {
                "key": "levi's",
                "doc_count": 473,
                "score": 0.5004641576199044,
                "bg_count": 1642
            }
            ,
            {
                "key": "jasonwood",
                "doc_count": 495,
                "score": 0.3789564299098067,
                "bg_count": 2276
            }
            ,
            {
                "key": "able",
                "doc_count": 304,
                "score": 0.36273857444325336,
                "bg_count": 948
            }
            ,
            {
                "key": "jeans",
                "doc_count": 307,
                "score": 0.3581144552023549,
                "bg_count": 977
            }
            ,
            {
                "key": "agamemnon",
                "doc_count": 119,
                "score": 0.3311112854845905,
                "bg_count": 169
            }
            ,
            {
                "key": "krbl/korakublue",
                "doc_count": 183,
                "score": 0.31616211163218183,
                "bg_count": 407
            }
            ,
            {
                "key": "wrangler",
                "doc_count": 198,
                "score": 0.264814268306479,
                "bg_count": 557
            }
            ,
            {
                "key": "evisu",
                "doc_count": 143,
                "score": 0.19733121908058618,
                "bg_count": 391
            }
            ]
        }
    }

### 7.4.16 Sampler Aggregation (experimental and may be changed or removed)
A filtering aggregation used to limit any sub aggregations' processing to a sample of the top-scoring documents. Optionally, diversity settings can be used to limit the number of matches that share a common value such as an "author".

Example use cases:

- Tightening the focus of analytics to high-relevance matches rather than the potentially very long tail of low-quality matches
- Removing bias from analytics by ensuring fair representation of content from different sources
- Reducing the running cost of aggregations that can produce useful results using only samples e.g. significant_terms

### 7.4.17 Terms Aggregation
A multi-bucket value source based aggregation where buckets are dynamically built - one per unique value.

    {
      "query": {
        "terms": {
          "smallSort": [
            "牛仔裤"
          ]
        }
      },
      "aggs": {
        "genders": {
          "terms": {
            "field": "genderS"
          }
        }
      }
    }

    "aggregations": {
        "genders": {
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 0,
            "buckets": [
            {
                "key": "男",
                "doc_count": 5271
            }
            ,
            {
                "key": "女",
                "doc_count": 2255
            }
            ]
        }
    }

The size parameter can be set to define how many term buckets should be returned out of the overall terms list. By default, the node coordinating the search process will request each shard to provide its own top size term buckets and once all shards respond, it will reduce the results to the final list that will then be returned to the client. This means that if the number of unique terms is greater than size, the returned list is slightly off and not accurate (it could be that the term counts are slightly off and it could even be that a term that should have been in the top size buckets was not returned). If set to 0, the size will be set to Integer.MAX_VALUE.

## 7.5 Pipeline Aggregations (experimental and may be changed or removed, Skipped）
## 7.6 Caching heavy aggregations
Frequently used aggregations (e.g. for display on the home page of a website) can be cached for faster responses. These cached results are the same results that would be returned by an uncached aggregation -- you will never get stale results.

See [Shard request cache](https://www.elastic.co/guide/en/elasticsearch/reference/2.3/shard-request-cache.html) for more details.
    
## 7.7 Returning only aggregation results
There are many occasions when aggregations are required but search hits are not. For these cases the hits can be ignored by setting size=0. For example:

    curl -XGET 'http://localhost:9200/twitter/tweet/_search' -d '{
      "size": 0,
      "aggregations": {
        "my_agg": {
          "terms": {
            "field": "text"
          }
        }
      }
    }
    '
Setting size to 0 avoids executing the fetch phase of the search making the request more efficient.
    
# 8 Indices APIs    
The indices APIs are used to manage individual indices, index settings, aliases, mappings, index templates and warmers.

## 8.1 Create Index
The create index API allows to instantiate an index. Elasticsearch provides support for multiple indices, including executing operations across several indices.

    curl -XPUT 'http://localhost:9200/twitter/' -d '{
        "settings" : {
            "index" : {
                "number_of_shards" : 3, 
                "number_of_replicas" : 2 
            }
        }
    }'
    
The create index API allows to provide a set of one or more mappings:

    curl -XPOST localhost:9200/test -d '{
        "settings" : {
            "number_of_shards" : 1
        },
        "mappings" : {
            "type1" : {
                "properties" : {
                    "field1" : { "type" : "string", "index" : "not_analyzed" }
                }
            }
        }
    }'
    
## 8.2 Delete Index
The delete index API allows to delete an existing index.

    curl -XDELETE 'http://localhost:9200/twitter/'

## 8.3 Get Index
The get index API allows to retrieve information about one or more indexes.

    curl -XGET 'http://localhost:9200/twitter/'

## 8.4 Indices Exists
Used to check if the index (indices) exists or not. For example:

    curl -XHEAD -i 'http://localhost:9200/twitter'
The HTTP status code indicates if the index exists or not. A 404 means it does not exist, and 200 means it does.

## 8.5 Open / Close Index API
The open and close index APIs allow to close an index, and later on opening it. A closed index has almost no overhead on the cluster (except for maintaining its metadata), and is blocked for read/write operations. A closed index can be opened which will then go through the normal recovery process.

The REST endpoint is /{index}/_close and /{index}/_open. For example:

    curl -XPOST 'localhost:9200/my_index/_close'
    curl -XPOST 'localhost:9200/my_index/_open'

## 8.6 Put Mapping
The PUT mapping API allows you to provide type mappings while creating a new index, add a new type to an existing index, or add new fields to an existing type:

    PUT twitter 
    {
      "mappings": {
        "tweet": {
          "properties": {
            "message": {
              "type": "string"
            }
          }
        }
      }
    }
    
    PUT twitter/_mapping/user 
    {
      "properties": {
        "name": {
          "type": "string"
        }
      }
    }
    
    PUT twitter/_mapping/tweet 
    {
      "properties": {
        "user_name": {
          "type": "string"
        }
      }
    }



- Creates an index called twitter with the message field in the tweet mapping type.
- Uses the PUT mapping API to add a new mapping type called user.
- Uses the PUT mapping API to add a new field called user_name to the tweet mapping type.

## 8.7 Get Mapping
The get mapping API allows to retrieve mapping definitions for an index or index/type.

    curl -XGET 'http://localhost:9200/twitter/_mapping/tweet'

## 8.8 Get Field Mapping
The get field mapping API allows you to retrieve mapping definitions for one or more fields. This is useful when you do not need the complete type mapping returned by the Get Mapping API.

The following returns the mapping of the field text only:

    curl -XGET 'http://localhost:9200/twitter/_mapping/tweet/field/text'

## 8.9 Types Exists
Used to check if a type/types exists in an index/indices.

    curl -XHEAD -i 'http://localhost:9200/twitter/tweet'

## 8.10 Index Aliases
APIs in elasticsearch accept an index name when working against a specific index, and several indices when applicable. The index aliases API allow to alias an index with a name, with all APIs automatically converting the alias name to the actual index name. An alias can also be mapped to more than one index, and when specifying it, the alias will automatically expand to the aliases indices. An alias can also be associated with a filter that will automatically be applied when searching, and routing values. An alias cannot have the same name as an index.

Here is a sample of associating the alias alias1 with index test1:

    curl -XPOST 'http://localhost:9200/_aliases' -d '
    {
        "actions" : [
            { "add" : { "index" : "test1", "alias" : "alias1" } }
        ]
    }'


### Filtered Aliases
Aliases with filters provide an easy way to create different "views" of the same index. The filter can be defined using Query DSL and is applied to all Search, Count, Delete By Query and More Like This operations with this alias.

To create a filtered alias, first we need to ensure that the fields already exist in the mapping:

    curl -XPUT 'http://localhost:9200/test1' -d '{
      "mappings": {
        "type1": {
          "properties": {
            "user" : {
              "type": "string",
              "index": "not_analyzed"
            }
          }
        }
      }
    }'
Now we can create an alias that uses a filter on field user:

    curl -XPOST 'http://localhost:9200/_aliases' -d '{
        "actions" : [
            {
                "add" : {
                     "index" : "test1",
                     "alias" : "alias2",
                     "filter" : { "term" : { "user" : "kimchy" } }
                }
            }
        ]
    }'

An alias can also be added or deleted with the endpoint

    curl -XPUT 'localhost:9200/logs_201305/_alias/2013'
    curl -XDELETE 'localhost:9200/logs_201305/_alias/2013'
    curl -XGET 'localhost:9200/_alias/2013'
    curl -XHEAD -i 'localhost:9200/_alias/2013'
    
## 8.11 Update Indices Settings
Change specific index level settings in real time.

The REST endpoint is /_settings (to update all indices) or {index}/_settings to update one (or more) indices settings.

    curl -XPUT 'localhost:9200/my_index/_settings' -d '
    {
        "index" : {
            "number_of_replicas" : 4
        }
    }'

### Bulk Indexing Usage
For example, the update settings API can be used to dynamically change the index from being more performant for bulk indexing, and then move it to more real time indexing state. Before the bulk indexing is started, use:

    curl -XPUT localhost:9200/test/_settings -d '{
        "index" : {
            "refresh_interval" : "-1"
        } }'
(Another optimization option is to start the index without any replicas, and only later adding them, but that really depends on the use case).

Then, once bulk indexing is done, the settings can be updated (back to the defaults for example):

    curl -XPUT localhost:9200/test/_settings -d '{
        "index" : {
            "refresh_interval" : "1s"
        } }'
And, a force merge should be called:

    curl -XPOST 'http://localhost:9200/test/_forcemerge?max_num_segments=5'

## 8.12 Get Settings
The get settings API allows to retrieve settings of index/indices:

    curl -XGET 'http://localhost:9200/twitter/_settings'

## 8.13 Analyze
Performs the analysis process on a text and return the tokens breakdown of the text.

Can be used without specifying an index against one of the many built in analyzers:

    curl -XGET 'localhost:9200/twitter/_analyze' -d '
    {
      "analyzer" : "standard",
      "text" : "this is a test"
    }'

All parameters can also supplied as request parameters. For example:

    curl -XGET 'localhost:9200/_analyze?tokenizer=keyword&filter=lowercase&text=this+is+a+test'

### Explain Analyze
If you want to get more advanced details, set explain to true (defaults to false). It will output all token attributes for each token. You can filter token attributes you want to output by setting attributes option.

    POST productindex/_analyze
    {
      "field": "productName.productName_ansj",
      "text": "S.T.A.M.P.S./诗坦表 时尚PU皮表带",
      "explain": true
    }

## 8.14 Index Templates
Index templates allow you to define templates that will automatically be applied when new indices are created. The templates include both settings and mappings, and a simple pattern template that controls whether the template should be applied to the new index.

## 8.15 Shadow replica indices(experimental and may be changed or removed)
If you would like to use a shared filesystem, you can use the shadow replicas settings to choose where on disk the data for an index should be kept, as well as how Elasticsearch should replay operations on all the replica shards of an index.

## 8.16 Indices Stats
Indices level stats provide statistics on different operations happening on an index. The API provides statistics on the index level scope (though most stats can also be retrieved using node level scope).

The following returns high level aggregation and index level stats for all indices:

    curl localhost:9200/_stats
Specific index stats can be retrieved using:

    curl localhost:9200/index1,index2/_stats

By default, all stats are returned, returning only specific stats can be specified as well in the URI. Those stats can be any of:

- docs:The number of docs / deleted docs (docs not yet merged out). Note, affected by refreshing the index.
- store: The size of the index.
- indexing: Indexing statistics, can be combined with a comma separated list of types to provide document type level stats.
- get: Get statistics, including missing stats.
- search: Search statistics. You can include statistics for custom groups by adding an extra groups parameter (search operations can be associated with one or more groups). The groups parameter accepts a comma separated list of group names. Use _all to return statistics for all groups.
- completion: Completion suggest statistics.
- fielddata: Fielddata statistics.
- flush:Flush statistics.
- merge:Merge statistics.
- request_cache: Shard request cache statistics.
- refresh: Refresh statistics.
- suggest: Suggest statistics.
- warmer: Warmer statistics.
- translog: Translog statistics.

## 8.17 Indices Segments
Provide low level segments information that a Lucene index (shard level) is built with. Allows to be used to provide more information on the state of a shard and an index, possibly optimization information, data "wasted" on deletes, and so on.

Endpoints include segments for a specific index, several indices, or all:

    curl -XGET 'http://localhost:9200/test/_segments'
    curl -XGET 'http://localhost:9200/test1,test2/_segments'
    curl -XGET 'http://localhost:9200/_segments'

- _0: The key of the JSON document is the name of the segment. This name is used to generate file names: all files starting with this segment name in the directory of the shard belong to this segment.
- generation: A generation number that is basically incremented when needing to write a new segment. The segment name is derived from this generation number.
- num_docs: The number of non-deleted documents that are stored in this segment.
- deleted_docs: The number of deleted documents that are stored in this segment. It is perfectly fine if this number is greater than 0, space is going to be reclaimed when this segment gets merged.
- size_in_bytes: The amount of disk space that this segment uses, in bytes.
- memory_in_bytes: Segments need to store some data into memory in order to be searchable efficiently. This number returns the number of bytes that are used for that purpose. A value of -1 indicates that Elasticsearch was not able to compute this number.
- committed: Whether the segment has been sync’ed on disk. Segments that are committed would survive a hard reboot. No need to worry in case of false, the data from uncommitted segments is also stored in the transaction log so that Elasticsearch is able to replay changes on the next start.
- search: Whether the segment is searchable. A value of false would most likely mean that the segment has been written to disk but no refresh occurred since then to make it searchable.
- version: The version of Lucene that has been used to write this segment.
- compound: Whether the segment is stored in a compound file. When true, this means that Lucene merged all files from the segment in a single one in order to save file descriptors.

## 8.18 Indices Recovery(Advanced Topic)
The indices recovery API provides insight into on-going index shard recoveries. Recovery status may be reported for specific indices, or cluster-wide.

## 8.19 Indices Shard Stores
Provides store information for shard copies of indices. Store information reports on which nodes shard copies exist, the shard copy version, indicating how recent they are, and any exceptions encountered while opening the shard index or from earlier engine failure.

By default, only lists store information for shards that have at least one unallocated copy. When the cluster health status is yellow, this will list store information for shards that have at least one unassigned replica. When the cluster health status is red, this will list store information for shards, which has unassigned primaries.

Endpoints include shard stores information for a specific index, several indices, or all:

    curl -XGET 'http://localhost:9200/test/_shard_stores'
    curl -XGET 'http://localhost:9200/test1,test2/_shard_stores'
    curl -XGET 'http://localhost:9200/_shard_stores'

## 8.20 Clear Cache
The clear cache API allows to clear either all caches or specific cached associated with one or more indices.

    curl -XPOST 'http://localhost:9200/twitter/_cache/clear'
The API, by default, will clear all caches. Specific caches can be cleaned explicitly by setting query, fielddata or request.

All caches relating to a specific field(s) can also be cleared by specifying fields parameter with a comma delimited list of the relevant fields.

## 8.21 Flush
The flush API allows to flush one or more indices through an API. The flush process of an index basically frees memory from the index by flushing data to the index storage and clearing the internal transaction log. By default, Elasticsearch uses memory heuristics in order to automatically trigger flush operations as required in order to clear memory.

    POST /twitter/_flush

The flush API accepts the following request parameters:
- wait_if_ongoing: If set to true the flush operation will block until the flush can be executed if another flush operation is already executing. The default is false and will cause an exception to be thrown on the shard level if another flush operation is already running.
- force: Whether a flush should be forced even if it is not necessarily needed ie. if no changes will be committed to the index. This is useful if transaction log IDs should be incremented even if no uncommitted changes are present. (This setting can be considered as internal)

## 8.22 Synced Flush(Advanced Topic)
Elasticsearch tracks the indexing activity of each shard. Shards that have not received any indexing operations for 5 minutes are automatically marked as inactive. This presents an opportunity for Elasticsearch to reduce shard resources and also perform a special kind of flush, called synced flush. A synced flush performs a normal flush, then adds a generated unique marker (sync_id) to all shards.

## 8.23 Refresh
The refresh API allows to explicitly refresh one or more index, making all operations performed since the last refresh available for search. The (near) real-time capabilities depend on the index engine used. For example, the internal one requires refresh to be called, but by default a refresh is scheduled periodically.

    curl -XPOST 'http://localhost:9200/twitter/_refresh'
    
## 8.24 Force Merge
The force merge API allows to force merging of one or more indices through an API. The merge relates to the number of segments a Lucene index holds within each shard. The force merge operation allows to reduce the number of segments by merging them.

This call will block until the merge is complete. If the http connection is lost, the request will continue in the background, and any new requests will block until the previous force merge is complete.

    curl -XPOST 'http://localhost:9200/twitter/_forcemerge'

The force merge API accepts the following request parameters:
- max_num_segments: The number of segments to merge to. To fully merge the index, set it to 1. Defaults to simply checking if a merge needs to execute, and if so, executes it.
- only_expunge_deletes: Should the merge process only expunge segments with deletes in it. In Lucene, a document is not deleted from a segment, just marked as deleted. During a merge process of segments, a new segment is created that does not have those deletes. This flag allows to only merge segments that have deletes. Defaults to false. Note that this won’t override the index.merge.policy.expunge_deletes_allowed threshold.
- flush: Should a flush be performed after the forced merge. Defaults to true.

# 9 cat APIs
JSON is great… for computers. Even if it’s pretty-printed, trying to find relationships in the data is tedious. Human eyes, especially when looking at an ssh terminal, need compact and aligned text. The cat API aims to meet this need.

All the cat commands accept a query string parameter help to see all the headers and info they provide, and the /_cat command alone lists all the available commands.

## 9.1 cat aliases
aliases shows information about currently configured aliases to indices including filter and routing infos.

    curl 'localhost:9200/_cat/aliases?v'
## 9.2 cat allocation
allocation provides a snapshot of how many shards are allocated to each data node and how much disk space they are using.

    curl 'localhost:9200/_cat/allocation?v'

## 9.3 cat count
count provides quick access to the document count of the entire cluster, or individual indices.

    curl 'localhost:9200/_cat/indices?v'
    green wiki1 3 0 10000 331 168.5mb 168.5mb
    green wiki2 3 0   428   0     8mb     8mb
    curl 'localhost:9200/_cat/count?v'
    1384314124582 19:42:04 10428
    curl 'localhost:9200/_cat/count/wiki2?v'
    1384314139815 19:42:19 428

## 9.4 cat fielddata
fielddata shows how much heap memory is currently being used by fielddata on every data node in the cluster.

    curl 'localhost:9200/_cat/fielddata?v'
    id                     host    ip            node          total   body    text
    c223lARiSGeezlbrcugAYQ myhost1 10.20.100.200 Jessica Jones 385.6kb 159.8kb 225.7kb
    waPCbitNQaCL6xC8VxjAwg myhost2 10.20.100.201 Adversary     435.2kb 159.8kb 275.3kb
    yaDkp-G3R0q1AJ-HUEvkSQ myhost3 10.20.100.202 Microchip     284.6kb 109.2kb 175.3kb
Fields can be specified either as a query parameter, or in the URL path.

## 9.5 cat health
health is a terse, one-line representation of the same information from /_cluster/health. It has one option ts to disable the timestamping.

    curl 'localhost:9200/_cat/health?v&ts=0'
    cluster status nodeTotal nodeData shards pri relo init unassign tasks
    foo     green          3        3      3   3    0    0        0     0

## 9.6 cat indices
The indices command provides a cross-section of each index. This information spans nodes.

    curl 'localhost:9200/_cat/indices?v'

## 9.7 cat master
master doesn’t have any extra options. It simply displays the master’s node ID, bound IP address, and node name.

    curl 'localhost:9200/_cat/master?v'
    id                     ip            node
    Ntgn2DcuTjGuXlhKDUD4vA 192.168.56.30 Solarr
    
## 9.8 cat nodeattrs
The nodeattrs command shows custom node attributes.

    curl 'localhost:9200/_cat/nodeattrs?v'
    node       host    ip          attr  value
    Black Bolt epsilon 192.168.1.8 rack  rack314
    Black Bolt epsilon 192.168.1.8 azone us-east-1    
    
## 9.9 cat nodes
The nodes command shows the cluster topology.

    curl 'localhost:9200/_cat/nodes?v'
    SP4H 4727 192.168.56.30 9300 2.3.4 1.8.0_73 72.1gb 35.4 93.9mb 79 239.1mb 0.45 3.4h d m Boneyard
    _uhJ 5134 192.168.56.10 9300 2.3.4 1.8.0_73 72.1gb 33.3 93.9mb 85 239.1mb 0.06 3.4h d * Athena
    HfDp 4562 192.168.56.20 9300 2.3.4 1.8.0_73 72.2gb 74.5 93.9mb 83 239.1mb 0.12 3.4h d m Zarek
## 9.10 cat pending tasks
pending_tasks provides the same information as the /_cluster/pending_tasks API in a convenient tabular format.

    curl 'localhost:9200/_cat/pending_tasks?v'
    
## 9.11 cat plugins
The plugins command provides a view per node of running plugins. This information spans nodes.

    curl 'localhost:9200/_cat/plugins?v'
    
## 9.12 cat recovery
The recovery command is a view of index shard recoveries, both on-going and previously completed. It is a more compact view of the JSON recovery API.

    curl 'localhost:9200/_cat/recovery?v'
    
## 9.13 cat repositories
The repositories command shows the snapshot repositories registered in the cluster.

    curl 'localhost:9200/_cat/repositories?v'
    
## 9.14 cat thread pool
The thread_pool command shows cluster wide thread pool statistics per node. By default the active, queue and rejected statistics are returned for the bulk, index and search thread pools.

    curl 'localhost:9200/_cat/thread_pool?v'

## 9.15 cat shards
The shards command is the detailed view of what nodes contain which shards. It will tell you if it’s a primary or replica, the number of docs, the bytes it takes on disk, and the node where it’s located.

Here we see a single index, with three primary shards and no replicas:

    curl 'localhost:9200/_cat/shards?v'

## 9.16 cat segments
The segments command provides low level information about the segments in the shards of an index. It provides information similar to the _segments endpoint.

    curl 'http://localhost:9200/_cat/segments?v'

## 9.17 cat snapshots
The snapshots command shows all snapshots that belong to a specific repository. To find a list of available repositories to query, the command /_cat/repositories can be used. Querying the snapshots of a repository named repo1 then looks as follows.

    curl 'localhost:9200/_cat/snapshots/repo1?v'
    
# 10. Cluster APIs
## 10.1 Cluster Health

The cluster health API allows to get a very simple status on the health of the cluster.

    curl 'http://localhost:9200/_cluster/health?pretty=true'
    {
      "cluster_name" : "testcluster",
      "status" : "green",
      "timed_out" : false,
      "number_of_nodes" : 2,
      "number_of_data_nodes" : 2,
      "active_primary_shards" : 5,
      "active_shards" : 10,
      "relocating_shards" : 0,
      "initializing_shards" : 0,
      "unassigned_shards" : 0,
      "delayed_unassigned_shards": 0,
      "number_of_pending_tasks" : 0,
      "number_of_in_flight_fetch": 0,
      "task_max_waiting_in_queue_millis": 0,
      "active_shards_percent_as_number": 100
    }

## 10.2 Cluster State
The cluster state API allows to get a comprehensive state information of the whole cluster.

    curl 'http://localhost:9200/_cluster/state'

## 10.3 Cluster Stats
The Cluster Stats API allows to retrieve statistics from a cluster wide perspective. The API returns basic index metrics (shard numbers, store size, memory usage) and information about the current nodes that form the cluster (number, roles, os, jvm versions, memory usage, cpu and installed plugins).

    curl -XGET 'http://localhost:9200/_cluster/stats?human&pretty'

## 10.4 Pending cluster tasks
The pending cluster tasks API returns a list of any cluster-level changes (e.g. create index, update mapping, allocate or fail shard) which have not yet been executed.

    curl -XGET 'http://localhost:9200/_cluster/pending_tasks'

## 10.5 Cluster Reroute(Advanced Topic)
The reroute command allows to explicitly execute a cluster reroute allocation command including specific commands. For example, a shard can be moved from one node to another explicitly, an allocation can be canceled, or an unassigned shard can be explicitly allocated on a specific node.

Here is a short example of how a simple reroute API call:

    curl -XPOST 'localhost:9200/_cluster/reroute' -d '{
        "commands" : [ {
            "move" :
                {
                  "index" : "test", "shard" : 0,
                  "from_node" : "node1", "to_node" : "node2"
                }
            },
            {
              "allocate" : {
                  "index" : "test", "shard" : 1, "node" : "node3"
              }
            }
        ]
    }'
    
## 10.6 Cluster Update Settings
Allows to update cluster wide specific settings. Settings updated can either be persistent (applied cross restarts) or transient (will not survive a full cluster restart). Here is an example:

    curl -XPUT localhost:9200/_cluster/settings -d '{
        "persistent" : {
            "discovery.zen.minimum_master_nodes" : 2
        }
    }'
Or:

    curl -XPUT localhost:9200/_cluster/settings -d '{
        "transient" : {
            "discovery.zen.minimum_master_nodes" : 2
        }
    }'
The cluster responds with the settings updated. So the response for the last example will be:

    {
        "persistent" : {},
        "transient" : {
            "discovery.zen.minimum_master_nodes" : "2"
        }
    }
Cluster wide settings can be returned using:

    curl -XGET localhost:9200/_cluster/settings 

## 10.7 Nodes statistics
The cluster nodes stats API allows to retrieve one or more (or all) of the cluster nodes statistics.

    curl -XGET 'http://localhost:9200/_nodes/stats'
    curl -XGET 'http://localhost:9200/_nodes/nodeId1,nodeId2/stats'
    
By default, all stats are returned. You can limit this by combining any of indices, os, process, jvm, transport, http, fs, breaker and thread_pool. For example:

- indices: Indices stats about size, document count, indexing and deletion times, search times, field cache size, merges and flushes
- fs: File system information, data path, free disk space, read/write stats (see FS information)
- http: HTTP connection information
- jvm: JVM stats, memory pool information, garbage collection, buffer pools, number of loaded/unloaded classes
- os: Operating system stats, load average, mem, swap (see OS statistics)
- process: Process statistics, memory consumption, cpu usage, open file descriptors (see Process statistics)
- thread_pool: Statistics about each thread pool, including current size, queue and rejected tasks
- transport: Transport statistics about sent and received bytes in cluster communication
- breaker: Statistics about the field data circuit breaker

## 10.8 Nodes Info
The cluster nodes info API allows to retrieve one or more (or all) of the cluster nodes information.

    curl -XGET 'http://localhost:9200/_nodes'
    curl -XGET 'http://localhost:9200/_nodes/nodeId1,nodeId2'
    
By default, it just returns all attributes and core settings for a node. It also allows to get only information on settings, os, process, jvm, thread_pool, transport, http and plugins.

## 10.9 Nodes hot_threads
An API allowing to get the current hot threads on each node in the cluster. Endpoints are /_nodes/hot_threads, and /_nodes/{nodesIds}/hot_threads.

    curl -XGET 'http://localhost:9200/_nodes/hot_threads'

The output is plain text with a breakdown of each node’s top hot threads. Parameters allowed are:

- threads: number of hot threads to provide, defaults to 3.
- interval: the interval to do the second sampling of threads. Defaults to 500ms.
- type: The type to sample, defaults to cpu, but supports wait and block to see hot threads that are in wait or block state.
- ignore_idle_threads: If true, known idle threads (e.g. waiting in a socket select, or to get a task from an empty queue) are filtered out. Defaults to true.
    
# 11. Query DSL

Elasticsearch provides a full Query DSL based on JSON to define queries. Think of the Query DSL as an AST of queries, consisting of two types of clauses:

- Leaf query clauses look for a particular value in a particular field, such as the match, term or range queries. These queries can be used by themselves.
- Compound query clauses wrap other leaf or compound queries and are used to combine multiple queries in a logical fashion (such as the bool or dis_max query), or to alter their behaviour (such as the not or constant_score query).

Query clauses behave differently depending on whether they are used in query context or filter context.

**Query context**
A query clause used in query context answers the question “How well does this document match this query clause?” Besides deciding whether or not the document matches, the query clause also calculates a _score representing how well the document matches, relative to other documents.

Query context is in effect whenever a query clause is passed to a query parameter, such as the query parameter in the search API.

**Filter context**
In filter context, a query clause answers the question “Does this document match this query clause?” The answer is a simple Yes or No -- no scores are calculated. Filter context is mostly used for filtering structured data, e.g.

    Does this timestamp fall into the range 2015 to 2016?
    Is the status field set to "published"?

Frequently used filters will be cached automatically by Elasticsearch, to speed up performance.

Filter context is in effect whenever a query clause is passed to a filter parameter, such as the filter or must_not parameters in the bool query, the filter parameter in the constant_score query, or the filter aggregation.

## 11.1 Match All Query
The most simple query, which matches all documents, giving them all a _score of 1.0.

    { "match_all": {} }

## 11.2 Full text queries
The high-level full text queries are usually used for running full text queries on full text fields like the body of an email. They understand how the field being queried is analyzed and will apply each field’s analyzer (or search_analyzer) to the query string before executing.

The queries in this group are:

- match query:The standard query for performing full text queries, including fuzzy matching and phrase or proximity queries.
- multi_match query: The multi-field version of the match query.
- common_terms query: A more specialized query which gives more preference to uncommon words.
- query_string query: Supports the compact Lucene query string syntax, allowing you to specify AND|OR|NOT conditions and multi-field search within a single query string. For expert users only.
- simple_query_string: A simpler, more robust version of the query_string syntax suitable for exposing directly to users.

### 11.2.1 Match Query
A family of match queries that accepts text/numerics/dates, analyzes them, and constructs a query. For example:

    POST /productindex/_search
    {
      "query": {
        "match": {
          "productName.productName_ansj": "vans鞋"
        }
      }
    }

There are three types of match query: boolean, phrase, and phrase_prefix:

**boolean**
The default match query is of type boolean. It means that the text provided is analyzed and the analysis process constructs a boolean query from the provided text. The *operator* flag can be set to or or and to control the boolean clauses (defaults to or). The minimum number of optional should clauses to match can be set using the *minimum_should_match* parameter.

The *analyzer* can be set to control which analyzer will perform the analysis process on the text. It defaults to the field explicit mapping definition, or the default search analyzer.

The *lenient* parameter can be set to true to ignore exceptions caused by data-type mismatches, such as trying to query a numeric field with a text query string. Defaults to false.

    POST /productindex/_search
    {
      "query": {
        "match": {
          "productName.productName_ansj": {
            "query": "vans鞋",
            "operator": "and"
          }
        }
      }
    }

**phrase**
The match_phrase query analyzes the text and creates a phrase query out of the analyzed text. For example:

    {
        "match_phrase" : {
            "message" : "this is a test"
        }
    }
Since match_phrase is only a type of a match query, it can also be used in the following manner:

    {
        "match" : {
            "message" : {
                "query" : "this is a test",
                "type" : "phrase"
            }
        }
    }

**match_phrase_prefix**
The match_phrase_prefix is the same as match_phrase, except that it allows for prefix matches on the last term in the text. For example:

    {
        "match_phrase_prefix" : {
            "message" : "quick brown f"
        }
    }

### 11.2.2 Multi Match Query
The multi_match query builds on the match query to allow multi-field queries:

    {
      "multi_match" : {
        "query":    "this is a test", 
        "fields": [ "subject^3", "message" ] 
      }
    }

Individual fields can be boosted with the caret (^) notation.

**Types of multi_match query:**
- best_fields: (default) Finds documents which match any field, but uses the _score from the best field. 
- most_fields: Finds documents which match any field and combines the _score from each field. 
- cross_fields: Treats fields with the same analyzer as though they were one big field. Looks for each word in any field. 
- phrase: Runs a match_phrase query on each field and combines the _score from each field. 
- phrase_prefix: Runs a match_phrase_prefix query on each field and combines the _score from each field. 

**best_fields**

    {
      "multi_match" : {
        "query":      "brown fox",
        "type":       "best_fields",
        "fields":     [ "subject^3", "message" ],
        "tie_breaker": 0.3
      }
    }

Normally the best_fields type uses the score of the single best matching field, but if tie_breaker is specified, then it calculates the score as follows:

- the score from the best matching field
- plus tie_breaker * _score for all other matching fields

Also, accepts analyzer, boost, operator, minimum_should_match, fuzziness, prefix_length, max_expansions, rewrite, zero_terms_query and cutoff_frequency, as explained in match query.

The best_fields and most_fields types are field-centric -- they generate a match query per field. This means that the operator and minimum_should_match parameters are applied to each field individually, which is probably not what you want.In other words, **all terms** must be present **in a single field** for a document to match.

**cross_fields**
The cross_fields type is particularly useful with structured documents where multiple fields should match. For instance, when querying the first_name and last_name fields for “Will Smith”, the best match is likely to have “Will” in one field and “Smith” in the other.

This sounds like a job for most_fields but there are two problems with that approach：
- The first problem is that operator and minimum_should_match are applied per-field, instead of per-term (see explanation above).
- The second problem is to do with relevance: the different term frequencies in the first_name and last_name fields can produce unexpected results.

In other words, **all terms** must be present **in at least one field** for a document to match. (Compare this to the logic used for best_fields and most_fields.)

One way of dealing with these types of queries is simply to index the first_name and last_name fields into a single full_name field. Of course, this can only be done at index time.

**most_fields**
The most_fields type is most useful when querying multiple fields that contain the same text analyzed in different ways. For instance, the main field may contain synonyms, stemming and terms without diacritics. A second field may contain the original terms, and a third field might contain shingles. By combining scores from all three fields we can match as many documents as possible with the main field, but use the second and third fields to push the most similar results to the top of the list.

**phrase and phrase_prefix**
The phrase and phrase_prefix types behave just like best_fields, but they use a match_phrase or match_phrase_prefix query instead of a match query.

### 11.2.3 Common Terms Query (Advanced Topic)
The common terms query is a modern alternative to stopwords which improves the precision and recall of search results (by taking stopwords into account), without sacrificing performance.

The common terms query divides the query terms into two groups: more important (ie low frequency terms) and less important (ie high frequency terms which would previously have been stopwords).

First it searches for documents which match the more important terms. These are the terms which appear in fewer documents and have a greater impact on relevance.

Then, it executes a second query for the less important terms -- terms which appear frequently and have a low impact on relevance. But instead of calculating the relevance score for all matching documents, it only calculates the _score for documents already matched by the first query. In this way the high frequency terms can improve the relevance calculation without paying the cost of poor performance.

### 11.2.4 Query String Query (Advanced Topic)
A query that uses a query parser in order to parse its content. Here is an example:

    {
        "query_string" : {
            "default_field" : "content",
            "query" : "this AND that OR thus"
        }
    }

### 11.2.5 Simple Query String Query (Advanced Topic)
A query that uses the SimpleQueryParser to parse its context. Unlike the regular query_string query, the simple_query_string query will never throw an exception, and discards invalid parts of the query. Here is an example:

    {
        "simple_query_string" : {
            "query": "\"fried eggs\" +(eggplant | potato) -frittata",
            "analyzer": "snowball",
            "fields": ["body^5","_all"],
            "default_operator": "and"
        }
    }

## 11.3 Term level queries
While the full text queries will analyze the query string before executing, the term-level queries operate on the exact terms that are stored in the inverted index.

These queries are usually used for structured data like numbers, dates, and enums, rather than full text fields. Alternatively, they allow you to craft low-level queries, foregoing the analysis process.

The queries in this group are:
- term query: Find documents which contain the exact term specified in the field specified.
- terms query: Find documents which contain any of the exact terms specified in the field specified.
- range query: Find documents where the field specified contains values (dates, numbers, or strings) in the range specified.
- exists query: Find documents where the field specified contains any non-null value.
- missing query: Find documents where the field specified does is missing or contains only null values.
- prefix query: Find documents where the field specified contains terms which begin with the exact prefix specified.
- wildcard query: Find documents where the field specified contains terms which match the pattern specified, where the pattern supports single character wildcards (?) and multi-character wildcards (*)
- regexp query: Find documents where the field specified contains terms which match the regular expression specified.
- fuzzy query: Find documents where the field specified contains terms which are fuzzily similar to the specified term. Fuzziness is measured as a Levenshtein edit distance of 1 or 2.
- type query: Find documents of the specified type.
- ids query: Find documents with the specified type and IDs.

### 11.3.1 Term Query

    {
      "query": {
        "term": {
          "productSkn": "51022624"
        }
      }
    }

A boost parameter can be specified to give this term query a higher relevance score than another query, for instance:

    GET /_search
    {
      "query": {
        "bool": {
          "should": [
            {
              "term": {
                "status": {
                  "value": "urgent",
                  "boost": 2.0 
                }
              }
            },
            {
              "term": {
                "status": "normal" 
              }
            }
          ]
        }
      }
    }

### 11.3.2 Terms Query
Filters documents that have fields that match any of the provided terms (not analyzed). For example:

    {
      "query": {
        "terms": {
          "brandId": [
            "144",
            "248"
          ]
        }
      }
    }

    更高效的方法是使用constant_score.filter
    {
        "constant_score." : {
            "filter" : {
                "terms" : { "brandId" : ["144", "248"]}
            }
        }
    }

    search on all the tweets that match the followers of user 2
    curl -XGET localhost:9200/tweets/_search -d '{
      "query" : {
        "terms" : {
          "user" : {
            "index" : "users",
            "type" : "user",
            "id" : "2",
            "path" : "followers"
          }
        }
      }
    }'

### 11.3.3 Range Query
Matches documents with fields that have terms within a certain range. The type of the Lucene query depends on the field type, for string fields, the TermRangeQuery, while for number/date fields, the query is a NumericRangeQuery. 

    {
      "query": {
        "range": {
          "salesNum": {
            "gte": "5",
            "lte": "100"
          }
        }
      }
    }

The range query accepts the following parameters:
- gte： Greater-than or equal to
- gt： Greater-than
- lte： Less-than or equal to
- lt： Less-than
- boost： Sets the boost value of the query, defaults to 1.0

### 11.3.4 Exists Query
Returns documents that have at least one non-null value in the original field:

    {
        "exists" : { "field" : "user" }
    }

The exists query can advantageously replace the missing query (Missing Query Deprecated in 2.2.0) when used inside a must_not clause as follows:

    "bool": {
        "must_not": {
            "exists": {
                "field": "user"
            }
        }
    }
This query returns documents that have no value in the user field.

### 11.3.5 Prefix Query
Matches documents that have fields containing terms with a specified prefix (not analyzed). The prefix query maps to Lucene PrefixQuery. The following matches documents where the user field contains a term that starts with ki:

    {
      "query": {
        "prefix": {
          "productName": "VA"
        }
      }
    }

### 11.3.6 Wildcard Query
Matches documents that have fields matching a wildcard expression (not analyzed). Supported wildcards are *, which matches any character sequence (including the empty one), and ?, which matches any single character. Note that this query can be slow, as it needs to iterate over many terms. In order to prevent extremely slow wildcard queries, a wildcard term should not start with one of the wildcards * or ?. The wildcard query maps to Lucene WildcardQuery.

    {
      "query": {
        "wildcard": {
          "productName": "VA*鞋*"
        }
      }
    }

### 11.3.7 Regexp Query
The regexp query allows you to use regular expression term queries. See Regular expression syntax for details of the supported regular expression language. The "term queries" in that first sentence means that Elasticsearch will apply the regexp to the terms produced by the tokenizer for that field, and not to the original text of the field.

Note: The performance of a regexp query heavily depends on the regular expression chosen. Matching everything like .* is very slow as well as using lookaround regular expressions. If possible, you should try to use a long prefix before your regular expression starts. Wildcard matchers like .*?+ will mostly lower performance.

    {
        "regexp":{
            "name.first": "s.*y"
        }
    }

### 11.3.8 Fuzzy Query
The fuzzy query uses similarity based on Levenshtein edit distance for string fields, and a +/- margin on numeric and date fields.

The fuzzy query generates all possible matching terms that are within the maximum edit distance specified in fuzziness and then checks the term dictionary to find out which of those generated terms actually exist in the index.

Here is a simple example:

    {
        "fuzzy" : { "user" : "ki" }
    }

### 11.3.9 Type Query
Filters documents matching the provided document / mapping type.

    {
        "type" : {
            "value" : "my_type"
        }
    }

### 11.3.10 Ids Query
Filters documents that only have the provided ids. Note, this query uses the _uid field.

    {
        "ids" : {
            "type" : "my_type",
            "values" : ["1", "4", "100"]
        }
    }
The type is optional and can be omitted, and can also accept an array of values. If no type is specified, all types defined in the index mapping are tried.

## 11.4 Compound queries
Compound queries wrap other compound or leaf queries, either to combine their results and scores, to change their behaviour, or to switch from query to filter context.

The queries in this group are:
- constant_score query: A query which wraps another query, but executes it in filter context. All matching documents are given the same “constant” _score.
- bool query: The default query for combining multiple leaf or compound query clauses, as must, should, must_not, or filter clauses. The must and should clauses have their scores combined -- the more matching clauses, the better -- while the must_not and filter clauses are executed in filter context.
- dis_max query: A query which accepts multiple queries, and returns any documents which match any of the query clauses. While the bool query combines the scores from all matching queries, the dis_max query uses the score of the single best- matching query clause.
- function_score query: Modify the scores returned by the main query with functions to take into account factors like popularity, recency, distance, or custom algorithms implemented with scripting.
- boosting query: Return documents which match a positive query, but reduce the score of documents which also match a negative query.
- indices query: Execute one query for the specified indices, and another for other indices.

### 11.4.1 Constant Score Query
A query that wraps another query and simply returns a constant score equal to the query boost for every document in the filter. Maps to Lucene ConstantScoreQuery.

    {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }

### 11.4.2 Bool Query
A query that matches documents matching boolean combinations of other queries. The bool query maps to Lucene BooleanQuery. It is built using one or more boolean clauses, each clause with a typed occurrence. The occurrence types are:

Occur	  |Description
----------|--------------
must      |The clause (query) must appear in matching documents and will contribute to the score.
filter    |The clause (query) must appear in matching documents. However unlike must the score of the query will be ignored.
should    |The clause (query) should appear in the matching document. In a boolean query with no must or filter clauses, one or more should clauses must match a document. The minimum number of should clauses to match can be set using the minimum_should_match parameter.
must_not  |The clause (query) must not appear in the matching documents.

**If this query is used in a filter context and it has should clauses then at least one should clause is required to match.**

The bool query takes a more-matches-is-better approach, so the score from each matching *must* or *should* clause will be added together to provide the final _score for each document.

    {
        "bool" : {
            "must" : {
                "term" : { "user" : "kimchy" }
            },
            "filter": {
                "term" : { "tag" : "tech" }
            },
            "must_not" : {
                "range" : {
                    "age" : { "from" : 10, "to" : 20 }
                }
            },
            "should" : [
                {
                    "term" : { "tag" : "wow" }
                },
                {
                    "term" : { "tag" : "elasticsearch" }
                }
            ],
            "minimum_should_match" : 1,
            "boost" : 1.0
        }
    }

### 11.4.3 Dis Max Query
A query that generates the union of documents produced by its subqueries, and that scores each document with the maximum score for that document as produced by any subquery, plus a tie breaking increment for any additional matching subqueries.

This is useful when searching for a word in multiple fields with different boost factors (so that the fields cannot be combined equivalently into a single search field). We want the primary score to be the one associated with the highest boost, not the sum of the field scores (as Boolean Query would give). If the query is "albino elephant" this ensures that "albino" matching one field and "elephant" matching another gets a higher score than "albino" matching both fields. To get this result, use both Boolean Query and DisjunctionMax Query: for each term a DisjunctionMaxQuery searches for it in each field, while the set of these DisjunctionMaxQuery’s is combined into a BooleanQuery.

The tie breaker capability allows results that include the same term in multiple fields to be judged better than results that include this term in only the best of those multiple fields, without confusing this with the better case of two different terms in the multiple fields.The default tie_breaker is 0.0.

This query maps to Lucene DisjunctionMaxQuery.

    {
        "dis_max" : {
            "tie_breaker" : 0.7,
            "boost" : 1.2,
            "queries" : [
                {
                    "term" : { "age" : 34 }
                },
                {
                    "term" : { "age" : 35 }
                }
            ]
        }
    }

### 11.4.4 Function Score Query
The function_score allows you to modify the score of documents that are retrieved by a query. This can be useful if, for example, a score function is computationally expensive and it is sufficient to compute the score on a filtered set of documents.

To use function_score, the user has to define a query and one or more functions, that compute a new score for each document returned by the query.

    "function_score": {
        "query": {},
        "boost": "boost for the whole query",
        "functions": [
            {
                "filter": {},
                "FUNCTION": {}, 
                "weight": number
            },
            {
                "FUNCTION": {} 
            },
            {
                "filter": {},
                "weight": number
            }
        ],
        "max_boost": number,
        "score_mode": "(multiply|max|...)",
        "boost_mode": "(multiply|replace|...)",
        "min_score" : number
    }

++*The* scores produced by the filtering query of each function do not matter.++

First, each document is scored by the defined functions. The parameter score_mode specifies how the computed scores are combined:
- multiply: scores are multiplied (default)
- sum: scores are summed
- avg: scores are averaged
- first: the first function that has a matching filter is applied
- max: maximum score is used
- min: minimum score is used

The newly computed score is combined with the score of the query. The parameter boost_mode defines how:
- multiply: query score and function score is multiplied (default)
- replace: only function score is used, the query score is ignored
- sum: query score and function score are added
- avg: average
- max: max of query score and function score
- min: min of query score and function score

The function_score query provides several types of score functions: 
- script_score
- weight
- random_score
- field_value_factor
- decay functions: gauss, linear, exp

**Field Value factor**
The field_value_factor function allows you to use a field from a document to influence the score. It’s similar to using the script_score function, however, it avoids the overhead of scripting. If used on a multi-valued field, only the first value of the field is used in calculations.

As an example, imagine you have a document indexed with a numeric popularity field and wish to influence the score of a document with this field, an example doing so would look like:

    "field_value_factor": {
      "field": "popularity",
      "factor": 1.2,
      "modifier": "sqrt",
      "missing": 1
    }
Which will translate into the following formula for scoring:

    sqrt(1.2 * doc['popularity'].value)

There are a number of options for the field_value_factor function:
- field: Field to be extracted from the document.
- factor: Optional factor to multiply the field value with, defaults to 1.
- modifier: Modifier to apply to the field value, can be one of: none, log, log1p, log2p, ln, ln1p, ln2p, square, sqrt, or reciprocal. Defaults to none.

Modifier  |	Meaning
----------|-------------
none      |Do not apply any multiplier to the field value
log       |Take the logarithm of the field value
log1p     |Add 1 to the field value and take the logarithm
log2p     |Add 2 to the field value and take the logarithm
ln        |Take the natural logarithm of the field value
ln1p      |Add 1 to the field value and take the natural logarithm
ln2p      |Add 2 to the field value and take the natural logarithm
square    |Square the field value (multiply it by itself)
sqrt      |Take the square root of the field value
reciprocal|Reciprocate the field value, same as 1/x where x is the field’s value

### 11.4.5 Boosting Query
The boosting query can be used to effectively demote results that match a given query. Unlike the "NOT" clause in bool query, this still selects documents that contain undesirable terms, but reduces their overall score.

    {
        "boosting" : {
            "positive" : {
                "term" : {
                    "field1" : "value1"
                }
            },
            "negative" : {
                "term" : {
                    "field2" : "value2"
                }
            },
            "negative_boost" : 0.2
        }
    }

### 11.4.6 Indices Query
The indices query is useful in cases where a search is executed across multiple indices. It allows to specify a list of index names and an inner query that is only executed for indices matching names on that list. For other indices that are searched, but that don’t match entries on the list, the alternative no_match_query is executed.

    {
        "indices" : {
            "indices" : ["index1", "index2"],
            "query" : {
                "term" : { "tag" : "wow" }
            },
            "no_match_query" : {
                "term" : { "tag" : "kow" }
            }
        }
    }

## 11.5 Joining queries
Performing full SQL-style joins in a distributed system like Elasticsearch is prohibitively expensive. Instead, Elasticsearch offers two forms of join which are designed to scale horizontally.

- nested query: Documents may contains fields of type nested. These fields are used to index arrays of objects, where each object can be queried (with the nested query) as an independent document.
- has_child and has_parent queries: A parent-child relationship can exist between two document types within a single index. The has_child query returns parent documents whose child documents match the specified query, while the has_parent query returns child documents whose parent document matches the specified query.
 
Also see the terms-lookup mechanism in the terms query, which allows you to build a terms query from values contained in another document.

## 11.6 Geo queries  (Advanced Topic)
## 11.7 Specialized queries (Advanced Topic)
This group contains queries which do not fit into the other groups:

- more_like_this query: This query finds documents which are similar to the specified text, document, or collection of documents.
- template query: The template query accepts a Mustache template (either inline, indexed, or from a file), and a map of parameters, and combines the two to generate the final query to execute.
- script query: This query allows a script to act as a filter. Also see the function_score query.

## 11.8 Span queries (Advanced Topic)
Span queries are low-level positional queries which provide expert control over the order and proximity of the specified terms. These are typically used to implement very specific queries on legal documents or patents.

Span queries cannot be mixed with non-span queries (with the exception of the span_multi query).

The queries in this group are:

- span_term query: The equivalent of the term query but for use with other span queries.
- span_multi query: Wraps a term, range, prefix, wildcard, regexp, or fuzzy query.
- span_first query: Accepts another span query whose matches must appear within the first N positions of the field.
- span_near query: Accepts multiple span queries whose matches must be within the specified distance of each other, and possibly in the same order.
- span_or query: Combines multiple span queries -- returns documents which match any of the specified queries.
- span_not query: Wraps another span query, and excludes any documents which match that query.
- span_containing query: Accepts a list of span queries, but only returns those spans which also match a second span query.
- span_within query: The result from a single span query is returned as long is its span falls within the spans returned by a list of other span queries.

# 12 Mapping
Mapping is the process of defining how a document, and the fields it contains, are stored and indexed. For instance, use mappings to define:

    which string fields should be treated as full text fields.
    which fields contain numbers, dates, or geolocations.
    whether the values of all fields in the document should be indexed into the catch-all _all field.
    the format of date values.
    custom rules to control the mapping for dynamically added fields.

**Mapping Types**
Each index has one or more mapping types, which are used to divide the documents in an index into logical groups. User documents might be stored in a user type, and blog posts in a blogpost type.

Each mapping type has:

- Meta-fields: Meta-fields are used to customize how a document’s metadata associated is treated. Examples of meta-fields include the document’s _index, _type, _id, and _source fields.
- Fields or properties: Each mapping type contains a list of fields or properties pertinent to that type. A user type might contain title, name, and age fields, while a blogpost type might contain title, body, user_id and created fields. Fields with the same name in different mapping types in the same index must have the same mapping.

**Field datatypes**
Each field has a data type which can be:

    a simple type like string, date, long, double, boolean or ip.
    a type which supports the hierarchical nature of JSON such as object or nested.
    or a specialised type like geo_point, geo_shape, or completion.
    
It is often useful to index the same field in different ways for different purposes. For instance, a string field could be indexed as an analyzed field for full-text search, and as a not_analyzed field for sorting or aggregations. Alternatively, you could index a string field with the standard analyzer, the english analyzer, and the french analyzer.

This is the purpose of multi-fields. Most datatypes support multi-fields via the fields parameter.

**Dynamic mapping**
Fields and mapping types do not need to be defined before being used. Thanks to dynamic mapping, new mapping types and new field names will be added automatically, just by indexing a document. New fields can be added both to the top-level mapping type, and to inner object and nested fields.

The dynamic mapping rules can be configured to customise the mapping that is used for new types and new fields.

**Explicit mappings**
You know more about your data than Elasticsearch can guess, so while dynamic mapping can be useful to get started, at some point you will want to specify your own explicit mappings.

You can create mapping types and field mappings when you create an index, and you can add mapping types and fields to an existing index with the PUT mapping API.

**Updating existing mappings**
Other than where documented, **existing type and field mappings cannot be updated**. Changing the mapping would mean invalidating already indexed documents. Instead, you should create a new index with the correct mappings and reindex your data into that index.

**Fields are shared across mapping types**
Mapping types are used to group fields, but the fields in each mapping type are not independent of each other. Fields with:

    the same name
    in the same index
    in different mapping types
    map to the same field internally,
    and must have the same mapping.
    
If a title field exists in both the user and blogpost mapping types, the title fields must have exactly the same mapping in each type. The only exceptions to this rule are the copy_to, dynamic, enabled, ignore_above, include_in_all, and properties parameters, which may have different settings per field.

Usually, fields with the same name also contain the same type of data, so having the same mapping is not a problem. When conflicts do arise, these can be solved by choosing more descriptive names, such as user_title and blog_title.

**Example mapping**
A mapping for the example described above could be specified when creating the index, as follows:

    PUT my_index 
    {
      "mappings": {
        "user": { 
          "_all":       { "enabled": false  }, 
          "properties": { 
            "title":    { "type": "string"  }, 
            "name":     { "type": "string"  }, 
            "age":      { "type": "integer" }  
          }
        },
        "blogpost": { 
          "properties": { 
            "title":    { "type": "string"  }, 
            "body":     { "type": "string"  }, 
            "user_id":  {
              "type":   "string", 
              "index":  "not_analyzed"
            },
            "created":  {
              "type":   "date", 
              "format": "strict_date_optional_time||epoch_millis"
            }
          }
        }
      }
    }

## 12.1 Field datatypes
Elasticsearch supports a number of different datatypes for the fields in a document:

### Core datatypes
**String datatype: string**
The following parameters are accepted by string fields:

Parameter    | Description
-------------|----------------------------------
analyzer     |The analyzer which should be used for analyzed string fields, both at index-time and at search-time (unless overridden by the search_analyzer). Defaults to the default index analyzer, or the standard analyzer.
boost        |Field-level index time boosting. Accepts a floating point number, defaults to 1.0.
doc_values   |Should the field be stored on disk in a column-stride fashion, so that it can later be used for sorting, aggregations, or scripting? Accepts true or false. Defaults to true for not_analyzed fields. Analyzed fields do not support doc values.
fielddata    |Can the field use in-memory fielddata for sorting, aggregations, or scripting? Accepts disabled or paged_bytes (default). Not analyzed fields will use doc values in preference to fielddata.
fields       |Multi-fields allow the same string value to be indexed in multiple ways for different purposes, such as one field for search and a multi-field for sorting and aggregations, or the same string value analyzed by different analyzers.
ignore_above |Do not index or analyze any string longer than this value. Defaults to 0 (disabled).
include_in_all|Whether or not the field value should be included in the _all field? Accepts true or false. Defaults to false if index is set to no, or if a parent object field sets include_in_all to false. Otherwise defaults to true.
index        |Should the field be searchable? Accepts analyzed (default, treat as full-text field), not_analyzed (treat as keyword field) and no.
index_options|What information should be stored in the index, for search and highlighting purposes. Defaults to positions for analyzed fields, and to docs for not_analyzed fields.
norms        |Whether field-length should be taken into account when scoring queries. Defaults depend on the index setting: analyzed fields default to { "enabled": true, "loading": "lazy" }; not_analyzed fields default to { "enabled": false }.
null_value   |Accepts a string value which is substituted for any explicit null values. Defaults to null, which means the field is treated as missing. If the field is analyzed, the null_value will also be analyzed.
position_increment_gap|The number of fake term position which should be inserted between each element of an array of strings. Defaults to the position_increment_gap configured on the analyzer which defaults to 100. 100 was chosen because it prevents phrase queries with reasonably large slops (less than 100) from matching terms across field values.
store        |Whether the field value should be stored and retrievable separately from the _source field. Accepts true or false (default).
search_analyzer|The analyzer that should be used at search time on analyzed fields. Defaults to the analyzer setting.
search_quote_analyzer|The analyzer that should be used at search time when a phrase is encountered. Defaults to the search_analyzer setting.
similarity  |Which scoring algorithm or similarity should be used. Defaults to default, which uses TF/IDF.
term_vector |Whether term vectors should be stored for an analyzed field. Defaults to no.

**Numeric datatypes: long, integer, short, byte, double, float**

**Date datatype: date **
JSON doesn’t have a date datatype, so dates in Elasticsearch can either be:
- strings containing formatted dates, e.g. "2015-01-01" or "2015/01/01 12:10:30"
- a long number representing milliseconds-since-the-epoch.
- an integer representing seconds-since-the-epoch.)

**Boolean datatype: boolean**
**Binary datatype: binary **
The binary type accepts a binary value as a Base64 encoded string. The field is not stored by default and is not searchable.
 
### Complex datatypes
**Array datatype**
Array support does not require a dedicated type (In Elasticsearch, there is no dedicated array type. Any field can contain zero or more values by default, however, all values in the array must be of the same datatype.)

**Object datatype: object for single JSON objects**
**Nested datatype: nested for arrays of JSON objects**

### Geo datatypes (Advanced Topic)
- Geo-point datatype: geo_point for lat/lon points
- Geo-Shape datatype: geo_shape for complex shapes like polygons

Specialised datatypes
- IPv4 datatype： ip for IPv4 addresses
- Completion datatype： completion to provide auto-complete suggestions
- Token count datatype： token_count to count the number of tokens in a string
- mapper-murmur3： murmur3 to compute hashes of values at index-time and store them in the index
- Attachment datatype： See the mapper-attachments plugin which supports indexing attachments like Microsoft Office formats, Open Document formats, ePub, HTML, etc. into an attachment datatype.

### Multi-fields

It is often useful to index the same field in different ways for different purposes. For instance, a string field could be indexed as an analyzed field for full-text search, and as a not_analyzed field for sorting or aggregations. Alternatively, you could index a string field with the standard analyzer, the english analyzer, and the french analyzer.

This is the purpose of multi-fields. Most datatypes support multi-fields via the fields parameter.

## 12.2 Meta-Fields
Each document has metadata associated with it, such as the _index, mapping _type, and _id meta-fields. The behaviour of some of these meta-fields can be customised when a mapping type is created.

### Identity meta-fields
- _index: The index to which the document belongs.
- _uid: A composite field consisting of the _type and the _id.
- _type: The document’s mapping type.
- _id: The document’s ID.

### Document source meta-fields
- _source: The original JSON representing the body of the document.
- _size: The size of the _source field in bytes, provided by the mapper-size plugin.

### Indexing meta-fields
- _all: The _all field is a special catch-all field which concatenates the values of all of the other fields into one big string, using space as a delimiter, which is then analyzed and indexed, but not stored. This means that it can be searched, but not retrieved; The _all field can be completely disabled per-type by setting enabled to false; While there is only a single _all field per index, the copy_to parameter allows the creation of multiple custom _all fields. For instance, first_name and last_name fields can be combined together into the full_name field.

- _field_names: The _field_names field indexes the names of every field in a document that contains any value other than null. This field is used by the exists and missing queries to find documents that either have or don’t have any non-null value for a particular field; The value of the _field_name field is accessible in queries, aggregations, and scripts.

### Routing meta-fields
- _parent: Used to create a parent-child relationship between two mapping types.
- _routing: A custom routing value which routes a document to a particular shard.

### Other meta-field
- _meta: Application specific metadata.

## 12.3 Mapping parameters
The following mapping parameters are common to some or all field datatypes:

- analyzer
- boost
- coerce
- copy_to: The copy_to parameter allows you to create custom _all fields. In other words, the values of multiple fields can be copied into a group field, which can then be queried as a single field. For instance, the first_name and last_name fields can be copied to the full_name field.
- doc_values: Doc values are the on-disk data structure, built at document index time, which makes this data access pattern possible. They store the same values as the _source but in a column-oriented fashion that is way more efficient for sorting and aggregations. Doc values are supported on almost all field types, with the notable exception of analyzed string fields.
- dynamic
- enabled
- fielddata: Most fields can use index-time, on-disk doc_values to support this type of data access pattern, but analyzed string fields do not support doc_values; Instead, analyzed strings use a query-time data structure called fielddata. This data structure is built on demand the first time that a field is used for aggregations, sorting, or is accessed in a script. It is built by reading the entire inverted index for each segment from disk, inverting the term ↔︎ document relationship, and storing the result in memory, in the JVM heap; Loading fielddata is an expensive process so, once it has been loaded, it remains in memory for the lifetime of the segment.
- geohash
- geohash_precision
- geohash_prefix
- format
- ignore_above
- ignore_malformed
- include_in_all
- index_options
- lat_lon
- index
- fields: It is often useful to index the same field in different ways for different purposes. This is the purpose of multi-fields. For instance, a string field could be indexed as an analyzed field for full-text search, and as a not_analyzed field for sorting or aggregations. 
- norms: Norms store various normalization factors -- a number to represent the relative field length and the index time boost setting -- that are later used at query time in order to compute the score of a document relatively to a query; Although useful for scoring, norms also require quite a lot of memory (typically in the order of one byte per document per field in your index, even for documents that don’t have this specific field). As a consequence, if you don’t need scoring on a specific field, you should disable norms on that field. In particular, this is the case for fields that are used solely for filtering or aggregations.
- null_value
- position_increment_gap
- properties: Type mappings, object fields and nested fields contain sub-fields, called properties. These properties may be of any datatype, including object and nested. 
- search_analyzer
- similarity
- store: By default, field values are indexed to make them searchable, but they are not stored. This means that the field can be queried, but the original field value cannot be retrieved; Usually this doesn’t matter. The field value is already part of the _source field, which is stored by default. If you only want to retrieve the value of a single field or of a few fields, instead of the whole _source, then this can be achieved with source filtering; In certain situations it can make sense to store a field. For instance, if you have a document with a title, a date, and a very large content field, you may want to retrieve just the title and the date without having to extract those fields from a large _source field.
- term_vector: Term vectors contain information about the terms produced by the analysis process, including: a list of terms/the position (or order) of each term/the start and end character offsets mapping the term to its origin in the original string; These term vectors can be stored so that they can be retrieved for a particular document.

The term_vector setting accepts:
- no: No term vectors are stored. (default)
- yes: Just the terms in the field are stored.
- with_positions: Terms and positions are stored.
- with_offsets: Terms and character offsets are stored.
- with_positions_offsets: Terms, positions, and character offsets are stored.

## 12.4 Dynamic Mapping
One of the most important features of Elasticsearch is that it tries to get out of your way and let you start exploring your data as quickly as possible. To index a document, you don’t have to first create an index, define a mapping type, and define your fields -- you can just index a document and the index, type, and fields will spring to life automatically:

    PUT data/counters/1 
    { "count": 5 }

Creates the data index, the counters mapping type, and a field called count with datatype long.

The automatic detection and addition of new types and fields is called dynamic mapping. The dynamic mapping rules can be customised to suit your purposes with:

- _default_ mapping: Configure the base mapping to be used for new mapping types.
- Dynamic field mappings: The rules governing dynamic field detection.
- Dynamic templates: Custom rules to configure the mapping for dynamically added fields.

# 13 Analysis (See another documents)
# 14 Modules (Skipped)
# 15 Index Modules (Skipped)
# 16 Testing (Skipped)
# 17 Glossary of terms (Skipped)


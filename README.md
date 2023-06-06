# ElasticSearch Essentials

## 1. Setup
### Exploring the cluster

Health of the cluster:
```
GET /_cat/health?v
```
<details>
    <summary>Explanation:</summary>
    1. GET: This is an HTTP method indicating that we want to retrieve information from the server.</br>
    2. /_cat: The "_cat" endpoint in Elasticsearch provides a way to access various cluster-related information in a simple and tabular format.</br>
    3. /health: This is the specific cat API endpoint used to retrieve the health information of the cluster.</br>
    4. ?v: The "v" parameter stands for "verbose" and is used to include additional details in the response. It helps in displaying more information about the cluster's health.
</details>

List of all nodes:
```
GET /_cat/nodes?v
```

**Def.** An **index** is a logical namespace which maps to one or more primary shards and can have zero or more replica shards.  
List of indices.
```
GET /_cat/indices?v
```

Create an index called `sales`:
```
PUT /sales
```

**Def.** The **documents** are JSON objects that are stored within an Elasticsearch index and are considered the base unit of storage. 
Create a document in `sales` index:
```
    PUT /sales/_doc/123
    {
        "orderID":"123",
        "orderAmount":"500"
    }
```

Retrieve the document data:
```
    GET /sales/_doc/123
```

Delete `sales` index:
```
    DELETE /sales
```

## 2. Loading Data
### Bulk Loading

Example file with requests `reqs`:
{ "index" : { "_index" : "my-test", "_id" : "1" } }
{ "col1" : "val1"}
{ "index" : { "_index" : "my-test", "_id" : "2" } }
{ "col1" : "val2"}
{ "index" : { "_index" : "my-test", "_id" : "3" } }
{ "col1" : "val3" }

using curl, upload data file to cluster and create index `reqs`
```
curl -XPOST -i -k \
-H "Content-Type: application/x-ndjson" \
-H "Authorization: ApiKey $ES_API_KEY" \
$ES_HOST/_bulk --data-binary "@reqs"; echo  
```    

Verify that the index was created
```
GET /_cat/indices?v
GET /my-test
GET /my-test/_doc/1
```

### Setting data types
Configure the mapping so that the location of the request will be correctly interpreted as a `geo_point`.

```
PUT /logstash-2015.05.18
```

```
PUT /logstash-2015.05.18/log/_mapping?include_type_name=true
{
    "properties":{
       "geo":{
          "properties":{
             "coordinates":{
                "type":"geo_point"
             }
          }
       }
    }  
}
```

## 3. Querying Data
### Simple queries

Examples of some queries to `bank` index.
Show all the documents:
```
GET /bank/_search
```

Show only data of accounts in California
```
GET /bank/_search
{
    "query": {
        "match": {
            "state": "CA"
        }
    }
}
```

Show only data of accounts in California and employer Techade
```
GET /bank/_search
{
    "query":{
        "bool":{
            "must": [
                {"match": {"state":"CA"}},
                {"match": {"employer":"Techade"}}
            ]
        }
    }
}
```

Show accounts that are not in California and employer not Techade
```
GET /bank/_search
{
    "query":{
        "bool":{
            "must_not": [
                {"match": {"state":"CA"}},
                {"match": {"employer":"Techade"}}
            ]
        }
    }
}
```

Show accounts that are in California and employer not Techade
```
GET /bank/_search
{
    "query":{
        "bool":{
            "must": [
                {"match": {"state":"CA"}
                }
            ],

            "must_not": [
                {"match": {"employer":"Techade"}
                }
            ]
        }
    }
}
```

The ability to boost the matches for a particular field, while using the `should` clause:
```
GET /bank/_search
{
    "query":{
        "bool":{
            "should": [
                {
                    "match":{"state":"CA"}
                },
                {
                    "match": {
                        "lastname": {
                            "query": "Smith",
                            "boost": 3
                        }
                    }
                }
            ]
        }
    }
    
}
```

### Term-level queries

Searching - using the `term`:
```
GET /bank/_search
{
    "query": {
        "term":{
            "account_number": 516
        }
    }
}
```

Note that the `account_number` is numeric.
Let's try the same approach with the `state` field.
```
GET /bank/_search
{
    "query": {
        "term":{
            "state": "RI"
        }
    }
}
```
However, this time we get no matches.

Because strings are not supported for `term` searches;
we have to use the `match` type query.

We can use multiple `terms`:
```
GET /bank/_search
{
    "query": {
        "terms":{
            "account_number": [516,851]
        }
    }
}
```

We can also use ranges for term-eligible fields.
For example, for account_number between those two, inclusively.
gte - greater than or equal to
lte - less than or equal to
```
GET /bank/_search
{
    "query": {
        "range":{
            "account_number": {
                "gte": 516,
                "lte": 851,
                "boost": 2
            }
        }
    }
}
```

Get all accounts where the holder is older than 35:
```
GET /bank/_search
{
    "query": {
        "range":{
            "age": {
                "gte": 35,
            }
        }
    }
}
```

### Analysis and tokenization
```
GET /bank/_analyze
{
    "tokenizer":"standard",
    "text": "The Moon is Made of Cheese Some Say"
}
```

```
GET bank/_analyze
{
  "tokenizer" : "standard",
  "text" : "The Moon-is-Made of Cheese.Some Say$"
}
```

types are `word` and words were parsed as separate words
```
GET bank/_analyze
{
  "tokenizer" : "letter",
  "text" : "The Moon-is-Made of Cheese.Some Say$"
}
```

parse email and url with types `email` and `url`
```
GET bank/_analyze
{
  "tokenizer": "uax_url_email",
  "text": "you@example.com login at https://bensullins.com attempt"
}
```

Now we'll make a new index, specifying the types of the two fields:
```
PUT /idx1
{
   "mappings":{
      "properties":{
         "title":{
            "type":"text",
            "analyzer":"standard"
         },
         "english_title":{
            "type":"text",
            "analyzer":"english"
         }
      }
   }
}
```

```
GET idx1/_analyze
{
  "field": "title",
  "text": "Bears"
}
```
Note that the token = `bears`.

```
GET idx1/_analyze
{
  "field": "english_title",
  "text": "Bears"
}
```
We get a token = `bear`, which it knows to be the root form.
This is because the `english_title` has analyzer "english"

## 4. Analyzing your data
### Basic aggregations

Get the count of accounts by state:
```
GET bank/_search
{
    "size": 0,
    "aggs": {
        "states": {
            "terms": {"field": "state.keyword"}
        }
    }
}
```

The average balance is by state:
```
GET bank/_search
{
    "size": 0,
    "aggs": {
        "states": {
            "terms": {"field": "state.keyword"},
            "aggs": {
                "avg_bal": {
                    "avg": {"field":"balance"}
                }
            }
        }
    }
}
```

The average balance is by state
And average balance by state and age:
```
GET bank/_search
{
    "size": 0,
    "aggs": {
        "states": {
            "terms": {"field": "state.keyword"},
            "aggs": {
                "avg_bal": {
                    "avg": {"field":"balance"}
                }
            },
            "age":{
                "terms":{
                    "field": "age"
                }
            },
            "aggs": {
                "avg_bal": {
                    "avg": {"field":"balance"}
                }
            }
        }
    }
}
```

```
GET bank/_search
{
    "size": 0
    "aggs": {
        "balance-stats": {
            "stats": {
                "field": "balance"
            }
        }
    }
}
```
Result:
```
...
    "balance-stats" : {
      "count" : 1000,
      "min" : 1011.0,
      "max" : 49989.0,
      "avg" : 25714.837,
      "sum" : 2.5714837E7
    }
...
```

### Filtering aggregations

Get the count of accounts by state
And filter to only California:
```
GET bank/_search
{
    "size": 0,
    "query": {
        "match": {
            "state.keyword":"CA"
        }
    },
    "aggs": {
        "states": {
            "terms": {
                "field":"state.keyword"
            }
        }
    }
}
```

```
GET bank/_search
{
    "size": 0,
    "query": {
        "bool":{
            "must":[
                {"match": {"state.keyword":"CA"}},
                {"range": {"age": {"gt": 35}}}
            ]
        }
    },
    "aggs": {
        "states": {
            "terms": {
                "field":"state.keyword"
            }
        },
        "aggs": {
            "avg_bal": {
                "avg": {"field":"balance"}
            }
        }
    }
}
```

Add a global average balance:
```
GET bank/account/_search
{
  "size": 0,
  "aggs": {
    "state_avg": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {"avg_bal": {"avg": {"field": "balance"}}}
    },
    "global_avg": {
      "global": {},
      "aggs": {"avg_bal": {"avg": {"field": "balance"}}}
    }
  }
}
```

### Percentiles and histograms

The balances are distributed across percentiles:
```
GET bank/_serach
{
    "size": 0,
    "aggs": {
        "pct_balances": {
            "percentiles": {
                "field": "balance",
                "percents": [
                    1,
                    5,
                    25,
                    50,
                    75,
                    95,
                    99
                ]
            }
        }
    }
}
```

We can also identify at which percentile specific balances occur:
```
GET bank/_search_
{
    "size":0,
    "aggs": {
        "bal_outlier": {
            "percentile_ranks": {
                "field": "balance",
                "values": [35000, 50000]
            }
        }
    }
}
```

Create data for histogram:
```
GET bank/_search
{
    "size": 0,
    "aggs": {
        "bals": {
            "hitsogram": {
                "field": "balance",
                "interval": 500
            }
        }
    }
}
```
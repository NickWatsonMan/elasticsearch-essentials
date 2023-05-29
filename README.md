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
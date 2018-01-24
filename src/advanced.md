# Advanced

---

## GridFS Support

Monstache supports indexing the raw content of files stored in GridFS into Elasticsearch for full
text search.  This feature requires that you install an Elasticsearch plugin which enables the field type `attachment`.
For versions of Elasticsearch prior to version 5 you should install the 
[mapper-attachments](https://www.elastic.co/guide/en/elasticsearch/plugins/2.4/mapper-attachments.html) plugin.
For version 5 or later of Elasticsearch you should instead install the 
[ingest-attachment](https://www.elastic.co/guide/en/elasticsearch/plugins/current/ingest-attachment.html) plugin.

Once you have installed the appropriate plugin for Elasticsearch, getting file content from GridFS into Elasticsearch is
as simple as configuring monstache.  You will want to enable the [index-files](/config#index-files) option and also tell monstache the 
namespace of all collections which will hold GridFS files. For example in your TOML config file,

```
index-files = true

file-namespaces = ["users.fs.files", "posts.fs.files"]

file-highlighting = true
```

The above configuration tells monstache that you wish to index the raw content of GridFS files in the `users` and `posts`
MongoDB databases. By default, MongoDB uses a bucket named `fs`, so if you just use the defaults your collection name will
be `fs.files`.  However, if you have customized the bucket name, then your file collection would be something like `mybucket.files`
and the entire namespace would be `users.mybucket.files`.

When you configure monstache this way it will perform an additional operation at startup to ensure the destination indexes in
Elasticsearch have a field named `file` with a type mapping of `attachment`.  

For the example TOML configuration above, monstache would initialize 2 indices in preparation for indexing into
Elasticsearch by issuing the following REST commands:

For Elasticsearch versions prior to version 5...

	POST /users.fs.files
	{
	  "mappings": {
	    "fs.files": {
	      "properties": {
		"file": { "type": "attachment" }
	}}}}

	POST /posts.fs.files
	{
	  "mappings": {
	    "fs.files": {
	      "properties": {
		"file": { "type": "attachment" }
	}}}}

For Elasticsearch version 5 and above...

	PUT /_ingest/pipeline/attachment
	{
	  "description" : "Extract file information",
	  "processors" : [
	    {
	      "attachment" : {
		"field" : "file"
	      }
	    }
	  ]
	}

When a file is inserted into MongoDB via GridFS, monstache will detect the new file, use the MongoDB api to retrieve the raw
content, and index a document into Elasticsearch with the raw content stored in a `file` field as a base64 
encoded string. The Elasticsearch plugin will then extract text content from the raw content using 
[Apache Tika](https://tika.apache.org/), tokenize the text content, and allow you to query on the content of the file.

To test this feature of monstache you can simply use the [mongofiles](https://docs.mongodb.com/manual/reference/program/mongofiles/)
command to quickly add a file to MongoDB via GridFS.  Continuing the example above one could issue the following command to put a 
file named `resume.docx` into GridFS and after a short time this file should be searchable in Elasticsearch in the index `users.fs.files`.

	mongofiles -d users put resume.docx

After a short time you should be able to query the contents of resume.docx in the users index in Elasticsearch

	curl -XGET "http://localhost:9200/users.fs.files/_search?q=golang"

If you would like to see the text extracted by Apache Tika you can project the appropriate sub-field

For Elasticsearch versions prior to version 5...

	curl -H "Content-Type:application/json" localhost:9200/users.fs.files/_search?pretty -d '{
		"fields": [ "file.content" ],
		"query": {
			"match": {
				"_all": "golang"
			}
		}
	}'

For Elasticsearch version 5 and above...

	curl -H "Content-Type:application/json" localhost:9200/users.fs.files/_search?pretty -d '{
		"_source": [ "attachment.content" ],
		"query": {
			"match": {
				"_all": "golang"
			}
		}
	}'

When [file-highlighting](/config#file-highlighting) is enabled you can add a highlight clause to your query

For Elasticsearch versions prior to version 5...

	curl -H "Content-Type:application/json" localhost:9200/users.fs.files/_search?pretty -d '{
		"fields": ["file.content"],
		"query": {
			"match": {
				"file.content": "golang"
			}
		},
		"highlight": {
			"fields": {
				"file.content": {
				}
			}
		}
	}'

For Elasticsearch version 5 and above...

	curl -H "Content-Type:application/json" localhost:9200/users.fs.files/_search?pretty -d '{
		"_source": ["attachment.content"],
		"query": {
			"match": {
				"attachment.content": "golang"
			}
		},
		"highlight": {
			"fields": {
				"attachment.content": {
				}
			}
		}
	}'


The highlight response will contain emphasis on the matching terms

For Elasticsearch versions prior to version 5...

	"hits" : [ {
		"highlight" : {
			"file.content" : [ "I like to program in <em>golang</em>.\n\n" ]
		}
	} ]

For Elasticsearch version 5 and above...

	"hits" : [{
		"highlight" : {
			"attachment.content" : [ "I like to program in <em>golang</em>." ]
		}
	}]


## Workers


You can run multiple monstache processes and distribute the work between them.  First configure
the names of all the workers in a shared config.toml file.

```toml
workers = ["Tom", "Dick", "Harry"]
```

In this case we have 3 workers.  Now we can start 3 monstache processes and give each one of the worker
names.

	monstache -f config.toml -worker Tom
	monstache -f config.toml -worker Dick
	monstache -f config.toml -worker Harry

monstache will hash the id of each document using consistent hashing so that each id is handled by only
one of the available workers.


## High Availability


You can run monstache in high availability mode by starting multiple processes with the same value for [cluster-name](/config#cluster-name).
Each process will join a cluster which works together to ensure that a monstache process is always syncing to Elasticsearch.

High availability works by ensuring a active process in the `monstache.cluster` collection in mongodb. Only the processes in
this collection will be syncing for the cluster.  Processes not present in this collection will be paused.  Documents in the 
`monstache.cluster` collection have a TTL assigned to them.  When a document in this collection times out it will be removed from
the collection by mongodb and another process in the monstache cluster will have a chance to write to the collection and become the
new active process.

When [cluster-name](/config#cluster-name) is supplied the [resume](/config#resume) feature is automatically turned on and the [resume-name](/config#resume-name) becomes the name of the cluster.
This is to ensure that each of the processes is able to pick up syncing where the last one left off.  

You can combine the HA feature with the workers feature.  For 3 cluster nodes with 3 workers per node you would have something like the following:

	// config.toml
	workers = ["Tom", "Dick", "Harry"]
	
	// on host A
	monstache -cluster-name HA -worker Tom -f config.toml
	monstache -cluster-name HA -worker Dick -f config.toml
	monstache -cluster-name HA -worker Harry -f config.toml

	// on host B
	monstache -cluster-name HA -worker Tom -f config.toml
	monstache -cluster-name HA -worker Dick -f config.toml
	monstache -cluster-name HA -worker Harry -f config.toml

	// on host C
	monstache -cluster-name HA -worker Tom -f config.toml
	monstache -cluster-name HA -worker Dick -f config.toml
	monstache -cluster-name HA -worker Harry -f config.toml

When the clustering feature is combined with workers then the [resume-name](/config#resume-name) becomes the cluster name concatenated with the worker name.

## Index Mapping

When indexing documents from MongoDB into elasticsearch the default mapping is as follows:

```text
mongodb database name . mongodb collection name -> elasticsearch index name
mongodb collection name -> elasticsearch type
mongodb document _id -> elasticsearch document _id
```

If these default won't work for some reason you can override the index and type mapping on a per collection basis by adding
the following to your TOML config file:

```toml
[[mapping]]
namespace = "test.test"
index = "index1"
type = "type1"

[[mapping]]
namespace = "test.test2"
index = "index2"
type = "type2"
```

With the configuration above documents in the `test.test` namespace in MongoDB are indexed into the `index1` 
index in elasticsearch with the `type1` type.

If you need your index and type mapping to be more dynamic, such as based on values inside the MongoDB document, then
see the sections [Middleware](#middleware) and  [Routing](#routing).

Make sure that automatic index creation is not disabled in elasticsearch.yml.

If automatic index creation must be controlled, whitelist any indexes in elasticsearch.yml that monstache will create.

Note that when monstache maps index and type names for ElasticSearch it does normalization based on the 
[Validity Rules](https://github.com/elastic/elasticsearch/issues/6736).  This includes making sure index names are
all lowercase and that index, types, and ids do not being with an underscore.

## Namespaces

When a document is inserted, updated, or deleted in mongodb a document is appended to the oplog representing the event.  This document has a field `ns` which is the namespace.  For inserts, updates, and deletes the namespace is the database name and collection name of the document changed joined by a dot. E.g. for `use test; db.foo.insert({hello: "world"});` the namespace for the event in the oplog would be `test.foo`.

In addition to inserts, updates, and deletes monstache also supports database and collection drops.  When a database or collection is dropped in mongodb an event is appended to the oplog.  Like the other types of changes this event has a field `ns` representing the namespace.  However, for drops the namespace is the database name and the string `$cmd` joined by a dot.  E.g. for `use test; db.foo.drop()` the namespace for the event in the oplog would be `test.$cmd`.  

When configuring namespaces in monstache you will need to account for both cases.  

!!! warning

	Specifically, be careful if you have configured `dropped-databases|dropped-collections=true` AND you also have a `namespace-regex` set.  If your namespace regex does not take into account the `db.$cmd` namespace the event may be filtered and the elasticsearch index not deleted on a drop.


## Middleware

monstache supports embedding user defined middleware between MongoDB and Elasticsearch.  middleware is able to transform documents,
drop documents, or define indexing metadata.  middleware may be written in either Javascript or in Golang as a plugin.  Golang plugins
require Go version 1.8 or greater on Linux. currently, you are able to use Javascript or Golang but not both (this may change in the future).

### Golang

monstache supports Golang 1.8+ plugins on Linux.  To implement a plugin for monstache you simply need to implement a specific function signature,
use the go command to build a .so file for your plugin, and finally pass the path to your plugin .so file when running monstache.

plugins must import the package `github.com/rwynn/monstache/monstachemap`

plugins must implement a function named `Map` with the following signature

```go
func Map(input *monstachemap.MapperPluginInput) (output *monstachemap.MapperPluginOutput, err error)
```

plugins can be compiled using

	go build -buildmode=plugin -o myplugin.so myplugin.go

to enable the plugin, start with `monstache -mapper-plugin-path /path/to/myplugin.so`

the following example plugin simply converts top level string values to uppercase

```go
package main
import (
	"github.com/rwynn/monstache/monstachemap"
	"strings"
)
// a plugin to convert document values to uppercase
func Map(input *monstachemap.MapperPluginInput) (output *monstachemap.MapperPluginOutput, err error) {
	doc := input.Document
	for k, v := range doc {
		switch v.(type) {
		case string:
			doc[k] = strings.ToUpper(v.(string))
		}
	}
	output = &monstachemap.MapperPluginOutput{Document: doc}
	return
}
```

the input parameter will contain information about the document's origin database and collection.
to drop the document (direct monstache not to index it) set `output.Drop = true`.
to simply pass the original document through to Elasticsearch, set `output.Passthrough = true`

`output.Index`, `output.Type`, `output.Parent` and `output.Routing` allow you to set the indexing metadata for each individual document.

if would like to embed other MongoDB documents (possibly from a different collection) within the current document 
before indexing, you can access the `*mgo.Session` pointer as `input.Session`.  With the mgo session you can use the 
[mgo API](https://godoc.org/gopkg.in/mgo.v2) to find documents in MongoDB and embed them in the Document set 
on output.  

### Javascript

#### Transformation

monstache uses the amazing [otto](https://github.com/robertkrimen/otto) library to provide transformation at the document field
level in Javascript.  You can associate one javascript mapping function per mongodb collection.  These javascript functions are
added to your TOML config file, for example:
	
```toml
[[script]]
namespace = "mydb.mycollection"
script = """
var counter = 1;
module.exports = function(doc) {
	doc.foo += "test" + counter;
	counter++;
	return _.omit(doc, "password", "secret");
}
"""

[[script]]
namespace = "anotherdb.anothercollection"
script = """
var counter = 1;
module.exports = function(doc) {
	doc.foo += "test2" + counter;
	counter++;
	return doc;
}
"""
```

The example TOML above configures 2 scripts. The first is applied to `mycollection` in `mydb` while the second is applied
to `anothercollection` in `anotherdb`.

You will notice that the multi-line string feature of TOML is used to assign a javascript snippet to the variable named
`script`.  The javascript assigned to script must assign a function to the exports property of the `module` object.  This 
function will be passed the document from mongodb just before it is indexed in elasticsearch.  Inside the function you can
manipulate the document to drop fields, add fields, or augment the existing fields.

The `this` reference in the mapping function is assigned to the document from mongodb.  

When the return value from the mapping function is an `object` then that mapped object is what actually gets indexed in elasticsearch.
For these purposes an object is a javascript non-primitive, excluding `Function`, `Array`, `String`, `Number`, `Boolean`, `Date`, `Error` and `RegExp`.

#### Filtering

If the return value from the mapping function is not an `object` per the definition above then the result is converted into a `boolean`
and if the boolean value is `false` then that indicates to monstache that you would not like to index the document. If the boolean value is `true` then
the original document from mongodb gets indexed in elasticsearch.

This allows you to return false or null if you have implemented soft deletes in mongodb.

```toml
[[script]]
namespace = "db.collection"
script = """
module.exports = function(doc) {
	if (!!doc.deletedAt) {
		return false;
	}
	return true;
}
"""
```

In the above example monstache will index any document except the ones with a `deletedAt` property.  If the document is first
inserted without a `deletedAt` property, but later updated to include the `deletedAt` property then monstache will remove the
previously indexed document from the elasticsearch index. 

Note you could also return `doc` above instead of returning `true` and get the same result, however, it's a slight performance gain
to simply return `true` when not changing the document because you are not copying data in that case.

You may have noticed that in the first example above the exported mapping function closes over a var named `counter`.  You can
use closures to maintain state between invocations of your mapping function.

Finally, since Otto makes it so easy, the venerable [Underscore](http://underscorejs.org/) library is included for you at 
no extra charge.  Feel free to abuse the power of the `_`.

#### Embedding Documents

In your javascript function you have access to the following global functions to retreive documents from MongoDB for
embedding in the current document before indexing.  Using this approach you can pull in related data.

```javascript
function findId(documentId, [options]) {
    // convenience method for findOne({_id: documentId})
    // returns 1 document or null
}

function findOne(query, [options]) {
    // returns 1 document or null
}

function find(query, [options]) {
    // returns an array of documents or null
}
```

Each function takes a `query` object parameter and an optional `options` object parameter.

The options object takes the following keys and values:

```javascript
var options = {
    database: "test",
    collection: "test",
    // to omit _id set the _id key to 0 in select
    select: {
        age: 1
    },
    // only applicable to find...
    sort: ["name"],
    limit: 2
}
```

If the database or collection keys are omitted from the options object, the values for database and/or
collection are set to the database and collection of the document being processed.

Here are some examples:

This example sorts the documents in the same collection as the document being processed by name and returns
the first 2 documents projecting only the age field.  The result is set on the current document before being
indexed.

```
[[script]]
namespace = "test.test"
script = """
module.exports = function(doc) {
    doc.twoAgesSortedByName = find({}, {
            sort: ["name"],
            limit: 2,
            select: {
              age: 1
            }
    });
    return doc;
}
"""
```

This example grabs a reference id from a document and replaces it with the corresponding document with that id.

```
[[script]]
namespace = "test.posts"
script = """
module.exports = function(post) {
    if (post.author) { // author is a an object id reference
        post.author = findId(post.author, {
          database: "test",
          collection: "users"
        });
    }
    return post;
}
"""
```

#### Indexing Metadata

You can override the indexing metadata for an individual document by setting a special field named
`_meta_monstache` on the document you return from your Javascript function.

Assume there is a collection in MongoDB named `company` in the `test` database.
The documents in this collection look like either 

```
{ "_id": "london", "type": "branch", "name": "London Westminster", "city": "London", "country": "UK" }
```
or
```
{ "_id": "alice", "type": "employee", "name": "Alice Smith", "branch": "london" }
```

Given the above the following snippet sets up a parent-child relationship in Elasticsearch based on the
incoming documents from MongoDB.

```
[[script]]
namespace = "test.company"
routing = true
script = """
module.exports = function(doc) {
    var meta = { type: doc.type, index: 'company' };
    if (doc.type === "employee") {
        meta.parent = doc.branch;
    }
    doc._meta_monstache = meta;
    return _.omit(doc, "branch", "type");
}
"""
```

The snippet above will route these documents to the `company` index in Elasticsearch instead of the
default of `test.company`.  Also, instead of using `company` as the Elasticsearch type, the type
attribute from the document will be used as the Elasticsearch type.  Finally, if the type is
employee then the document will be indexed as a child of the branch the person belongs to.  

We can throw away the type and branch information by deleting it from the document before returning
since the type information will be stored in Elasticsearch under `_type` and the branch information
will be stored under `_parent`.

The example is based on the Elasticsearch docs for [parent-child](https://www.elastic.co/guide/en/elasticsearch/guide/current/parent-child.html)

## Routing

Domain knowledge of your data can lead to better performance with a custom routing solution. Routing
is the process by which ElasticSearch determines which shard a document will reside in.  Monstache
supports user defined, or custom, routing of your MongoDB documents into ElasticSearch.  

Consider an example where you have a `comments` collection in MongoDB which stores a comment and 
its associated post identifier.  

```javascript
use blog;
db.comments.insert({title: "Did you read this?", post_id: "123"});
db.comments.insert({title: "Yeah, it's good", post_id: "123"});
```

In this case monstache will index those 2 documents in an index named `blog.comments` under the id
created by MongoDB.  When ElasticSearch routes a document to a shard, by default, it does so by hashing 
the id of the document.  This means that as the number of comments on post `123` grows, each of the comments
will be distributed somewhat evenly between the available shards in the cluster.  

Thus, when a query is performed searching among the comments for post `123` ElasticSearch will need to query
all of those shards just in case a comment happened to have been routed there.

We can take advantage of the support in ElasticSearch and in monstache to do some intelligent
routing such that all comments for post `123` reside in the same shard.

First we need to tell monstache that we would like to do custom routing for this collection by setting `routing`
equal to true on a custom script for the namespace.  Then we need to add some metadata to the document telling
monstache how to route the document when indexing.  In this case we want to route by the `post_id` field.

```toml
[[script]]
namespace = "blog.comments"
routing = true
script = """
module.exports = function(doc) {
	doc._meta_monstache = { routing: doc.post_id };
	return doc;
}
"""
```

Now when monstache indexes document for the collection `blog.comments` it will set the special `_routing` attribute
for the document on the index request such that ElasticSearch routes comments based on their corresponding post. 

The `_meta_monstache` field is used only to inform monstache about routing and is not included in the source
document when indexing to ElasticSearch.  

Now when we are searching for comments and we know the post id that the comment belongs to we can include that post
id in the request and make a search that normally queries all shards query only 1 shard.

```
$ curl -H "Content-Type:application/json" -XGET 'http://localhost:9200/blog.comments/_search?routing=123' -d '
{
   "query":{
      "match_all":{}
   }
}'
```

You will notice in the response that only 1 shard was queried instead of all your shards.  Custom routing is great
way to reduce broadcast searches and thus get better performance.

The catch with custom routing is that you need to include the routing parameter on all insert, update, and delete
operations.  Insert and update is not a problem for monstache because the routing information will come from your
MongoDB document.  Deletes, however, pose a problem for monstache because when a delete occurs in MongoDB the
information in the oplog is limited to the id of the document that was deleted.  But monstache needs to know where the
document was originally routed in order to tell ElasticSearch where to look for it.

Monstache gets around this problem by using a lookup table that it stores in MongoDB at `monstache.meta`.  In this collection monstache stores
the routing information for each document with custom routing.  When a delete occurs monstache looks up the route
in this collection and forwards that information to ElasticSearch on the delete request.

For more information see [Customizing Document Routing](https://www.elastic.co/blog/customizing-your-document-routing)

In addition to letting your customize the shard routing for a specific document, you can also customize the elasticsearch
`index` and `type` using a script by putting the custom information in the meta attribute. 

```toml
[[script]]
namespace = "blog.comments"
routing = true
script = """
module.exports = function(doc) {
	if (doc.score >= 100) {
		// NOTE: prefix dynamic index with namespace for proper cleanup on drops
		doc._meta_monstache = { index: "blog.comments.highscore", type: "highScoreComment", routing: doc.post_id };
	} else {
		doc._meta_monstache = { routing: doc.post_id };
	}
	return doc;
}
"""
```

## Merge Patches

A unique feature of monstache is support for JSON Merge Patches [rfc-7396](https://tools.ietf.org/html/rfc7396).

If merge patches are enabled monstache will add an additional field to documents indexed into elasticsearch. The
name of this field is configurable but it defaults to `json-merge-patches`.  

Consider the following example with merge patches enabled... 

```javascript
db.test.insert({name: "Joe", age: 16, friends: [1, 2, 3]})
```
At this point you would have the following document source in elasticsearch.

	 "_source" : {
	  "age" : 16,
	  "friends" : [
	    1,
	    2,
	    3
	  ],
	  "json-merge-patches" : [
	    {
	      "p" : "{\"age\":16,\"friends\":[1,2,3],\"name\":\"Joe\"}",
	      "ts" : 1487263414,
	      "v" : 1
	    }
	  ],
	  "name" : "Joe"
	}

As you can see we have a single timestamped merge patch in the json-merge-patches array.  

Now let's update the document to remove a friend and update the age.

```javascript
db.test.update({name: "Joe"}, {$set: {age: 21, friends: [1, 3]}})
```

If we now look at the document in elasticsearch we see the following:

	"_source" : {
	  "age" : 21,
	  "friends" : [
	    1,
	    3
	  ],
	  "json-merge-patches" : [
	    {
	      "p" : "{\"age\":16,\"friends\":[1,2,3],\"name\":\"Joe\"}",
	      "ts" : 1487263414,
	      "v" : 1
	    },
	    {
	      "p" : "{\"age\":21,\"friends\":[1,3]}",
	      "ts" : 1487263746,
	      "v" : 2
	    }
	  ],
	  "name" : "Joe"
	}

You can see that the document was updated as expected and an additional merge patch was added.

Each time the document is updated in mongodb the corresponding document in elasticsearch gains a
timestamped merge patch.  Using this information we can time travel is the document's history.

There is a merge patch for each version of the document.  To recreate a specific version we simply need
to apply the merge patches in order up to the version that we want.

To get version 1 of the document above we start with {} and apply the 1st merge patch.  

To get version 2 of the document above we start with {}

- apply the 1st merge patch to get v1
- apply the 2nd merge patch to v1 to get v2

The timestamps associated with these merge patches are in seconds since the epoch, taken from the
timestamp recorded in the oplog when the insert or update occured. 

To enable the merge patches feature in monstache you need to add the following to you TOML config:

	enable-patches = true
	patch-namespaces = ["test.test"]

You need you add each namespace that you would like to see patches for in the patch-namespaces array.

Optionally, you can change the key under which the patches are stored in the source document as follows:

    merge-patch-attribute = "custom-merge-attr"	

Merge patches will only be recorded for data read from the MongoDB oplog.  Data read using the direct read
feature will not be enhanced with merge patches.

Most likely, you will want to turn off indexing for the merge patch attribute.  You can do this by creating
an index template for each patch namespace before running monstache...

	PUT /_template/test.test
	{
	    "template" : "test.test",
	    "mappings" : {
		"test" : {
		    "json-merge-patches" : { "index" : false }
		}
	    }
	}

## Systemd

Monstache has support built in for integrating with systemd. The following `monstache.service` is an example 
systemd configuration.

    [Unit]
    Description=monstache sync service

    [Service]
    Type=notify
    ExecStart=/usr/local/bin/monstache
    WatchdogSec=30s
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

System-d unit files are normally saved to `/lib/systemd/system`.  

After saving the monstache.service file you can run `systemctl daemon-reload` to tell systemd to reload 
all unit files. 

You can enable the service to start on boot with `systemctl enable monstache.service` and start the service with `systemctl start monstache.service`.

With the configuration above monstache will notify systemd when it has started successfully and then notify 
systemd repeatedly at half the WatchDog interval to signal liveness.  The configuration above causes systemd
to restart monstache if it does not start or respond within the WatchdDog interval.

## HTTP Server

Monstache has a built in HTTP server that you can enable with --enable-http-server. It
listens on :8080 by default but you can change this with --http-server-addr.

When using monstache with kubernetes this server can be used to detect liveness and 
[act accordingly](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

The following GET endpoints are available

#### /started

Returns the uptime of the server

#### /healthz

Returns at 200 status code with the text "ok" when monstache is running

#### /stats

Returns the current indexing statistics in JSON format. Only available if stats are enabled

#### /config

Returns the configuration monstache is using in JSON format

---


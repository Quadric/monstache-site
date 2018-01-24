# Monstache

Sync MongoDB to ElasticSearch in realtime

---

Monstache is a sync daemon written in Go that continously indexes your MongoDB collections into
Elasticsearch. Monstache gives you the ability to use Elasticsearch to do complex searches and aggregations 
of your MongoDB data and easily build realtime Kibana visualizations and dashboards.

## Features

- Optionally filter the set of collections to sync

- Support for sharded MongoDB clusters

- Direct read mode to do a full sync of collections in addition to tailing the oplog

- Transform and filter documents before indexing using Golang plugins or JavaScript

- Index the content of GridFS files

- Support for hard and soft deletes in MongoDB

- Support for propogating database and collection drops

- Optional custom document routing in ElasticSearch

- Stateful resume feature

- Worker and Clustering modes for High Availability

- Support for [rfc7396](https://tools.ietf.org/html/rfc7396) JSON merge patches

- Single binary with a light footprint 

- Systemd support

- Optional http server to get access to liveness, stats, etc

See [Getting Started](/start/) for instructions how to get
it up and running.

---
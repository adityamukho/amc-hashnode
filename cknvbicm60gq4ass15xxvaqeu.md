## Introducing CivicGraph

I would like to introduce an open source, Apache 2.0 licensed project of mine:
 
https://github.com/CivicGraph/CivicGraph


CivicGraph is a versioned-graph data store - it retains all changes that its data (vertices and edges) have gone through to reach their current state. It supports point-in-time graph traversals, letting the user query any past state of the graph just as easily as the present.

It is a  [Foxx Microservice](https://www.arangodb.com/why-arangodb/foxx/)  for  [ArangoDB ](https://www.arangodb.com/) that features VCS-like semantics in many parts of its interface, and is backed by a transactional event tracker. It is currently being developed and tested on ArangoDB v3.5, with support for v3.6 in the pipeline.

CivicGraph is a potential fit for scenarios where data is best represented as a network of vertices and edges (i.e., a graph) having the following characteristics:

1. Both vertices and edges can hold properties in the form of attribute/value pairs (equivalent to JSON objects).

1. Documents (vertices/edges) mutate within their lifespan (both in their individual attributes/values and in their relations with each other).

1. Past states of documents are as important as their present, necessitating retention and queryability of their change history.

Its API is split into 3 top-level categories:

# Document

**Create** - Create single/multiple documents (vertices/edges).

**Replace** - Replace entire single/multiple documents with new content.

**Delete** - Delete single/multiple documents.

**Update** - Add/Update specific fields in single/multiple documents.

**(Planned) Explicit Commits** - Commit a document's changes separately, after it has been written to DB via other means (AQL / Core REST API / Client).

**(Planned) CQRS/ES Operation Mode** - Async implicit commits.

# Event

**Log** - Fetch a log of events (commits) for a given path pattern (path determines scope of documents to pick). The log can be optionally grouped/sorted/sliced within a specified time interval.

**Diff** - Fetch a list of forward or reverse commands (diffs) between commits for specified documents.

**(Planned) Branch/Tag** - Create parallel versions of history, branching off from a specific event point of the main timeline. Also, tag specific points in branch+time for convenient future reference.

**(Planned) Materialization** - Point-in-time checkouts.

# History

**Show** - Fetch a set of documents, optionally grouped/sorted/sliced, that match a given path pattern, at a given point in time.

**Filter** - In addition to a path pattern like in 'Show', apply an expression-based, simple/compound post-filter on the retrieved documents.

**Traverse** - A point-in-time traversal (walk) of a past version of the graph, with the option to apply additional post-filters to the result.

I hope some of you may find this a useful service to address several types of data modelling challenges pertaining to retention and querying of historical graph data.
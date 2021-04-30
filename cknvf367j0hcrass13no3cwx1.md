## Exploring Graph Database Versioning Approaches

Building on my [previous post](/the-case-for-versioned-graph-databases/), where I emphasize the need for versioned graph databases, I explore a few of the possible approaches to designing one. This is not a ground-up approach that delves into the design of the database engine itself, but rather an _overlay_ approach that can let us run version control atop any standard graph database.

## A Naive Approach
Think of a historical key-value store - one where the history of values against every key is retained, and retrieved using a composite of the key and a revision number (absolute or relative). Optionally, to get the **current** value of a key, we may omit the revision number. This is one of the simplest conceptual forms of a historical database. Since many real world graph databases are built on top of an underlying key-value store ([ArangoDB](https://www.arangodb.com/docs/stable/architecture-storage-engines.html#rocksdb), [JanusGraph](https://docs.janusgraph.org/latest/storage-backends.html), [DataStax Enterprise Graph](https://www.datastax.com/wp-content/uploads/resources/datasheets/DataStax-Enterprise-Graph-DS.pdf)), we will try to use our historical key-value store to see if it gives our graph database some degree of history retention.

We can naively concoct a graph representation on this database by storing documents as key-value pairs - keys being the unique document ids and values being their respective attribute-value pairs or property lists. For the sake of this thought experiment, we will not bother with how the connections are internally represented, i.e. whether source and destination node ids are stored as edge attributes, or incoming and outgoing edge lists are stored as node attributes, or some other fancy scheme (In real-life graph databases, this is dictated by the design the underlying implementation).

### The Historical KV Store - A Closer Look

<figure id="historical-keyvalue-store">
![historical_keyvalue_store.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619248652851/7jOdzUHUW.png)
<figcaption>Figure 1: Evolution of stored data over time in the historical KV store</figcaption>
</figure>

The figure above shows what the contents of this key-value store might look like for a few sample key-value pairs. Every write operation (create/update/delete) for a given KV pair is associated with a **revision number** and a **timestamp**. The latest value for a key is always immediately available using a simple key lookup. This can happen in `O(1)` for an in-memory store. This is depicted in the figure by projecting the latest values of existing pairs onto the line `Tnow`. However, when we want to find out the state of the database at some point of time in the past, things get a little more complicated. The table below shows the values of all the keys depicted in the figure, at different times.

<table class="table table-hover" id="projection-historical-values">
    <caption>Table 1: Projection of historical values at different times</caption>
    <thead>
        <tr>
            <th></th>
            <th>T<sub>1</sub></th>
            <th>T<sub>2</sub></th>
            <th>T<sub>3</sub></th>
            <th>T<sub>4</sub></th>
            <th>T<sub>now</sub></th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><b>K<sub>1</sub></b></td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>a</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>a</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>a</sub>)</td>
            <td>V<sub>1</sub> @ (R<sub>1</sub>, T<sub>f</sub>)</td>
            <td>V<sub>1</sub></td>
        </tr>
        <tr>
            <td><b>K<sub>2</sub></b></td>
            <td>--</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>c</sub>)</td>
            <td>V<sub>1</sub> @ (R<sub>1</sub>, T<sub>e</sub>)</td>
            <td>V<sub>1</sub> @ (R<sub>1</sub>, T<sub>e</sub>)</td>
            <td>D @ (R<sub>2</sub>, T<sub>h</sub>)</td>
        </tr>
        <tr>
            <td><b>K<sub>3</sub></b></td>
            <td>--</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>d</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>d</sub>)</td>
            <td>D @ (R<sub>1</sub>, T<sub>g</sub>)</td>
            <td>D @ (R<sub>1</sub>, T<sub>g</sub>)</td>
        </tr>
        <tr>
            <td><b>K<sub>4</sub></b></td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>b</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>b</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>b</sub>)</td>
            <td>V<sub>0</sub> @ (R<sub>0</sub>, T<sub>b</sub>)</td>
            <td>V<sub>0</sub></td>
        </tr>
    </tbody>
</table>

### Temporal Queries
This table was easy to build from the visual depiction in [Figure 1](#historical-keyvalue-store). But how does our historical KV store figure out the state of the DB at, say time `T3`? Here's what we have to work with:
1. Revision numbers can be relatively referenced (similar to Git), so we can assume these to be integer values starting with 0. Positive numbers reflect revisions w.r.t. the beginning of revision history (`R0`) and negative numbers w.r.t to the end (`Rlatest`).
2. Since the database allows both **key-only** lookups (for latest revisions) and **key+revision** lookups, we can assume that it internally maintains a 2-field hash index `(key, revision)`, keeping the 2nd level sorted in reverse order of the revision number (to facilitate retrieving the latest version for key-only lookups).

Therefore, to find the value of, say, `K2` at time `T3`, we need to perform the following steps:
1. Determine if `T3` is closer to `T0` or `Tnow`. Based on this, we will start from the oldest or the newest revision of `K2` respectively.
2. If `T3` is closer to `T0`.
    i. Set `n` to `0`.
    ii. Lookup `K2 @ Rn` and note its timestamp `T`.
    iii. If `T > T3` then return `K2 @ R(n-1)` (If `n = 0` then return `null`).
    iv. Else increase `n` by `1` and go to step (ii).
3. Else if `T3` is closer to `Tnow`.
    i. Get `K2 @ Tnow`. Note the revision number and assign to `N`. Note its timestamp `T`.
    ii. Set `n` to `N`.
    iii. If `T < T3` then return `K2 @ Rn`.
    iv. Else decrease `n` by `1`.
    v. Get `K2 @ Rn`, note its timestamp `T` and go to step (iii).

This is evidently far more complex than a simple key or _key+revision_ lookup, and not in `O(1)` anymore. It is now `O(N(K))` where `N(K)` is the number of revisions for key `K`. This greatly inefficient lookup can be speeded up drastically by additionally maintaining a 2-field skiplist (see below) index `(key, timestamp)`, keeping the 2nd level sorted in reverse chronological order. This is still at best an `O(log(N(K)))` operation.

> A [skiplist](https://en.wikipedia.org/wiki/Skip_list) index supports both equality and range queries. So we can ask it to return all values for key = `K` and timestamp < `T`.

## Conclusion
We see that while this design provides a history of every individual document out of the box, and the history can be queried by revision number or point-in-time,
1. It does not readily expose the the structure of the graph as a whole (or subgraphs, or `k-hop` neighborhoods - all of which might be of interest to a network analyst.),
2. Fetching a past state of the graph is more expensive than fetching its current state (this should be an acceptable trade-off for most [OLTP](https://en.wikipedia.org/wiki/Online_transaction_processing) scenarios),
3. The revision-number based historical KV store does not offer any intrinsic benefits for time-based lookups, and
4. There is potential for a lot of storage and write-bandwidth overhead when storing multiple versions of large documents where each version only updates a small portion of a document, since it results in redundant storage.

We see that this approach has several drawbacks which make it unsuitable for an efficient historical graph data store. We will explore other, hopefully better approaches in future posts.
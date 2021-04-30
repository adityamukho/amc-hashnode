## The Case for Versioned Graph Databases

## Why Graph Databases?
Graph databases have become ubiquitous over the years due to their incredible representational and querying capabilities, when it comes to highly interconnected data. Wherever data has an inherent networked structure, graph databases fare better at storing and querying that data than other NoSQL databases as well as relational databases, because they naturally persist the underlying connected structure. This allows for traversal semantics in declarative graph query languages, and also, better performance than SQL - especially for deep traversals. Additionally, they often help unravel emergent network topologies in legacy data, that had not previously been mined for such structures. At the very least, they make the process a lot less tedious.

Most real world network data intrinsically lend themselves to graphical representations, and hence, can be modeled in graph databases. These include, to name a few:
1. Wired and wireless computer networks and cellular networks
2. Road, rail, air and shipping routes,
3. Supply and distribution chains,
4. Biological and artificial neural networks,
5. Complex chemical and nuclear reaction chains,
6. Social networks,
7. Software package and library dependency trees, and many more.

## Why Versioned Graphs?
In addition to reaping the benefits of living in graph databases, many real world applications also stand to take advantage of network evolution models, i.e. a record of changes to a network over time; for example, analyzing railway track utilization efficiency as a function of signal array timing, or the simulation of nucleotide concentration changes over time in a nuclear fission reactor. However, in most of the prominent mainstream graph databases that are freely available at the time of this writing, I have not come across any that offer some sort of built-in revision tracking (meaning older versions of data are retained for future retrieval).

Particularly for graph databases, the concept of revisions applies not only to individual nodes and edges, but also to the structure of the graph as a whole, i.e. it should be relatively easy to store and retrieve not only individual document (node/edge) histories, but also the structural history of the graph or a portion of it. This is a key difference between a hypothetical _versioned or historical graph database_ and a _general purpose event store_ (see below), which is usually tuned for the former but not the latter.

> An event store is a database that records entity write operations (creates/updates/deletes) as a series of deltas wrapped in events. Each delta is the difference between the contents of the updated entity and its previous version. It is part of an _event payload_, where the event represents the particular write operation (create/update/delete) that occurred. Thus, deltas encode the entire write history of the entity.

I therefore submit that there is a need for a practical, historical graph database that has the following minimal set of characteristics:
1. A mechanism for efficiently recording individual document (node/edge) writes (creates/updates/deletes) in such a way that they can be rewound and replayed.
2. An internal storage architecture that not only maintains the current structure of the graph, but also allows for a quick rebuild and retrieval of its structure at any point of time in the past. This could, optionally, be optimized to retrieve recent structures faster than older ones.
3. An efficient query engine that can traverse current/past graph structures to retrieve subgraphs or `k-hop` neighborhoods of specified nodes. In case of historical traversals, this should be optimized to rebuild only the relevant portions of the graph, where feasible.

## The Current State of Historical Graph Databases
There is a general consensus in the computing and scientific research community for the need of a historical graph database, and to the best of my knowledge, research has been carried out along two primary forks:

1. Graph databases with built-in revision support at the DB engine level, i.e. they are designed from the ground up to support revisions:

    i. Vijitbenjaronk et al. [Scalable time-versioning support for property graph databases](https://ieeexplore.ieee.org/document/8258092).

2. Database designs, supplemented by external application/service layers to provide a _revision tracking facade_ on top of conventional static (based on the database taxonomy proposed by Snodgrass et al. in their 1985 ACM SIGMOID paper titled [A Taxonomy of Time in Databases](https://www.researchgate.net/publication/221212735_A_Taxonomy_of_Time_in_Databases).) graph database engines:

    i. Khurana et al. [Efficient Snapshot Retrieval over Historical Graph Data](https://www.researchgate.net/publication/229556866_Efficient_Snapshot_Retrieval_over_Historical_Graph_Data),

    ii. Khurana et al. [Storing and Analyzing Historical Graph Data at Scale](https://www.researchgate.net/publication/282403421_Storing_and_Analyzing_Historical_Graph_Data_at_Scale).

There would be many more publications and implementations available upon a quick search, but I believe they would all fall under one of the above two categories.
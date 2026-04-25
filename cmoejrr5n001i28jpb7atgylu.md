---
title: "Minigraf: A Spiritual Successor to RecallGraph"
seoTitle: "Minigraf: An Embedded Bi-Temporal Graph Database in Rust"
seoDescription: "Minigraf is an embedded bi-temporal graph database in Rust — Datalog queries, single-file storage, and the valid-time dimension RecallGraph never shipped."
datePublished: 2026-04-25T16:22:00.254Z
cuid: cmoejrr5n001i28jpb7atgylu
slug: minigraf-a-spiritual-successor-to-recallgraph
tags: version-control, rust, graph-database, temporal-databases, datalog

---

It has been roughly five years since the [last release](https://github.com/RecallGraph/RecallGraph/releases/tag/v1.0.0) of RecallGraph. The repository has been quiet for most of that time, and is now formally archived. The reasons are not particularly dramatic: the ArangoDB Foxx ecosystem moved on, the operational shape of "install a microservice into your database instance" turned out to be a poor fit for the kinds of applications that most needed point-in-time graph queries, and my own time and attention drifted to other problems. None of this is a story worth a long post.

What does seem worth a post is that the underlying problem — the [case for versioned graph databases](https://adityamukho.com/the-case-for-versioned-graph-databases) — has not gone away. If anything, it has become more pointed. AI agents now routinely accumfulate beliefs, retract them, and need to justify decisions made under earlier knowledge states. Mobile applications need to record corrections to data that was first entered offline, without losing the original record. Auditability is no longer an enterprise nice-to-have; it is a baseline expectation in a growing list of regulated and semi-regulated domains.

So I have started over. The new project is called [**Minigraf**](https://github.com/project-minigraf/minigraf), and this post is partly an announcement and partly an attempt to be honest about what I learned the hard way the first time around.

### What Minigraf Is

Minigraf is an embedded, single-file, bi-temporal graph database written in Rust, with Datalog as its query language. The elevator pitch I have been using is "the SQLite of bi-temporal graph databases," which is glib but reasonably accurate as a positioning statement: one file, no server, no daemon, one library dependency, runs anywhere Rust runs — including WebAssembly, Android via Kotlin bindings, and iOS via Swift bindings.

The feature set, abbreviated:

1.  **Bi-temporal queries** — both transaction time (when a fact was recorded) and valid time (when the fact was true in the world). Valid time was on RecallGraph's roadmap and never shipped. In Minigraf it is first-class.
    
2.  **Datalog with recursive rules** — multi-hop traversals are natural rather than bolted on, and time-travel is just another dimension in the relations rather than a separate API surface.
    
3.  **A single** `.graph` **file** — no ArangoDB instance, no Foxx deployment, no service to install into a database that you also have to install.
    
4.  **Rust core, ~99% Rust codebase** — which means the WASM and mobile targets are a recompile rather than a port.
    

The timing is also different from RecallGraph's. Agent-shaped systems have created a new and arguably larger audience for these semantics than the audit-and-compliance world I had in mind in 2019, which I'll come back to below.

The query model is the EAV (entity-attribute-value) triple store that Datomic popularised, with the four covering indexes (EAVT/AEVT/AVET/VAET) that make graph traversals tractable in either direction. None of this is novel by itself; the contribution is the packaging.

### What Changed in My Thinking

A few things, looking back across both projects.

The first is that **the substrate matters more than the algorithm**. RecallGraph was a versioning facade on top of ArangoDB, in the second of the two categories I [outlined in 2019](https://adityamukho.com/the-case-for-versioned-graph-databases) — "database designs supplemented by external application/service layers to provide a revision-tracking facade on top of conventional static graph database engines." That category has a real ceiling. The facade approach inherits the operational footprint of the underlying database, inherits its release cadence and breakage, and inherits its assumptions about deployment topology. Building the version-tracking logic *into* the storage engine, rather than on top of it, removes a class of constraints that I had spent years working around without quite realising.

The second is that **the deployment story is part of the design**. RecallGraph required users to run an ArangoDB cluster, install a Foxx microservice into it, and configure both. Even when this worked, it placed the project firmly in the "weekend infrastructure" category — fine for an experiment, hard to justify for production. Minigraf's design is the opposite extreme: `cargo add minigraf`, open a file, query. The downstream consequences of that single change are larger than I expected. It is the difference between a database you evaluate and a library you import.

The third is that **bi-temporal really does mean both dimensions**. The original RecallGraph implementation handled transaction time well and treated valid time as a future feature. The two are independent in ways that are not obvious until you start trying to retrofit one onto the other, which is part of why Minigraf was a clean rewrite rather than a port. The [taxonomy from Snodgrass et al.](https://www.researchgate.net/publication/221212735_A_Taxonomy_of_Time_in_Databases) that I have cited in several earlier posts turns out to be load-bearing rather than pedagogical.

The fourth, less technical, is that **the audience for this kind of database has shifted**. In 2019, the people who cared about versioned graph data were primarily working on audit trails, regulatory compliance, and historical analysis of biological or infrastructure networks. Those use cases are still present, but the loudest current demand is from people building agent-shaped systems — programs that maintain beliefs, revise them, and need to be able to explain themselves later. Bi-temporal databases turn out to be a remarkably good fit for this, in ways that were not at the front of my mind when I first designed RecallGraph.

### What Happens to RecallGraph

The repository remains available and the documentation site is still online. I will not be developing it further. The README now points to Minigraf as the spiritual successor, which seems like the honest thing to do given the traffic it still receives. If anyone has been running it in production and would like help with a migration path, get in touch — though I should warn that the data models are similar enough to be familiar and different enough that "migration" is a project rather than a script.

### Where to Go Next

The Minigraf repository is at [github.com/project-minigraf/minigraf](https://github.com/project-minigraf/minigraf). The [Comparison wiki page](https://github.com/project-minigraf/minigraf/wiki/Comparison) covers how it relates to XTDB, Datomic, Cozo, and the other projects in this neighbourhood. The [Use Cases page](https://github.com/project-minigraf/minigraf/wiki/Use-Cases) covers the agent-memory, mobile, and browser scenarios in more detail.

There is also a working sibling project, [temporal-reasoning](https://github.com/adityamukho/temporal_reasoning), which uses Minigraf as the storage layer for a structured-memory skill aimed at coding agents — Claude Code, OpenCode, Codex, and similar. It is the most concrete validation I have for the agent-memory use case so far.

If you arrived here from an older post, or from a search for "versioned graph database" that has been turning up the same handful of results since 2019: the answer to that search is no longer just RecallGraph, and increasingly it is Minigraf.
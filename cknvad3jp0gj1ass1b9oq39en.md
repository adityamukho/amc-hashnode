---
title: "Introducing foxx-tracer"
seoTitle: "foxx-tracer - an OpenTracing Implementation for Foxx Microservices"
seoDescription: "foxx-tracer is an opentracing library and ecosystem built for the Foxx runtime."
datePublished: 2020-08-17T05:09:00.000Z
cuid: cknvad3jp0gj1ass1b9oq39en
slug: foxx-tracer-an-opentracing-implementation-for-foxx-microservices
canonical: https://adityamukho.com/foxx-tracer-an-opentracing-implementation-for-foxx-microservices/
tags: opensource

---

# About OpenTracing
Application performance monitoring and distributed tracing have become invaluable tools in the hands of modern application developers, providing critical insights into the inner workings of applications and helping isolate performance bottlenecks and stress points.

[OpenTracing](https://opentracing.io/) is a free and widely used standard for implementing distributed tracing. It allows developers and operations teams to trace execution pathways across applications built on different platforms and running on different servers, even across multiple data centers, tracking trace context across application and service boundaries as they communicate with each other as long as they all implement the common tracing semantics specified by the OpenTracing standard.

> ...per-process logging and metric monitoring have their place, but neither can reconstruct the elaborate journeys that transactions take as they propagate across a distributed system. Distributed traces are these journeys.
>
> Source: https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736

# About ArangoDB
ArangoDB is a free and open-source native multi-model database system developed by ArangoDB GmbH. The database system supports three data models (key/value, documents, graphs) with one database core and a unified query language AQL (ArangoDB Query Language). The query language is declarative and allows the combination of different data access patterns in a single query. ArangoDB is a NoSQL database system but AQL is similar in many ways to SQL.

ArangoDB has been referred to as a universal database but its creators refer to it as a "native multi-model" database to indicate that it was designed specifically to allow key/value, document, and graph data to be stored together and queried with a common language.

## The Foxx Runtime
Quoted from the ArangoDB website:
> Foxx is a JavaScript framework for writing data-centric HTTP microservices that run directly inside of ArangoDB.
> 
> Traditionally, server-side projects were developed as standalone applications that guide communications between the client-side front-end and the database back-end. Through the Foxx Microservice Framework, ArangoDB allows application developers to write their data access and domain logic as microservices and running directly within the database with native access to in-memory data.
>
> Source: https://www.arangodb.com/why-arangodb/foxx/

Foxx services consist of JavaScript code running in the V8 JavaScript runtime embedded inside ArangoDB. Each service is mounted in each available V8 context (the number of contexts can be adjusted in the server configuration). Incoming requests are distributed across these contexts automatically.

Great, so this is just like programming for the Node.js environment, which also runs on V8 and supports the CommonJS module loading mechanism.

**Except, there's a catch - Foxx is 100% synchronous!**

# Why _foxx-tracer_?
Most tracing libraries in the nodeverse are asynchronous, and so do not work in the synchronous V8 runtime that ArangoDB uses to run its Foxx services. [foxx-tracer](https://github.com/RecallGraph/foxx-tracer) bridges this gap by being a 100% synchronous, dedicated module built for the Foxx runtime.

It is a CommonJS-loadable package available through the NPM registry. However, it relies on a number of features only available in a Foxx environment. It also depends on a companion [collector service](https://github.com/RecallGraph/foxx-tracer-collector) which itself is a Foxx microservice. **These dependencies make this module incompatible with Node.js and browser-based runtimes.**

## The _foxx-tracer ecosystem_
_foxx-tracer_ works in conjunction with a bunch of other applications/modules which together comprise the _foxx-tracer ecosytem_. These are:
1. [foxx-tracer](https://github.com/RecallGraph/foxx-tracer) - a module that you include in your foxx microservice when you want to enable distributed tracing for it.
2. [foxx-tracer-collector](https://github.com/RecallGraph/foxx-tracer-collector) - a collector agent that receives OpenTracing spans within the sychronous Foxx environment, and then asynchrounously pushes them to multiple, configurable destinations. The collector supports a simple plugin mechanism through which one can add multiple _reporters_ to talk to different destinations.
3. A bunch of [reporters](https://www.npmjs.com/search?q=foxx-tracer-reporter) that let the collector push its incoming spans to different APM endpoints (like Datadog, NewRelic, etc). It is very easy to write your own reporter if you don't find one for your specific endpoint in the NPM registry. Two reporters written by me are:
    1. A [console reporter](https://github.com/RecallGraph/foxx-tracer-reporter-console) that prints traces to the ArangoDB log.
    2. A [production-ready reporter](https://github.com/RecallGraph/foxx-tracer-reporter-datadog) available for the Datadog Cloud Monitoring Service.

All components are well documented and there is a reference implementation in [RecallGraph](https://github.com/RecallGraph/RecallGraph), which one can look up to see how all the pieces fit together.
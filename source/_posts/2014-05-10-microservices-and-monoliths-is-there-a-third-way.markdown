---
layout: post
title: "Microservices and monoliths - is there a third way?"
date: 2014-05-10 12:00:00 +0100
comments: true
categories: [Microservices,Architecture,NetKernel,Erlang]
---

**TL;DR:** Microservices vs. monolith is a false dichotomy.

Microservices are gaining a lot of interest at the moment as an antidote to the limitations of monolithic application architecture. But as with all things in software (and life in general) they come at a significant cost. Arguably, monoliths and microservices occupy extreme points in a design space, and I've recently been wondering about other strategies that could yield similar benefits but with different tradeoffs.

<!-- more -->

Essentials
----------
Microservices are independently deployable, which means (provided other design constraints are met) that teams can ship new features independently from one another, avoiding the coordination cost associated with merging code into a single artefact. Services should be single purpose (and therefore have only one reason at a time to change), so the release of feature A should never be delayed while feature B is still in progress.

Splitting system components into separate processes that can only communicate via a network forces a greater degree of decoupling than is typical between components within a monolith. Rich Hickey commented in a presentation (which I can't find now, annoyingly) that microservices encourage component interfaces based on immutable data with pass-by-value semantics, which is impossible to achieve in most languages (Clojure being an exception) other than by self-imposed discipline. An avoidance of shared state (e.g. two services accessing the same DB schema) reinforces this separation.

This decoupling enables much greater freedom in technology choices. Services with APIs based entirely on platform-agnostic standards such as HTTP and JSON can be replaced transparently with something based on an totally different tech stack. So when you've passed Peak NodeJS and want to move onto the next big thing, you can manage the transition incrementally rather doing a total re-write (which you should [Never Do](http://www.joelonsoftware.com/articles/fog0000000069.html), right?).

Finally, microservices are by default more operationally transparent than modules within a monolith. Of course it's possible to instrument and monitor individual components in a single app, but it's not common practice to do so and it requires explicit effort. By comparison, services that communicate via HTTP or messaging give away plenty of operational data (log files, messages on a topic etc.) for free.


Tradeoffs
---------
Benjamin Wootton's post ["Microservices - Not a free lunch!"](http://contino.co.uk/blog/2013/03/31/microservices-no-free-lunch.html) does a good job of explaining the additional costs of a microservice architecture, so I'll merely summarise the points I think are relevant.

Systems distributed via a network are subject to high latency and numerous failure modes absent when in-process. While some degree of distribution is inevitable in modern applications, microservices could be said to be *maximally distributed*, so these issues have to be considered for practically every bit of behaviour, even where there's no benefit in terms of scaling or resilience.

A corollary of the prevous point is that all (or most) service interactions should be asynchronous if you want to build a high performing and available system. However, async programming is hard, and even harder to test effectively. Asynchronous systems are more likely to develop [emergent behaviours](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.35.7368&rep=rep1&type=pdf) that are near impossible to predict or test for, and that will almost certainly burn you in production at some point. 

Tolerating high and unpredicatable latency in practice usually means exposing coarse-grained APIs, which isn't always ideal.

Finally, moving features between services is far more costly than between modules in a single program, particularly if those services were built on different technologies.


There are some other technologies that appear to exhibit at least some of the benefits described above, while potentially mitigating some of the problems. I'll caveat by saying that I've never built or run anything with these, so I can't personally vouch for whether they live up to their marketing!


Erlang
------
[Martin Fowler's microservices article](http://martinfowler.com/articles/microservices.html) mentions in passing similarities to [Erlang](http://www.erlang.org/). Erlang supports hot code swapping, inter-process communication via asynchronous message passing and distribution over a network. This ticks many of the same boxes as microservices - independently deployable components, component boundaries enforced by immutable, data-centric communication protocols, no shared state.

A big additional advantage is that (many) processes can be physically co-located, avoiding the pain of distribution when it isn't required. Even when it is, Erlang OTP provides a good base for building in graceful failure recovery.

Erlang's model is of a much lower level of abstraction than a typical microservice's, however. It imposes no inter-process messaging semantics beyond async, immutable message passing, whereas microservices are often resource-oriented (i.e. HTTP). As a result, an Erlang based system would have no more natural transparency than a typical monolith.

There are both benefits and drawbacks to targetting a single platform like Erlang. Clearly, this makes refactoring much easier, but on the other hand you are constrained to a single set of languages, programming models and runtime characteristics which might be rendered obsolete (or suboptimal at least) when something radically different comes along.


NetKernel
---------
[NetKernel](http://1060research.com/products) is a less well known but nonetheless very interesting option in this space. It bills itself as a "Uniform Resource Engine" and is the result of a piece of research on the web and the reasons for its outstanding success (in a somewhat similar vein to Roy Fielding's work on REST).

Systems are built on NetKernel by composing resources exposed by modules. Resources are addressed (and address each other) via what could be considered a superset of HTTP. A killer feature that this enables is highy efficient caching - since relationships between resources are transparent to NetKernel, it can walk the dependency graph when a resource's state changes and invalidate cache entries where necessary (implying that state can otherwise be cached indefinitely). An additional benefit is that NetKernel can surface that transparency to system administrators, and provides various visualisations for things like resource utilisation and call paths.

NetKernel modules are hot deployable and as with Erlang, many can be co-located on the same host, making independent releases a possibility while not forcing maximal distribution to achieve it. In-process communication can be done synchronously, alleviating the overhead of going fully async. However, NetKernel can be run distributed, communicating via a custom network protocol that preserves cache coherence between nodes.

Like with Erlang, NetKernel limits you the the languages and APIs of a single platform, in this case the JVM, with all the benefits and drawbacks that implies.

["Why Microservices and not Yoctoservices?"](http://durablescope.blogspot.co.uk/2014/05/why-microservices-and-not-yoctoservices.html) is a thoughtful post from one of NetKernel's creators on the problems of service sizing and the solutions offered by resource oriented computing.


Others?
-------
I considered including [OSGi](http://www.osgi.org/Technology/WhatIsOSGi) in this post due to its support for hot deployable modules into a single process. However, intra-module communication is via direct calls between objects, which fails the decoupling test in my opinion. Also nobody I've spoken to with practical experience of OSGi has a good word to say about it.

Containers (e.g. [Docker](https://www.docker.io/)) and container-based PaaS products (e.g. [Cloud Foundry](http://cloudfoundry.org/) and [OpenShift](https://www.openshift.com/)) are on the rise as a solution for hosting microservices. Perhaps (and I'm speculating wildly now) some of these will evolve to support service locality awareness, such that services could choose to expose synchronous, fine grained APIs to co-located clients with reduced error handling requirements.



Conclusion
----------
The debate around microservices is yielding valuable insights about how we build sustainable and scalable systems. I hope over time these will give rise to a more nuanced range of solutions than the current monolith vs. microservices dichotomy, much like the NoSQL movement did for data persistence.

Many thanks to Matthew Skelton, Peter Rodgers and Rob Elliot for helping me improve this post.




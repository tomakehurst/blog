---
layout: post
title: "A better microservices?"
date: 2014-05-01 21:56:00 +0100
comments: true
categories: [Microservices,Architecture,NetKernel,Erlang]
---

Microservices are gaining a lot of interest at the moment as an antidote to the limitations of monolithic application architecture. But as with all things in software (and life in general) they come with [significant tradeoffs](http://contino.co.uk/blog/2013/03/31/microservices-no-free-lunch.html). Arguably, monoliths and microservices occupy extreme points in a design space, and I've recently been wondering about other strategies that could yield similar benefits but with different tradeoffs.

<!-- more -->

Essentials
----------
Microservices are independently deployable, which means (provided other design constraints are met) that teams can ship new features independently from one another, avoiding the coordination cost associated with merging into a single artefact. Services should be single purpose (and therefore have only one reason at a time to change), so the release of feature A should never be delayed while feature B is still in progress.

Splitting system components into separate processes that can only communicate via a network enforces a degree of decoupling that is difficult to achieve in mainstream programming languages. Rich Hickey commented in a presentation (which I can't find now, annoyingly) that microservices encourage component interfaces based on immutable data with pass-by-value semantics, something he strongly advocates within single programs (and Clojure has unusually good support for). An avoidance of shared state (e.g. two services accessing the same DB schema) reinforces this separation.

This decoupling enables much greater freedom in technology choices. Services with APIs based entirely on platform-agnostic standards such as HTTP and JSON can be replaced transparently with something based on an entirely different platform. So when you've passed Peak NodeJS and decided that Go is the one true way or whatever, you can manage that transition incrementally rather doing a total re-write (which you should [Never Do](http://www.joelonsoftware.com/articles/fog0000000069.html)).

Finally, microservices are by default more operationally transparent than modules within a monolith. Of course it's possible to instrument and monitor individual components in a single app, but it's not common practice to do so and it requires explicit effort. By comparison, services that communicate via HTTP or messaging give away plenty of operational data for free.


Tradeoffs
---------
Benjamin Wootton's blog post (linked in the first paragraph) does a good job of explaining the additional costs of a microservice architecture, so I'll merely summarise the points I think are relevant.

Systems distributed via a network are subject to high latency and numerous failure modes absent when in-process. While some degree of distribution is inevitable in modern applications, microservices could be said to be *maximally distributed*, so these issues have to be considered for practically every bit of behaviour, even where there's no benefit in terms of scaling or resilience.

A corollary of the prevous point is that all (or most) service interactions should be asynchronous if you want to build a performant and available system. However, async programming is hard and even harder to test effectively. Also, asynchronous systems are more likely to develop emergent behaviours that are impossible to predict or test for, and that will almost certainly burn you in production at some point.

I'd also add that moving features between services is far more costly than between modules in a single program, particularly if those services were built on different technologies.

There are some other technologies that appear to exhibit at least some of the benefits described above, while potentially mitigating some of the problems. I'll caveat by saying that I've never built or run anything with these, so I can't personally vouch for whether they live up to their marketing!


Erlang
------
[Martin Fowler's article](http://martinfowler.com/articles/microservices.html) mentions in passing Erlang's commonality with microservices. Erlang supports hot code swapping, inter-process communication via asynchronous message passing and distribution over a network. This ticks many of the same boxes as microservices - independently deployable components, component boundaries enforced by immutable, data-centric communication protocols, no shared state.

A big additional advantage is that (many) processes can be physically co-located, avoiding the pain of distribution when it isn't required. Even when it is, Erlang OTP provides a good base for building in graceful failure recovery.

Erlang's model is of a much lower level of abstraction than a typical microservice's, however. It imposes no inter-process messaging semantics, whereas microservices are often resource-oriented (i.e. HTTP).

* could use it mixed e.g. over HTTP. would that defeat the point?
* not necessarily transparent

NetKernel
---------
[NetKernel](http://1060research.com/products) is a less well known but nonetheless very interesting option in this space. It bills itself as a "Uniform Resource Engine" and is the result of a piece of research analysing the web and the reasons for its outstanding success as a distributed computing platform (in a somewhat similar vein to Roy Fielding's work on REST).

Systems are built on NetKernel by composing resources exposed by modules. Resources are addressed (and address each other) via what could be considered a superset of HTTP semantics. A killer feature that this enables is highy efficient caching - since relationships between resources are transparent to NetKernel, it can walk the dependency graph when a resource's state changes and invalidate cache entries where necessary (implying that state can otherwise be cached indefinitely).

NetKernel modules are hot deployable, and can also be co-located on the same host. Distributed deployments are possible via the use of a proprietary network protocol.

* multi language, but you're stuck with the JVM
* synchronous
* transparancy


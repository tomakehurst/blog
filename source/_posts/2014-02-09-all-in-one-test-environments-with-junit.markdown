---
layout: post
title: "All-in-one test environments with JUnit"
date: 2014-02-09 13:19:50 +0000
comments: true
categories: [Java, Testing]
---

A major benefit of building on the JVM is the wide range of infrastructure built natively for it. Combined with first-class support for threading this allows entire scaled-down environments to be run in-memory for testing. There are numerous advantages to this, including:

* Faster feedback
* Consistent environment across the team
* Easy debugging - set breakpoints anywhere in the stack
* Write and run integration tests for your adapters (as in [Ports and Adapters](http://www.natpryce.com/articles/000772.html)) without needing a full environment

<!-- more -->

I've been regularly testing apps against [HSQL](http://hsqldb.org/) and [H2](http://www.h2database.com/) for quite a while, and [WireMock](http://wiremock.org/) was built specifically to provide in-memory REST services to test against. More recently I've been running [Cassandra](http://cassandra.apache.org/), [Zookeeper](http://zookeeper.apache.org/) and [Kafka](https://kafka.apache.org/) (all Apache projects) in-situ with my apps. Virtually everything I've built over the last
couple of years has been on top of [Dropwizard](http://dropwizard.io), which fits well with this model given its embedded web server and relatively quick startup time.

JUnit rules are an extremely handy way of managing environment components. For the unfamiliar, JUnit rule classes allow you to abstract away code you might otherwise put in your ``@BeforeClass``, ``@AfterClass``, ``@Before`` and ``@After`` methods. Most (all?) environment components are essentially daemons that need to be started, stopped and sometimes reset at the right moments, so rules are ideal wrappers to manage this lifecycle.

WireMock and Dropwizard ship with rule implementations, so including them in your setup is as simple as:

```java
@Rule
public WireMockRule wireMockRule = new WireMockRule();
```
and

```java
@ClassRule
public static DropwizardAppRule<MyAppConfiguration> RULE =
  new DropwizardAppRule<>(MyApplication.class,
                            "/path/to/testconfig.yaml");
```
For tools that don't come with their own rules, creating them can be achieved by subclassing JUnit's ExternalResource class e.g.

```java
public class SomeHipDbRule extends ExternalResource {

    private SomeHipDb db = new HipDb();
    
    @Override
    protected void before() throws Throwable {
        db.init();
        db.start();
    }
    
    @Override
    protected void after() {
        db.shutdown();
    }
}
```

I find it useful to hang test utility methods here too e.g.

```java
    public void clearData() {
        db.deleteAllData();
    }

    public void createSchema(String name) {
        db.createSchema(name);
    }
```

If you have many more than a couple of components in your environment, the number of rules required can become a bit unwieldy. Also, including each rule as a class field as in the above examples gives you no control over the order in which they're evaluated, which can be a problem if there are dependencies between the components (e.g. Kafka requires a Zookeeper server). JUnit's RuleChain solves both of these problems. On my current project we've created an "environment" class, also a test rule, which composes the individual components and imposes a specific ordering. This looks a bit like this:

```java
public class MyTestEnvironment implements TestRule {
    private final RuleChain ruleChain =
        RuleChain.outerRule(new ZookeeperRule())
                 .around(new KafkaRule())
                 .around(new HipDbRule());

    @Override
    public Statement apply(Statement base, Description description) {
        return ruleChain.apply(base, description);
    }
}
```

Gotchas
-------
Many libraries aren't built with this kind of usage in mind, so a little bit of extra complexity is sometimes required. Issues I've encountered are:

Cassandra manages some state in static variables, meaning that it survives restarts inside a single JVM. We worked around this by only starting Cassandra once per test run (by making the Cassandra server a static member of the rule and only starting it if it's not already running).

You'll be pulling lots of extra dependencies, and there will inevitably be clashes. I've been making heavy use of ``mvn dependency:tree`` while working on this. I'd like to find a good way to do cross-platform JVM forking as a solution to this and the previous point.

Where one component depends on another and startup happens on a background thread it's possible to start in a bad state. Polling and sleeps are often adequate if not great solutions to this.

Some servers have large or volatile startup times. I've found that the Kafka + Zookeeper combination can take between 10 seconds and over minute to stabilise, which makes it hard to get into a TDD groove.

Despite these obstacles I've found that the ability to run an embedded test environment is a valuable addition to a project.

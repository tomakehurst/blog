---
layout: post
title: "All-in-one test environments with JUnit"
date: 2014-02-09 13:19:50 +0000
comments: true
categories: 
---

A major benefit of being on the JVM is the wide range of infrastructure that built natively for it. Combined with first-class support for threading this allows entire (scaled-down) environments to be run in-memory for testing. There are numerous advantages to this - faster dev feedback cycles, consistent environments across the team and the ability to set a breakpoint anywhere in the stack, to name a few.

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
public class HipDbRule extends ExternalResource {

    private HipDb db = new HipDb();
    
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

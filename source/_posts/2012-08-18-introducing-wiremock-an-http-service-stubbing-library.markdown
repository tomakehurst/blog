---
layout: post
title: "Introducing WireMock – an HTTP service stubbing library"
date: 2012-08-18 12:14:45 +0000
comments: true
categories: 
---

[WireMock](http://wiremock.org) is a tool that allows HTTP exchanges to be stubbed and verified. It does this by creating an actual HTTP endpoint, rather than by stubbing or mocking the HTTP client class. It can be used directly from within JUnit (or your weapon of choice), run as a standalone process or deployed into a container with the aim of covering off a wide range of testing scenarios. It has a JSON API so you don't have to be working in JVM language to make use of it, although there is a also a fluent
Java API available if you are.

<!-- more -->

Other handy stuff it'll do includes conditional forwarding of requests to other services (enabling proxy/intercept), record/playback of stubs, fault injection, stateful behaviour and response delays.

Unit testing
------------

While it's possible to mock your HTTP client for unit testing with something like JMock or Mockito, I've found this to be problematic in practice. For example, how will my code cope with a 503 response? What if the Content-Type header doesn't match the body? Does it follow redirects (or not) as expected? Using mock objects, you have to attempt to figure out how your client will react to each of these scenarios and replicate this in your mock/stub setup. Given HTTP's complexity, it's
pretty likely you'll get some of it wrong. This is really just an example of why you [shouldn't mock types you don't own](http://www.mockobjects.com/2007/04/test-smell-everything-is-mocked.html).

Writing unit tests with WireMock is very straightforward using the fluent Java API. If using JUnit 4.x a @Rule can be added to your test to manage the startup, shutdown and reset of the server between tests.

``` java
@Rule
public WireMockRule wireMockRule = new WireMockRule();

@Test
public void exampleTest() {
     stubFor(get(urlEqualTo("/my/resource"))
        .withHeader("Accept", equalTo("text/plain"))
           .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "text/plain")
                .withBody("Some content")));

      Result result = myHttpServiceCallingObject.doSomething();

      assertTrue(result.wasSuccessFul());

      verify(postRequestedFor(urlMatching("/my/resource/[a-z0-9]+"))
        .withRequestBody(matching(".*1234.*"))
        .withHeader("Content-Type&", notMatching("application/json")));
}
```

Integrated testing
------------------

When testing an application end to end, I've found HTTP stubbing to be immensely useful. The real web services we typically have to integrate with are often slow, unreliable, unavailable in certain environments, or won't support automated creation of test data. Also, it's usually difficult to generate error responses on-demand. A consequence of these issues that I've often observed is that tests at this level only cover happy paths and are very non-specific in their assertions.

Running all your acceptance tests against real services all the time tends to be very slow, and personally I tend to agree with Dan Bodart's view that [the value of tests decreases rapidly as their execution time increases](http://dan.bodar.com/2012/02/28/crazy-fast-build-times-or-when-10-seconds-starts-to-make-you-nervous/).

WireMock supports a variety of integrated testing scenarios. It can be started as a standalone process and have stubs configured via a JSON API (file and HTTP based). It can also be wrapped up in a WAR file if you'd prefer to deploy it into an existing web container.

Starting WireMock standalone is as simple as this:

java -jar wiremock-1.24-standalone.jar

Then you can add some stubs via the JSON API (file or HTTP):

``` json
{
    "request": {
        "method": "GET",
        "urlPattern": "/some/resource/\d+"
    },
    "response": {
        "status": 200,
        "body": "Some content"
    }
}
```

More details are available on Github: https://github.com/tomakehurst/wiremock. Java artifacts are available in the Maven central repo.

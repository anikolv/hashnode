---
title: "Throughput, Throttling and how to implement it in Java"
datePublished: Thu Dec 24 2020 14:44:23 GMT+0000 (Coordinated Universal Time)
cuid: ckj2ylymx01wio1s169nhh1q2
slug: throughput-throttling-and-how-to-implement-it-in-java
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1608821177019/k_A5ygiWU.jpeg
tags: software-development, java, software-engineering

---

**What is Throughput?**

We all have heard the term **Throughput**. It's *that* non-functional attribute of a given system, which gives information about the maximum load it can process at a fixed period of time.  And since my preferred way of explaining and understanding software concepts is with real life analogies, the best possible analogy for Throughput is the water pipe. Each pipe has particular size in inches. Given that, each pipe is able to handle given maximum water flow per second. Its a number which the manufacturer determined with corresponding tests. If you try to overload the pipe with more water, unexpected side-effects may occur - something inside the water provisioning infrastructure might defect, the pipe might be blown away, in general all sorts of undesired behavior could be expected. You can even imagine the case where you must consume the water with whatever debit it is provided by your water supplier, without any manual regulation from the sink handle. It's just not going to work. Now, it is pretty much the same in the communication between distributed systems. Each system has a given capacity which is able to process at a given period of time. There is no such thing as infinite processing capacity. Whether it is established with performance and load tests, it doesn't matter, there is always a limit. Once the system gets overloaded, it might exhaust thread pools or connection pools, experience significant performance degradation and outages, experience concurrency race conditions, and many more.

**What is Throttling?**

As I mentioned in my analogy with the water pipes, we all have sink handles, which gives us the control over the water flow. Those devices are called throttles. Each device which controls certain fluid flow at a given rate is called a throttle. Having said that, **Throttling ** is the process of limiting the rate that an API is being invoked. Each API which takes place in a distributed environment must have some sort of Throttling mechanism implemented, being either out-of-the-box, provided by your framework, or custom solution. In this article I am briefing some of the possible Throttling solutions, applicable for Java applications.

**Guava RateLimitter**

The RateLimitter is a utility, provided by Google Guava library, which enables some quick and handy solutions for simple Throttling implementations.

First we need to add the guava dependency (example for Maven, last guava version available in Maven central):


```
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>29.0-jre</version>
</dependency>
``` 

Now, if we want to limit the execution rate of a given method, we first initialize the RateLimiter like this:


```
RateLimiter rateLimiter = RateLimiter.create(2);
``` 

What we are doing here is creating RateLimiter with maximum of 2 invocations per second for our method. In order this to work, each invocation must first acquire permission from the RateLimiter as follows:


```
rateLimiter.acquire();
processingMethod();
``` 

This means that we acquire permit from the RateLimiter before invoking our processingMethod(). The acquire() method basically checks if there are free slots for processing from the 5 limit we instantiated. If no, it uses waits behind the scenes to suspend the thread for a given time and resume it later. For more detailed information on the RateLimiter internals you can visit the guava documentation.

Pros:

 - easy to plugin, suitable for simple cases

Cons:

 - sync, blocking approach behind the scenes, suspending the throttled threads might lead to thread pool exhaustion in case of high overload

 - no retry mechamism

**Resilience4j Ratelimiter**

Resilience4j is a lightweight, easy-to-use fault tolerance library inspired by
Netflix Hystrix, as described by the official  [documentation](https://resilience4j.readme.io/docs/getting-started). It provides multiple modules, including Circuit Breaker, Rate Limiter, Retry and Bulkhead. I will brief only the Rate Limiter base capabilities.

First we add the dependency as follows:


```
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-ratelimiter</artifactId>
    <version>${resilience4jVersion}</version>
</dependency>
``` 
Next we have to define the in-memory registry as follows:


```
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.ofDefaults();
``` 

This line of code creates registry instance with default configurations. You can also build your own configuration using the RateLimiterConfig builder capabilities:


```
// Build config
RateLimiterConfig config = RateLimiterConfig.custom()
  .limitRefreshPeriod(Duration.ofSeconds(1))
  .limitForPeriod(10)
  .timeoutDuration(Duration.ofMillis(25))
  .build();

// Create registry
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.of(config);
``` 

Then you can get your RateLimitter from the registry as follows:


```
// Use registry with default config
RateLimiter rateLimiterWithDefaultConfig = rateLimiterRegistry
  .rateLimiter("name");

//User registry with your custom config
RateLimiter rateLimiterWithCustomConfig = rateLimiterRegistry
  .rateLimiter("name", config);
``` 

where name is the rateLimitter name which you define. Now we can use the rateLimitter to decorate out service method like this:


```
// Decorate your call to BackendService.doSomething()
CheckedRunnable restrictedCall = RateLimiter
    .decorateCheckedRunnable(rateLimiter, backendService::doSomething);

Try.run(restrictedCall)
    .andThenTry(restrictedCall)
    .onFailure((RequestNotPermitted throwable) -> LOG.info("Wait before call it again :)"));
``` 

What basically happens in this snippet is defining a decorator for out service doSomething() method which is the invoked vie the Try.run API. If its throttled, the onFailure handler fires. Key point here is that the throttled request is being blocked but for a given timeout period, configured via the RateLimiterConfig. There are also additional capabilities for subscribing to the rateLimitter events (success ot fail) as follows:


```
rateLimiter.getEventPublisher()
    .onSuccess(event -> logger.info(...))
    .onFailure(event -> logger.info(...));
``` 

In conclusion, Resilience4j provides nice throttling capabilities, along with other useful fault-tolerance modules which you might evaluate as well.

Pros:

 - flexible custom configuration of the throttling

 - async support for reactive programming

 - easy for plug-in, support for Spring Boot configuration

 - a lot of additional modules provided for resilience handling

 - supports retrying if Retry module included

Cons: 

 - it's mainly designed for functional programming, suites best for lambdas, functional interfaces and method references


**Spring Boot Throttling**

If you already use the Spring Boot ecosystem, you might definitely consider the **spring-boot-throttling** library which provides some handy out-of-the-box solutions for Throttling. In order to use the library, you first need to add it's repo and dependency as follows:


```
<repositories>
    <repository>
        <id>spring-boot-throttling-repo</id>
        <url>https://raw.github.com/weddini/spring-boot-throttling/mvn-repo/</url>
        <snapshots>
            <enabled>true</enabled>
            <updatePolicy>always</updatePolicy>
        </snapshots>
    </repository>
</repositories>

<dependencies>
  <dependency>
    <groupId>com.weddini.throttling</groupId>
    <artifactId>spring-boot-throttling-starter</artifactId>
    <version>0.0.9</version>
  </dependency>
</dependencies>
``` 
Once having this in place, you are enabled to use the following dedicated annotation:


```
@Throttling 
``` 

The implementation behind this annotation is slightly different from what we had in RateLimiter. Here we have non-blocking mechanism which purely rejects the request in case it exceeds the defined Throughput. Lets see an example.


```
@Throttling
public void serviceMethod() {
}
``` 

The default behavior of the Throttling annotation is allowing 1 method calls per SECOND for each unique *HttpServletRequest#getRemoteAddr()*. The above definition is the equivalent of


```
@Throttling(type = ThrottlingType.RemoteAddr, limit = 1, timeUnit = TimeUnit.SECONDS)
public void serviceMethod() {
}
``` 

You have the flexibility to configure your custom configuration, where the throttling type could be remoteAddress, header, cookie, username or custom *Spring Expression Language (SpEL)* rule. More details in this approach available in the official [documentation](https://github.com/shlomokoren/spring-boot-throttling).

Pros:

 - non-blocking throttling approach, exceeding requests are being rejected

 - easy to plugin for Spring Boot projects

 - relatively flexible configuration

Cons:

 - throttles only based on given criteria, not able to perform general throttle for a given method

- no retry mechamism


**Request Queues**

One of the best approaches for throttling which I have used in practice is the message queue approach. The idea is that all requests are being pushed to a queue, which itself then polls them at a given, pre-configured rate and sends them to the corresponding receiver. Of course there are a lot of variations of this approach, we have the publish-subscribe method, we have the polling method (client itself polling the messages at a given rate, inversing the communication). At a large scale this approach is also known as the "Messaging queue" pattern, known from the "Enterprise integration patterns" book. Here are some of the implementations which might be considered, depending on the context and the use case:

 - third-party message queue (ActiveMQ, RabbitMQ, etc). Most of them support some kind of throttling out of the box (also referenced as Flow Control)

 - custom middleware broker/gateway. You can implement custom broker which controls the flow by storing the events/requests in a queue and processes them at fully customizable rate and speed (using quartz scheduler jobs for example).

What I would like to accent here is that message queue solution can be implemented for any type of processing even inside your business logic (not even on API level). Any service which we might want to throttle could be placed behind some sort of message queue, being custom or third party one. Once you have your requests stored in a queue, you have full control over their processing (including retry policies, which is a must for a completely fault-tolerant solution).

**Conclusion**

This article aims to brief some basic ideas and approaches when dealing with throttling of either an API, or internal service method invocation. The exact approach depends on the context and the requirements, but I hope my quick summary might be helpful for your solution investigation.

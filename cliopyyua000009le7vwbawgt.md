---
title: "Java Multithreading explained: Project Loom and Virtual Threads [Part 4]"
datePublished: Fri Jun 09 2023 15:25:38 GMT+0000 (Coordinated Universal Time)
cuid: cliopyyua000009le7vwbawgt
slug: java-multithreading-explained-project-loom-and-virtual-threads-part-4
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686322659867/c8283c71-f673-4b1e-9689-077545475d5b.avif
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1686324329355/d3e29f10-408b-49f5-ad76-4bf37e9eb67e.avif
tags: java, multithreading, loom, virtual-threads

---

## Introduction

Project Loom is an ongoing project in the OpenJDK community that aims to significantly simplify concurrent programming for Java developers. The project was initiated around 2017 and is led by Ron Pressler from Oracle.

The main goals of Project Loom are:

1. **Simplify concurrent programming**: Project Loom introduces a new abstraction — virtual threads — that makes concurrent programming more approachable. These virtual threads are similar to user-mode threads, known as fibers, green threads, or coroutines, in other languages and runtimes.
    
2. **Improve efficiency and responsiveness**: By introducing lightweight, efficient threads (virtual threads), Project Loom aims to improve the efficiency and responsiveness of Java applications. Virtual threads have a much smaller footprint than traditional OS threads and can therefore be created in much larger numbers (millions or more). They are cheap to create and cheap to block, so developers don't need to worry about the cost of creating too many threads or blocking them.
    
3. **Preserve existing Java programming model**: One of the key goals of Project Loom is to introduce these new features without disrupting the existing Java programming model. This means that existing code should continue to work correctly when run on virtual threads, and developers can adopt the new features at their own pace.
    

One of the hot changes introduced within Project Loom is called *virtual threads*. It was first introduced in Java 19. Virtual threads are lightweight threads that dramatically reduce the effort of writing, maintaining, and observing high-throughput concurrent applications.

## Virtual threads in a nutshell

Today, every instance of [`java.lang.Thread`](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/Thread.html) in the JDK is a *platform thread*. A platform thread runs Java code on an underlying OS thread and captures the OS thread for the code's entire lifetime. The number of platform threads is limited to the number of OS threads.

A *virtual thread* is an instance of `java.lang.Thread` that runs Java code on an underlying OS thread but does not capture the OS thread for the code's entire lifetime. This means that many virtual threads can run their Java code on the same OS thread, effectively sharing it. While a platform thread monopolizes a precious OS thread, a virtual thread does not. The number of virtual threads can be much larger than the number of OS threads.

Virtual threads are a lightweight implementation of threads that is provided by the JDK rather than the OS. They are a form of *user-mode threads*, which have been successful in other multithreaded languages (e.g., goroutines in Go and processes in Erlang). User-mode threads even featured as so-called ["green threads"](https://en.wikipedia.org/wiki/Green_threads) in early versions of Java, when OS threads were not yet mature and widespread. However, Java's green threads all shared one OS thread (M:1 scheduling) and were eventually outperformed by platform threads, implemented as wrappers for OS threads (1:1 scheduling). Virtual threads employ M:N scheduling, where a large number (M) of virtual threads is scheduled to run on a smaller number (N) of OS threads.

## Scheduling virtual threads

To do useful work a thread needs to be scheduled, that is, assigned for execution on a processor core. For platform threads, which are implemented as OS threads, the JDK relies on the scheduler in the OS. For virtual threads, by contrast, the JDK has its own scheduler. Rather than assigning virtual threads to processors directly, the JDK's scheduler assigns virtual threads to platform threads (this is the M:N scheduling of virtual threads mentioned earlier). The platform threads are then scheduled by the OS as usual.

The JDK's virtual thread scheduler is a work-stealing [`ForkJoinPool`](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/concurrent/ForkJoinPool.html) that operates in FIFO mode. The *parallelism* of the scheduler is the number of platform threads available for the purpose of scheduling virtual threads. By default, it is equal to the number of [available processors](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/Runtime.html#availableProcessors()), but it can be tuned with the system property `jdk.virtualThreadScheduler.parallelism`. This `ForkJoinPool` is distinct from the [common pool](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/util/concurrent/ForkJoinPool.html#commonPool()) which is used, for example, in the implementation of parallel streams, and which operates in LIFO mode.

The platform thread to which the scheduler assigns a virtual thread is called the virtual thread's *carrier*. A virtual thread can be scheduled on different carriers throughout its lifetime; in other words, the scheduler does not maintain *affinity* between a virtual thread and any particular platform thread. From the perspective of Java code, a running virtual thread is logically independent of its current carrier:

* The identity of the carrier is unavailable to the virtual thread. The value returned by `Thread.currentThread()` is always the virtual thread itself.
    
* The stack traces of the carrier and the virtual thread are separate. An exception thrown in the virtual thread will not include the carrier's stack frames. Thread dumps will not show the carrier's stack frames in the virtual thread's stack, and vice-versa.
    
* Thread-local variables of the carrier are unavailable to the virtual thread, and vice-versa.
    

In addition, from the perspective of Java code, the fact that a virtual thread and its carrier temporarily share an OS thread is invisible. From the perspective of native code, by contrast, both the virtual thread and its carrier run on the same native thread. Native code that is called multiple times on the same virtual thread may thus observe a different OS thread identifier at each invocation.

The scheduler does not currently implement *time sharing* for virtual threads. Time-sharing is the forceful preemption of a thread that has consumed an allotted quantity of CPU time. While time sharing can be effective at reducing the latency of some tasks when there are a relatively small number of platform threads and CPU utilization is at 100%, it is not clear that time sharing would be as effective with a million virtual threads.

## Executing virtual threads

To take advantage of virtual threads, it is not necessary to rewrite your program. Virtual threads do not require or expect application code to explicitly hand control back to the scheduler; in other words, virtual threads are not *cooperative*. User code must not make assumptions about how or when virtual threads are assigned to platform threads any more than it makes assumptions about how or when platform threads are assigned to processor cores.

To run code in a virtual thread, the JDK's virtual thread scheduler assigns the virtual thread for execution on a platform thread by *mounting* the virtual thread on a platform thread. This makes the platform thread become the carrier of the virtual thread. Later, after running some code, the virtual thread can *unmount* from its carrier. At that point the platform thread is free so the scheduler can mount a different virtual thread on it, thereby making it a carrier again.

Typically, a virtual thread will unmount when it blocks on I/O or some other blocking operation in the JDK, such as `BlockingQueue.take()`. When the blocking operation is ready to complete (e.g., bytes have been received on a socket), it submits the virtual thread back to the scheduler, which will mount the virtual thread on a carrier to resume execution.

The mounting and unmounting of virtual threads happens frequently and transparently, and without blocking any OS threads. For example, the server application shown earlier included the following line of code, which contains calls to blocking operations:

```java
response.send(future1.get() + future2.get());
```

These operations will cause the virtual thread to mount and unmount multiple times, typically once for each call to `get()` and possibly multiple times in the course of performing I/O in `send(...)`.

The vast majority of blocking operations in the JDK will unmount the virtual thread, freeing its carrier and the underlying OS thread to take on new work. However, some blocking operations in the JDK do not unmount the virtual thread, and thus block both its carrier and the underlying OS thread. This is because of limitations at either the OS level (e.g., many filesystem operations) or the JDK level (e.g., [`Object.wait()`](https://download.java.net/java/early_access/jdk21/docs/api/java.base/java/lang/Object.html#wait())). The implementations of these blocking operations compensate for the capture of the OS thread by temporarily expanding the parallelism of the scheduler. Consequently, the number of platform threads in the scheduler's `ForkJoinPool` may temporarily exceed the number of available processors. The maximum number of platform threads available to the scheduler can be tuned with the system property `jdk.virtualThreadScheduler.maxPoolSize`.

Virtual threads are cheap and plentiful, and thus should never be pooled: A new virtual thread should be created for every application task. Most virtual threads will thus be short-lived and have shallow call stacks, performing as little as a single HTTP client call or a single JDBC query. Platform threads, by contrast, are heavyweight and expensive, and thus often must be pooled. They tend to be long-lived, have deep call stacks, and be shared among many tasks.

## Create a Java virtual thread

1. **Using the** `Thread.startVirtualThread` **Method:**
    
    This is the simplest way to create a virtual thread. Here's an example:
    
    ```java
    javaCopy codeThread.startVirtualThread(() -> {
        // Your task here
    });
    ```
    
    In this example, a new virtual thread is started, and the task inside the lambda function is executed in this thread.
    
2. **Using the** `Thread.Builder` **Class:**
    
    The `Thread.Builder` class provides a more flexible way to create virtual threads. Here's an example:
    
    ```java
    javaCopy codeThread.Builder.virtual().task(() -> {
        // Your task here
    }).start();
    ```
    
    In this example, a new virtual thread is started using the `Thread.Builder` class. The `task` method is used to specify the task to be executed in the thread, and the `start` method is used to start the thread.
    
3. **Using the** `ExecutorService` **Interface:**
    
    The `ExecutorService` interface provides a way to manage and control thread execution in a concurrent environment. Here's an example:
    
    ```java
    javaCopy codeExecutorService executor = Executors.newVirtualThreadExecutor();
    executor.execute(() -> {
        // Your task here
    });
    ```
    
    In this example, an `ExecutorService` is created using the `Executors.newVirtualThreadExecutor` method. The `execute` method is then used to start a new virtual thread and execute the task specified in the lambda function.
    

## Conclusion

In summary, virtual threads preserve the reliable thread-per-request style that is harmonious with the design of the Java Platform while utilizing the hardware optimally. Using virtual threads does not require learning new concepts, though it may require unlearning habits developed to cope with today's high cost of threads. Virtual threads will not only help application developers — they will also help framework designers provide easy-to-use APIs that are compatible with the platform's design without compromising on scalability.
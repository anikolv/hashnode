---
title: "Behind the Scenes: An Insight into Successful Java 11 to Java 17 Migration of multiple Spring Boot production apps"
seoTitle: "Java 11 to Java 17 migration"
seoDescription: "Java 11 to Java 17 migration"
datePublished: Thu Jun 22 2023 10:44:12 GMT+0000 (Coordinated Universal Time)
cuid: clj70n4sg000609mj3jlyg3px
slug: behind-the-scenes-an-insight-into-successful-java-11-to-java-17-migration-of-multiple-spring-boot-production-apps
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1687430604591/e10bf3c5-0d2b-4d23-87a5-cb562092a7e1.webp
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1687430636595/385bbdd0-5505-47f2-99c1-e890e8e34172.webp
tags: java, springboot, spring-framework, java11, java-17

---

## Introduction

In an ever-evolving technology landscape, staying up-to-date with the latest software upgrades is a cornerstone for maintaining robust and efficient applications. Among the many tools that developers employ, Java remains a steadfast presence, continually evolving to meet the changing demands of the software world. With its recent update, Java 17, the language offers a plethora of new features, performance enhancements, and improved security. But how does a business make the transition without disrupting its existing production services?

In this article, I dive into my recent experience of successfully migrating multiple **production**\-level **Spring Boot** applications with **Maven** from Java 11 to Java 17. The path to upgrade was neither smooth nor straightforward, but through careful planning, rigorous testing, and strategic execution, we successfully navigated the transition and reaped the benefits that Java 17 has to offer.

This article aims to provide an in-depth review of the actual migration process, challenges faced, solutions developed, and lessons learned. Our journey is marked by a blend of technical acumen and strategic planning, revealing the potential hurdles and demonstrating the methods to overcome them in such migrations. Whether you're a CTO considering the shift, a Project Manager planning the next steps, or a developer curious about the ins and outs of upgrading Java versions, this article has something for you.

## Why Java 17?

Before diving into the technical aspects of the migration, it's essential to understand the significance of Long-Term Support (LTS) and why it played a critical role in our decision to upgrade to Java 17 rather than upgrading straight to the latest Java 20 which was just freshly released during the process.

Java's version release cycle, governed by Oracle, involves a new version release every six months. Among these releases, some are earmarked as LTS (Long-Term Support) versions, which Oracle and the wider Java community support for a longer period - typically several years. LTS versions receive updates that include critical bug fixes, security patches, and stability improvements. Java 11, from which we migrated, was the previous LTS version before Java 17.

The stability of an LTS version like Java 17 comes from its mature state, having been refined through several iterations and patches since its initial release. This stability is crucial for production environments, where unexpected bugs or issues can lead to significant disruptions or, in worst-case scenarios, costly downtime.

On the other hand, the non-LTS releases, such as the fresh Java 20, are more akin to 'feature releases.' They introduce new functionalities and changes, but their support life is significantly shorter, typically lasting until the release of the next version. While these versions are exciting in terms of new features and improvements, they may not offer the same level of stability and reliability as an LTS version, which is critical for production applications.

Considering the importance of stability and long-term support for our production Spring Boot applications, Java 17 emerged as the obvious choice for our upgrade. Despite the potential benefits that Java 20 may bring, the assurance of extended support and proven stability that comes with an LTS version like Java 17 made it a more suitable choice for our scenario. This decision reflects a common practice in the industry to prefer LTS versions for production environments, where risk mitigation and long-term reliability are of utmost importance. However, we also left the door open for further migration towards the next LTS Java 21.

## Major features between Java 11 and Java 17

* **Java 12**
    
    * Switch Expressions (Preview): Enhanced the `switch` statement, allowing it to be used as an expression with a more simplified syntax.
        
    * JVM Constants API: An API to model nominal descriptions of key class-file and run-time artifacts.
        
    * Shenandoah: A Low-Pause-Time Garbage Collector (Experimental): A new garbage collector aimed at improving the performance of Java applications by reducing GC pause times.
        
* **Java 13**
    
    * Switch Expressions (Second Preview): Continued improvement and refinement of the switch expressions feature.
        
    * Text Blocks (Preview): Introduced multi-line string literal which simplifies writing HTML, XML, JSON, and SQL directly in Java code.
        
    * Dynamic CDS Archives: Enhanced the existing Class-Data Sharing ("CDS") feature to dynamically archive classes at the end of Java application execution.
        
* **Java 14**
    
    * Switch Expressions (Standard Feature): Graduated switch expressions from a preview feature to a permanent feature.
        
    * Pattern Matching for `instanceof` (Preview): Simplifies common coding patterns by allowing `instanceof` operator to be used with pattern matching.
        
    * Helpful NullPointerExceptions: Improved the usability of `NullPointerExceptions` by describing precisely which variable was `null`.
        
    * Records (Preview): Provides a compact syntax for declaring classes which are intended to be simple "data carriers".
        
* **Java 15**
    
    * Text Blocks (Standard Feature): Graduated Text Blocks from a preview feature to a permanent feature.
        
    * Sealed Classes (Preview): Allows the author of a class or interface to control which code is responsible for implementing it.
        
    * Edwards-Curve Digital Signature Algorithm (EdDSA): A new cryptographic signature scheme, offering better performance and security over older schemes.
        
    * Hidden Classes: Classes that cannot be used directly by the bytecode of other classes.
        
* **Java 16**
    
    * Pattern Matching for `instanceof` (Standard Feature): Graduated Pattern Matching for `instanceof` from a preview feature to a permanent feature.
        
    * Records (Standard Feature): Graduated Records from a preview feature to a permanent feature.
        
    * Sealed Classes (Second Preview): Continued refinement and improvement of the sealed classes feature.
        
* **Java 17**
    
    * Sealed Classes (Standard Feature): Graduated Sealed Classes from a preview feature to a permanent feature.
        
    * Strong encapsulation of JDK internals: Except for critical internal APIs such as `sun.misc.Unsafe`, all internal elements of the JDK are now strongly encapsulated.
        
    * Deprecating and removing older features and components: Several older and seldom-used features like the Applet API, the Security Manager, and the RMI Activation System have been deprecated and/or removed.
        

## **Defining the migration scope: Choosing the right candidates for the upgrade**

Deciding which of our Spring Boot applications to upgrade was the first critical step in our journey from Java 11 to Java 17. The decision was guided by several key factors, including the size of the application, its coupling with other services, the estimated time required for the migration and team capacity.

1. **Size of the Applications:** The size of an application significantly impacts the migration process, affecting both the required time and the complexity of the upgrade. While smaller applications might appear to be ideal for migration due to their lower complexity, larger (especially legacy) applications can provide more substantial benefits from the upgrade, thanks to the performance enhancements and new features of the updated Java version.
    
2. **Coupling of the Applications:** Coupling refers to the degree of interdependence between different applications or services. Highly coupled applications often need to be updated together to avoid potential compatibility issues. Therefore, in deciding our migration candidates, we also had to consider the logical coupling between applications. If two or more applications were tightly coupled, it made more sense to upgrade them together to ensure consistent functionality and avoid potential conflicts.
    
3. **Time Estimation:** Estimating the time for the migration process was another crucial consideration. The upgrade's benefits need to be balanced against potential disruptions to our ongoing operations. We estimated the time required for the complete migration process for each application, which included planning, execution, testing, and dealing with any potential issues that might arise. This helped us ensure that we had adequate resources available and could maintain our service levels during the upgrade.
    

The applications we chose to upgrade were a combination of small and large, each presenting different challenges and benefits. By considering these factors, we made informed decisions about which applications to migrate and when achieving a balance between leveraging the new Java features and maintaining stability and functionality in our production environment. In the following sections, we'll delve deeper into the specific challenges we encountered during each application's upgrade and how we navigated through them.

## **Feasibility study and preliminary research**

Before we dived into the migration process, a feasibility study was conducted to understand the implications of upgrading the core framework - Spring and Spring Boot - to support Java 17. This study was crucial for understanding what changes we needed to make to our applications, how these changes would impact our codebase, and what steps we needed to follow to successfully upgrade our Spring Boot applications to the new Java version. We also were aware that once we upgrade Spring Boot, the Java 17 compiler will tell us what else needs to be upgraded.

**Spring & Spring Boot upgrade**

The first critical aspect of the feasibility study was to identify which version of Spring Boot supports Java 17. After going through the official Spring Boot documentation and release notes, we found that the support for Java 17 was introduced in [**Spring Boot 2.5.5**](https://docs.spring.io/spring-boot/docs/2.5.5/reference/html/getting-started.html#getting-started.system-requirements). However, according to [Maven Central Repository](https://mvnrepository.com/artifact/org.springframework.boot/spring-boot/2.5.5), all versions between 2.5.5 and 2.6.7 have Spring Core CVE vulnerabilities. That's why we decided to go with [**Spring Boot** **2.6.8**](https://docs.spring.io/spring-boot/docs/2.6.8/reference/html/getting-started.html#getting-started.system-requirements) **(with Spring Core 5.3.20)** which seemed like the first potential stable candidate which supports Java 17. The decision to migrate to this Spring Boot version was additionally backed-up by our initial idea to **avoid unnecessary major version upgrades which might creep the scope of the migration.**

**Spring migration guides and release notes**

After establishing the target Spring Boot version, we researched comprehensive migration guides and official [release notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes) for Spring Boot 2.6. Unfortunately, there are no explicit migration guides, but the release notes provided us with valuable insights into what changes we could expect in this version, what new features it offered, and any potential issues we might encounter during the migration. This research was instrumental in helping us understand the full scope of the upgrade and preparing us for the tasks ahead.

**Java 17 migration guides and release notes**

We also made ourselves familiar with the official [Oracle JDK Migration Guide](https://docs.oracle.com/en/java/javase/17/migrate/getting-started.html#GUID-C25E2B1D-6C24-4403-8540-CFEA875B994A) which highlights the significant changes and enhancements done in JDK 17.

This guide contains the following sections:

* [Significant Changes in the JDK](https://docs.oracle.com/en/java/javase/17/migrate/significant-changes-jdk-release.html#GUID-561005C1-12BB-455C-AD41-00455CAD23A6)
    
* [Security Updates](https://docs.oracle.com/en/java/javase/17/migrate/security-updates.html#GUID-D68C70D5-721F-46CB-AF56-A5E9302C0FB1)
    
* [Removed APIs](https://docs.oracle.com/en/java/javase/17/migrate/removed-apis.html#GUID-FAD4E80D-64BA-42AC-A682-38D06EE61AC6)
    
* [Removed Tools and Components](https://docs.oracle.com/en/java/javase/17/migrate/removed-tools-and-components.html#GUID-D7936F0D-08A9-411E-AD2F-E14A38DA56A7)
    
* [Preparing For Migration](https://docs.oracle.com/en/java/javase/17/migrate/preparing-migration.html#GUID-5657F44A-B2D7-4FB6-AAD7-295AC4533ABC)
    
* [Migrating From JDK 8 to Later JDK Releases](https://docs.oracle.com/en/java/javase/17/migrate/migrating-jdk-8-later-jdk-releases.html#GUID-7744EF96-5899-4FB2-B34E-86D49B2E89B6)
    
* [Next Steps](https://docs.oracle.com/en/java/javase/17/migrate/next-steps.html#GUID-8074FBBD-92DF-4C46-837D-86D1D55DEC84)
    

**Can we upgrade Java and Spring smoothly?**

We needed to understand where are the JDK Compiler and Spring Boot versions defined in our code. In our applications, those versions were configured in the `pom.xml` of a project which was a parent for all Java applications. Oops..this means that upgrading in the parent `pom.xml` file would effectively kickstart a migration for hundreds of Java apps that rely on that parent. What we realized was this approach is not agile. Each Java project should have the freedom to upgrade any of its dependencies without affecting other projects. The solution we picked up was dropping the dependency to this parent maven project and moving all dependencies inside the projects we want to upgrade.

## Local migration execution

Once we had the preliminary idea of what should be performed and how do we start, we picked up one of our target projects and performed the following steps:

**Build migration**

* install JDK 17 and verify it
    
    ```bash
    java -version
    java version "17.0.6" 2023-01-17 LTS
    Java(TM) SE Runtime Environment (build 17.0.6+9-LTS-190)
    Java HotSpot(TM) 64-Bit Server VM (build 17.0.6+9-LTS-190, mixed mode, sharing)
    ```
    
* install JDK 17 in your IDE
    
* move all dependencies from the parent maven project to our `pom.xml` file and replace the old parent with
    

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.6.8</version>
</parent>
```

* upgrade Maven compiler version
    

```xml
<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
```

* bump all Spring Boot-related dependencies to 2.6.8 and Spring Core-related dependencies to 5.3.20
    
* rebuild project
    
* bump versions of other dependencies in case of compile errors
    
* whenever you need to upgrade a given Maven dependency, choose the lowest possible version which supports Java 17 (look for the release date in Maven Central, for example - Java 17 was released 14th of September 2021, use the dependency version right after this date). Upgrade certain dependency to the highest possible version might introduce a decent amount of breaking changes that you just do not need to handle for the sake of this migration.
    
* Spring Boot 2.6 introduces **Circular References Prohibited by Default** policy which causes the Spring application context failing to start if such are present. More info on how to solve it in [Spring Boot 2.6 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes#circular-references-prohibited-by-default). In summary - a quick fix is applying `spring.main.allow-circular-references=true` in your application properties, the suggested approach is to refactor your spring beans references once you have time for that
    
* reiterate until green build is achieved
    

**Runtime migration**

* it is generally a better and faster approach if you perform a local startup of the application and solve potential runtime issues before deploying it to a real environment
    
* in our case, we had Cucumber based integration tests which emulate successfully our application at runtime
    
* if you have issues with local startup of your app, a possible approach is to add a dummy unit test with Spring runner which at least initializes the Spring context
    
* most of the issues we encountered were related to non-existing classes or methods being created/invoked at runtime, which resulted in additional maven dependencies being bumped with the strategy I already mentioned
    

## CI/CD migration (Jenkins & Docker-based)

Our CI/CD pipeline is Jenkins based, so in case you are planning to upgrade from Java 11 to Java 17 and you are using Jenkins Pipeline (also known as "Jenkinsfile"), there are several areas you might need to pay attention to:

* **Java Version Configuration:** The Java version that Jenkins uses is generally configured at the system level or at the project level. Ensure that Jenkins, and the system it is running on, can support and run Java 17. If your Jenkinsfile specifies the Java version explicitly, for example, in a `tools` block or in the build environment setup, you will need to update that version to '17'.
    
* **JDK Tool Configuration:** If you're using the `jdk` tool in your Jenkinsfile, you need to make sure that a JDK for Java 17 is properly installed and configured on your Jenkins server. You might need to update the JDK tool configuration accordingly.
    
* **Build Scripts:** If your Jenkinsfile invokes other build scripts (like Ant, Maven, Gradle, etc.), ensure these scripts are updated to use Java 17.
    
* **Docker Images:** If your Jenkins Pipeline uses Docker, and the Docker images include a specific JDK version, you need to update those Docker images to use JDK 17.
    
* In a **Dockerfile**, the base image is the main component you'll need to change when upgrading from Java 11 to Java 17.
    
    For example, if you're using the OpenJDK image for Java 11, your Dockerfile might start with something like:
    
    ```javascript
    dockerfileCopy codeFROM openjdk:11-jdk
    ```
    
    To upgrade to Java 17, you'll simply change this to:
    
    ```javascript
    dockerfileCopy codeFROM openjdk:17-jdk
    ```
    
    Note that this assumes you're using the OpenJDK Docker images. If you're using a different base image, you'll need to find the equivalent for Java 17.
    
    In some cases, you might also need to make changes to your Dockerfile if you are relying on features of the JVM or Java libraries that have changed between Java 11 and Java 17.
    
    Remember to thoroughly test your Docker images and the application after making these changes to ensure that your application still works as expected with Java 17
    
* **Java arguments**
    
    One of the arguments we needed to change was related to remote debugging: to allow a debugger to attach to a running Java Virtual Machine (JVM). The change was from
    
    ```bash
    -Xrunjdwp:transport=dt_socket,server=y,address=*:5101
    ```
    
    To
    
    ```bash
    -agentlib:jdwp=transport=dt_socket,server=y,address=0.0.0.0:5101
    ```
    
    The difference between them lies in their age and level of standardization:
    
    1. `-Xrunjdwp` is the older, non-standard way of enabling remote debugging in Java. The `-X` prefix designates options that are non-standard and subject to change without notice. This option uses the Java Debug Wire Protocol (JDWP) to communicate with the debugger. This option has been deprecated in recent versions of Java.
        
    2. `-agentlib:jdwp` is the newer, standardized way of enabling remote debugging in Java. This option is also using JDWP, but it does so in a manner that's expected to be supported across different JVM implementations and versions.
        

In our case, we also needed to add this argument:

```bash
--add-opens=java.base/java.lang=ALL-UNNAMED
```

The `--add-opens` option is a JVM argument introduced in Java 9 as part of the Java Platform Module System (JPMS). It's used to increase the accessibility of the classes in a specific module or package.

Here's a breakdown of `--add-opens=java.base/java.lang=ALL-UNNAMED`:

* `--add-opens`: This is the command that modifies the module access controls to "open" a package.
    
* `java.base/java.lang`: This is the module/package combination that's being opened. Here, it's the `java.lang` package in the `java.base` module. The `java.base` module contains fundamental classes and interfaces that are always implicitly imported and available for use.
    
* `ALL-UNNAMED`: This specifies which module gets access to `java.base/java.lang`. The `ALL-UNNAMED` value represents all modules that are not named, including the classpath. This means that all code running from the classpath will have access to the internals of `java.lang`.
    

In effect, `--add-opens=java.base/java.lang=ALL-UNNAMED` allows all code running from the classpath to access (via reflection or otherwise) the internals of the classes in the `java.lang` package, even if those internals are not normally accessible due to module encapsulation.

This option can be necessary when you're using libraries that haven't been updated to be fully compatible with the Java Platform Module System introduced in Java 9. Such libraries might still need access to internal APIs that are no longer accessible by default.

However, `--add-opens` is generally seen as a workaround. The preferred solution is to update or replace the libraries that require such access, if possible.

## Deployment and regression testing

Once we had a green build and we managed to fix all runtime issues, the rollout plan followed this path:

* deployment on the testing environment, verify startup is successful
    
* active monitoring period to verify there are no broken async processes that we didn\`t manage to fix earlier
    
* active regression testing of some form - could be automation testing, manual QA verification of core system flows or whatever makes us comfortable that our service core flows are operational without regressions
    
* passive monitoring period - letting the service work for some time without actively testing it, just scanning the logs (alerts/dashboards/alarms/metrics) for errors periodically
    
* roll-out services to staging and production environments, reiterating the cycle according to your needs
    

## Conclusion

In conclusion, the migration from Java 11 to Java 17 brings with it a host of enhancements, feature updates, and performance improvements that are set to augment Java's robustness and efficiency. With the move to a six-month release cycle, Oracle has given developers quicker access to new features, while also ensuring that critical bug fixes and security patches are provided more rapidly.

However, it's important to bear in mind the potential challenges posed by the migration, including compatibility issues, deprecated APIs, and the need to become familiar with new programming paradigms. Furthermore, features like the sealed classes and pattern matching are still in the preview stage as of Java 17, meaning they could undergo further refinements based on feedback from the developer community.
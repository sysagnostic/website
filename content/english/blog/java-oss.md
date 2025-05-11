---
draft: true
author: David Lakatos
title: Migrating production Java applications to Eclipse Temurin
description: null
publishDate: null
date: 2025-05-11
Image_webp: images/blog/java-oss-cover.webp
image: images/blog/java-oss-cover.jpg
tags:
  - java
  - foss
  - temurin
  - compiler
  - jit
  - hotspot
  - aot
  - oracle
  - migration
---

Free and open-source software not only reduces operational costs, but reduces strategy and legal liabilities as well. Oracle's Hotspot JDK is the perfect example where liabilities exclusively originate from the vendor's behavior and its product's non-FOSS nature.


## The Oracle JDK nightmare

In 2019, Oracle Corporation incrementally destroyed its corporate image in the eyes of its customers, and the Java engineering community overall by transforming Oracle JDK to a 100% payware software. Trying to undo the PR nightmare around Java, the red company introduced a new license called NFTC which promised some free to use releases for Oracle JDK. Unfortunately for the Java community, Oracle phrased NFTC's licensing terms in an ambiguous manner from the legal perspective. As [whichjdk.com](https://whichjdk.com/#oracle-java-se-development-kit-jdk) accurately summarizes, the phrase "internal business operations" is undefined by Oracle and may lead to costly lawsuits for enterprises. As a result, the new policies increased licensing fees, and created legal risks for enterprise customers. In the meantime, Oracle migrated to a subscription based Java licensing strategy, and also changed the license fee calculation rules, resulting in a legally booby trapped and expensive chaos for enterprise customers.

## Migration to FOSS

The Java platform is highly standardized due to Sun Microsystem's (Java's original manufacturer) Java Community Process. The Java Bytecode format and the Java Virtual Machine is 100% standard, There are not many conceptual aspects to the migration from Oracle Hotspot JDK to Eclipse Temurin. Here's a short list where migration is probably explicitly required for your application:

- JavaFX
- Java Web Start
- Java Flight Recorder (JFR) & Java Mission Control (JMC)

Although the list of explicitly required migration steps are short, there are many aspects for a safe and successful Java runtime migration.


## Performance

Oracle does not publish the full list of OpenJDK customizations its Hotspot runtime incorporates. Increased garbage collector algorithm efficiency, enhanced code cache memory segmentation implementation, etc. We just don't know how big of a downgrade we may experience when switching to Eclipse Temurin.

The only way to ensure proper system performance is through rigorous performance testing, like executing long-running soak tests, or running quick-peek stress tests, and measure our application's preformance difference compared to the Oracle JDK executions. Tune the JVM system parameters if unacceptable performance degradation is experienced during testing until the system meets its expected performance.


## Skeletons in the closet

Some Java packages are packaged into the Oracle JDK due to its implementation details. These code segments are not part of the Java Virtual Machine specification, nor any other Java specification out there. Before the introduction of the [Java Platform Module System](https://www.oracle.com/corporate/features/understanding-java-9-modules.html), our Java application could reach these non-standard Java packages without additional effort, and our software developers could build business logic on top of them. Working around the [JPMS](https://www.oracle.com/corporate/features/understanding-java-9-modules.html) errors is always easier, so dirty configuration workarounds may hide in our production systems. Until we want to migrate to a different JDK vendor.

Skeletons fall out of the closet when we start our application on Eclipse Temurin with the same Java parameters we used for decades. Packages and classes are not found during application startup. As a quick 'n dirty solution, application dependencies added as external Java libraries usually resolve these issues. A more proper way is to analyze what business logic is related to the skeletons, and decide how to resolve the issue one by one.


## Garbage collection

As per Java 24, Oracle JDK has no unique garbage collection algorithm that Eclipse Temurin does not have. Although, Temurin ships the Shenandoah GC not present in the Oracle JDK, so exploring the possibility to use Shenandoah is available after the successful migration.


### Compilation

Ahead-of-Time (AOT) compiled applications, like Oracle GraalVM based applications are compiled and executed in a very different way compared to just-in-time (JIT) compiled applications. Migration from GraalVM to Temurin is not only a switch of a runtime vendor but a change of paradigm regarding the application. Things get more complicated with GraalVM's newly introduced JIT compiler.

Changes in compilation paradigm require special design iterations. As a rule of thumb, switching between AOT and JIT compilation should not be in scope of a Java runtime vendor migration.

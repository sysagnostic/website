---
draft: true
author: David Lakatos
title: Just-in-time - Part 2
description: Case study on how great out of the box technology can lead to laziness.
#publishDate: 2025-03-12
date: 2025-03-12
Image_webp: images/blog/just-in-time-cover.webp
image: images/blog/just-in-time-cover.jpg
tags:
  - java
  - tuning
  - cache
  - jit
  - testing
  - performance
---

In [Part 1](/blog/just-in-time-part1/) of this blog post series, we discovered how the Java platform relates to other ones regarding compilation and execution, compared compilation strategies, and identified hot code in Java bytecode. Let's continue the journey by exploring Java's JIT internals.

## Code cache

Let's review the compilation and execution diagram once again, now with only focusing to Java.

![Java code execution](/images/blog/just-in-time-java-exec.svg)

JIT compiles the platform agnostic Java Bytecode to platform dependent machine code. The compiled machine code is stored in a temporary storage called code cache, which is located in virtual memory. The code cache is empty during JVM startup, and its content is discarded when the Java process exits. Like in most caches, newly compiled native code may override an older machine code segment. The code cache is a designated memory area of the JVM (like metaspace, stack, heap, etc.), it similarly has the ability to grow until it reaches the configured maximum size, although code cache shrinking is not implemented by most JVMs. Since the JVM caches native code here, the code cache is considered part of the native JVM memory area (like metaspace).

Code cache memory area may either be continuous or segmented, depending on the JVM's version and vendor. Continuous code cache is the traditional, simple approach that fits generic use-cases. On the other hand, segmented cache design is more advanced; it aims to reduce cache fragmentation and increase performance by disecting the memory area to non-method, profiled, and non-profiled (see [oracle.com](https://docs.oracle.com/en/java/javase/21/vm/java-hotspot-virtual-machine-performance-enhancements.html#GUID-85BA7DE7-4AF9-47D9-BFCF-379230C66412) for details).

## Code Profiling

Compiled native code can either be profiled or non-profiled. Profiled native code collects statistics when executed. These collected statistics can be later used for optimization purposes, e.g. fine-tuned branch prediction heuristics. Since profiling involves the overhead of collecting statistics, which statistics data also take up some memory eventually, non-profiled native code has better execution performance, and lower memory footprint over profiled native code.

## Tiered compilation

Client and server JVMs have different goals regarding code execution. Client VMs prefer fast startup, low memory footprint, and low latency. On the other hand, server VMs prefer eventually high throughput and performance. The JIT compiler has a lot of configuration options that govern these JVM preferences (see [Tuning options](#tuning-options)). Client VMs use the client compiler, while server VMs use the server compiler by design. Specific compilers are tuned to produce native code that corresponds with the code execution goals of client and server VMs.

Without tiered compilation, the JVM either uses the client or the server compiler based on the VM's corresponding settings. The client compiler produces non-profiled non-optimized native code quickly. On the other hand, the server compiler outputs non-profiled optimized native code through a slower compilation process that involves optimization steps.

Tiered compilation unites the benefits of client VM and server VM preferences. It aims to start up quickly, incrementally optimize hot code, while maintaining a reduced memory footprint. In order to achieve its goals, tiered compiler flags all bytecode segments with one of the following related compilation level designators.

1. Interpreted: Bytecode execution is profiled and emulated by the interpreter. The collected profiling statistics are stored in the code cache.
1. C1: Profiled native code is compiled by the client VM compiler. Native code is stored in the code cache for further use, along with the collected profiling statistics.
1. C2: Non-profiled native code is compiled by the server VM compiler. Native code is stored in the code cache for further use. The code is non-profiled, so no statistics are collected.

Eventually, tiered compilation achieves faster startup, better overall server performance (longer code profiling, more accurate data to fuel compiler heuristics), and lower code cache memory footprint.

## Commonly used JIT tuning JVM options

This chapter explains the Hotspot JVM's JIT related JVM options. For specific JVM default option values, please refer to your JVM vendor's documentation.

- `-Xint`: Disable all compilers, only use the interpreter. May be used for performance debugging, never use it in production.
- `-XX:CICompilerCount`: Manually set the number of compiler threads. Should only be used to work around OS/CPU detection JVM bugs.
- `-XX:ReservedCodeCacheSize`: Maximum size of the code cache.
- `-XX:[+|-]TieredCompilation`: Toggle tiered compilation.
- `-XX:+AggressiveOpts`: Enable experimental performance optimization features, including JIT related ones.
- `-XX:[+|-]BackgroundCompilation`: Determines if compilers should execute without blocking the execution of the Java application. Used for testing scenarios, where deterministic execution is important.
- `-XX:CompileThreshold`, `-Xcomp`: Only compile methods that has already been invoked in an interpreted manner the number of compile threshold times. `-Xcomp` disables the interpreter, and effectively means `-XX:CompileThreshold=0`.
- `-XX:InitialCodeCacheSize`: Size of the empty code cache during JVM startup.
- `-XX:+Inline`, `-XX:+PrintInlining`: JIT replaces method bodies in native code. Traditionally, the stack is used for passing arguments, while jumps and returns redirect code execution to the code of the invoked method. Inlining replaces the use of stack, jumps, and returns with the copy-pasted native code of the invoked method. Inlining offers better native code execution performance by introducing the tradeoff of code cache content duplication. Usually only very hot methods are inlined, so the optimization technique can produce code with an acceptable performance gain vs. increased memory footprint tradeoff.
- `-XX:InlineSmallCode`: Limits how much native code can a method compile to that can still be inlined.
- `-XX:[+|-]UseAES`, `-XX:[+|-]UseAESIntrinsics`, `-XX:[+|-]UseSHA`, `-XX:[+|-]UseSHA1Intrinsics`, `-XX:[+|-]UseSHA256Intrinsics`, `-XX:[+|-]UseSHA512Intrinsics`: JIT should compile native code that uses available hardware accelerated processor instructions (e.g. for TLS).
- `-XX:[+|-]UseCodeCacheFlushing`: Determines if a full code cache should be flushed or not. A full code cache leads to a disabled compiler.
- `-XX:[+|-]UseSuperWord`: Determines if the technique of vectorization should be used for executing repetitive tasks that can be executed in parallel for better execution performance.


## Unpleasant wake up from the JIT comfort

Most JIT features are non-deterministic by design. It is not determined in build time how just-in-time compilation will happen. It is not determined just before compilation what result will compilation produce. Everything is determined just-in-time based on branch prediction heuristics, statistics, and profiling. This is a scary situation for software engineers, verification engineers, system engineers, eventually for everyone who is responsible for a software to work as expected and succeed as a product.

The situation is not as bad as it sounds until the experience with JIT is the same as with essential civil engineering infrastructure elements. The "If it works, it works" kind of mindset eases software engineer nerves to a certain level that everyone even forgets about JIT's existence, including its non-deterministic nature.

Software products mature by time, they are usually extended with new features and more dependencies, leading to an ever growing code base (software shrinks rarely in code volume). All looks good in UAT, new version of the product is released into production. For days, maybe even for a week, the production system works as expected. Suddenly, a major production outage covers all observability dashboards in red, all horizontally scaled instances seem to crash or slow down dramatically one after another. What happened? JIT gradually filled its code cache, so - depending on the configuration - the JVM exited or continued to emulate execution of bytecode in an interpreted manner.

This kind of issue is the most common and destructive pitfall of JIT: the just-in-time compiler's undeterministic, but too good out of the box behavior. In this example, no one performed soak tests that would be able to identify JIT related weaknesses, nor did anyone monitor JIT's internal state to early detect a growing issue in the production system. As I said, JIT being too good can lead to engineer laziness.

## Avoiding JIT pitfalls

Based on our extensive Java related project experience, we always recommend our customers to keep the following rules of thumb in mind:

1. Check your code and dependencies if dynamic code generation may occur during runtime, since it may fill the JVM code cache with once frequently but later never used machine code. If dynamic code generation is a vital part of the product, increase the `-XX:CompileThreshold` to a number where no once frequently but later never used bytecode is compiled to native code.
1. Run continuous soak tests for a few days before releasing into production to detect sneaky non-functional issues.
1. Closely monitor code cache metrics in all systems, and set proper alert thresholds in the alerting system.
1. Closely monitor your product's response times, also response time deviations, and set proper alert thresholds in the alerting system.
1. Use the same memory configuration in test and production systems.
1. Size your code cache memory properly, so no code cache flushing should be necessary to happen in the production system.
1. Use the `-XX:+UseCodeCacheFlushing` JVM option anyway if not enabled by default in your JVM, so unintended code cache usage behavior will not lead instantly to a production brownout or outage.
1. Make sure your JVM detects your OS appropriately if it's a client or server (verify `java -XshowSettings:vm --version` output), because defaults are calculated based on the kind of VM. If the detection is faulty, override the kind of VM with `-client` or `-server` JVM options appropriately.

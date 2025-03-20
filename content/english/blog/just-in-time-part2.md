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

In [Part 1](/blog/just-in-time-part1/) of this blog post series, we discovered how the Java platform relates to others in terms of compilation and execution, we compared compilation strategies, and identified hot code in Java bytecode. Let's continue the journey by exploring Java's JIT internals.

## Code cache

Let's review our compilation and execution diagram once again, now focusing on Java only.

![Java code execution](/images/blog/just-in-time-java-exec.svg)

JIT compiles the platform-agnostic Java bytecode to platform-dependent machine code. The compiled machine code is stored in a temporary storage called code cache, which is located in virtual memory. The code cache is empty during JVM startup, and its contents are discarded when the Java process exits. Like in most caches, newly compiled machine code may override an older machine code segment in the code cache. 

The code cache is a designated memory area of the JVM, just like metaspace, stack and heap. It has the ability to grow until it reaches the configured maximum size, although shrinking the code cache is not implemented by most JVMs. Since the JVM caches native code here, the code cache is considered part of the native JVM memory area (like metaspace).

The code cache memory area may either be contiguous or segmented, depending on the JVM's version and vendor. Contiguous code cache is the traditional, simple approach, which fits generic use-cases. Segmented cache design is more advanced; it aims at reducing cache fragmentation and increasing performance by dividing the memory area into non-method, profiled, and non-profiled areas (see [oracle.com](https://docs.oracle.com/en/java/javase/21/vm/java-hotspot-virtual-machine-performance-enhancements.html#GUID-85BA7DE7-4AF9-47D9-BFCF-379230C66412) for details).

## Code Profiling

Compiled native code can either be profiled or non-profiled. Profiled native code collects statistics when executed, and statistics are stored in the code cache. These collected statistics can be later used for optimization purposes, e.g. fine-tuned branch prediction heuristics. Since profiling involves the overhead of collecting statistics, and statistics data also take up some memory eventually, non-profiled native code has better execution performance and lower memory footprint.

## Tiered compilation

The JVM can operate in client or server mode. Client and server JVMs have different goals regarding code execution. Client VMs prefer fast startup, low memory footprint, and low latency. Server VMs, on the other hand, prefer eventually high throughput and performance. The JIT compiler has a lot of configuration options that govern such JVM preferences (see [Tuning options](#tuning-options)). Client VMs use the client compiler, while server VMs use the server compiler by design. Specific compilers are tuned to produce native code that corresponds to the code execution goals of client and server VMs.

Without tiered compilation, the JVM either uses the client or the server compiler, based on the VM's settings. The client compiler produces non-profiled, non-optimized native code quickly. On the other hand, the server compiler outputs non-profiled, optimized native code through a slower compilation process that involves optimization steps.

Tiered compilation unites the benefits of client VM and server VM preferences. It aims at starting up quickly and optimizing hot code incrementally, while maintaining a reduced memory footprint.

In order to achieve its goals, tiered compiler flags each bytecode segment with a compilation level designator, as discussed below. By using levels of optimization, tiered compilation achieves faster startup, faster convergence to an eventually equal overall server performance (longer code profiling and more accurate data to fuel compiler heuristics).

Code segments are optimized by incrementing their assigned levels. The algorithm of how and when certain bytecode segments are compiled via tiered compilation is out of scope for this article due to its high complexity. 

Here are the different compilation levels:

### Level 0: Interpreted

Bytecode execution is profiled and emulated by the interpreter. The collected profiling statistics are stored in the code cache.

### Level 1-3: C1

C1 compilation may be separated into 3 levels, all performed by the client VM compiler, but those levels differ in profiling settings. At level 1, trivial methods without the possibility of further optimization are non-profiled. Levels 2 and 3 provide some sort of profiled native code, without significant optimization efforts. Native code is always stored in the code cache for further use, along with the collected profiling statistics, if collected.

### Level 4: C2

Non-profiled native code is compiled by the server VM compiler. Native code is stored in the code cache for further use. The code is non-profiled, so no statistics are collected.


## Commonly used JVM options for JIT tuning

This chapter explains the Hotspot JVM's JIT-related options. For the default values of specific JVM options, please refer to your JVM vendor's documentation.

- `-Xint`: Disable all compilers, only use the interpreter. May be used for performance debugging, never use it in production.
- `-XX:CICompilerCount`: Manually set the number of compiler threads. Should only be used to work around OS/CPU detection JVM bugs.
- `-XX:ReservedCodeCacheSize`: Maximum size of the code cache.
- `-XX:[+|-]TieredCompilation`: Toggle tiered compilation.
- `-XX:+AggressiveOpts`: Enable experimental performance optimization features, including JIT related ones.
- `-XX:[+|-]BackgroundCompilation`: Determines if compilers should execute without blocking the execution of the Java application. Used for testing scenarios, where deterministic execution is important.
- `-XX:CompileThreshold`, `-Xcomp`: Only compile methods that have already been invoked in an interpreted manner the number of compile threshold times. `-Xcomp` disables the interpreter, and effectively means `-XX:CompileThreshold=0`. Ignored by tiered compilation.
- `-XX:InitialCodeCacheSize`: Size of the empty code cache during JVM startup.
- `-XX:+Inline`, `-XX:+PrintInlining`: JIT replaces method bodies in native code. Traditionally, the stack is used for passing arguments, while jumps and returns redirect code execution to the code of the invoked method. Inlining replaces the use of stack, jumps, and returns with the copy-pasted native code of the invoked method. Inlining offers better native code execution performance by introducing the tradeoff of code cache contents duplication. Usually only very hot methods are inlined, so the optimization technique can produce code with acceptable performance gain, at the expense of  an increased memory footprint.
- `-XX:InlineSmallCode`: Limits how much native code can a method compile to that can still be inlined.
- `-XX:[+|-]UseAES`, `-XX:[+|-]UseAESIntrinsics`, `-XX:[+|-]UseSHA`, `-XX:[+|-]UseSHA1Intrinsics`, `-XX:[+|-]UseSHA256Intrinsics`, `-XX:[+|-]UseSHA512Intrinsics`: JIT should compile native code that uses available hardware accelerated processor instructions (e.g. for TLS).
- `-XX:[+|-]UseCodeCacheFlushing`: Determines if a full code cache should be flushed or not. A full code cache leads to a disabled compiler.
- `-XX:[+|-]UseSuperWord`: Determines if the technique of vectorization should be used for executing repetitive tasks that can be executed in parallel for better execution performance.


## Unpleasant wake-up from your JIT dreams

Most JIT features are non-deterministic by design. How just-in-time compilation will happen is not determined in build time. The result the compilation will produce is not determined just before compilation. Everything is determined just-in-time, based on branch prediction heuristics, statistics, and profiling. That is a scary situation for software engineers, verification engineers, and systems engineers; virtually anyone responsible for software to work as expected and succeed as a product.

The situation is not so bad if your experience with JIT is the same as with essential civil engineering infrastructure elements. The "if it works, it works" kind of mindset eases software engineers' nerves to a certain level. One might even forget about JIT's existence, including its non-deterministic nature.

Everything looks good in UAT, and a new version is released into production. For days, maybe even for a week, the production system works as expected. Suddenly, a major production outage covers all observability dashboards in red: all horizontally scaled instances seem to have crashed or to be slowing down dramatically, one after another. What is happening? JIT has gradually filled its code cache, so – depending on the configuration – the JVM exited or continued to emulate execution of bytecode in an interpreted manner.

This kind of issue is the most common and destructive pitfall of JIT: the just-in-time compiler's undeterministic, but too good out-of-the-box behavior. In this example, no-one has performed soak tests that would identify JIT-related weaknesses, nor has anyone monitored JIT's internal state to early detect an exacerbating situation in the production system. As I said, JIT being too good can lead to engineer laziness.

## Avoiding JIT pitfalls

Based on our extensive Java-related project experience, we always recommend our customers to keep the following rules of thumb in mind:

1. Check your code and dependencies if dynamic code generation may occur during runtime, since it may fill the JVM code cache with once frequently, but later never used machine code. If dynamic code generation is a vital part of the product, increase the `-XX:CompileThreshold` to a number where no once frequently, but later never used bytecode is compiled to native code.
1. Run continuous soak tests for a few days before a production release to detect sneaky non-functional issues.
1. Closely monitor code cache metrics in all systems, and set up proper thresholds in the alerting system.
1. Closely monitor your product's response times as well as response time deviations, and set up proper thresholds in the alerting system.
1. Use the same memory configuration in test and production systems.
1. Size your code cache memory properly so that no code cache flushing should be necessary in the production system.
1. Use the `-XX:+UseCodeCacheFlushing` JVM option anyway, if not enabled by default in your JVM, so that unintended code cache usage behavior would not lead to an instant production brownout or outage.
1. Make sure your JVM detects your OS appropriately if it's a client or server (verify `java -XshowSettings:vm --version` output), because defaults are calculated based on the kind of VM. If the detection is faulty, override the kind of VM with `-client` or `-server` JVM options appropriately.

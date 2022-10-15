---
layout: post
title:  "What I talk when I talk about coverage"
date:   2022-10-15 00:07:12 +0800
tags: tiny-detail cxx fuzzing
categories: blog
layout: post
---


## Preface

After having been inactive in blog writing for over a year, I decided to resume my blogging habit by starting something simpler, i.e., a series called ["the tiny detail"](../../tag/tiny-detail).
I will be talking about something very tiny, which is easier to write for me, and hopefully, it will be easier to read for you.
The contents of this series will be mostly about some confusing details (mostly in fuzzing) that I found many people are not aware of, but could have some impact in building and evaluating a technique (e.g., fuzzer).


We'll start with a very common topic: [**"coverage"**](https://en.wikipedia.org/wiki/Code_coverage), and specifically code coverage in C++.

## Huh.. Which coverage?

I am not going to repeat how to distinguish coverage in shapes of "lines", "branches" and "functions", or explaining fancy ones like "K-context-sensitive edge" etc.
Instead, I am talking about two coverage types at the implementation level:

1. Source coverage.
2. CFG coverage.

### Source coverage

Source-level coverage is most commonly used for evaluating the quality of a test suite, which can be evaluated by tools like [gcov](https://gcc.gnu.org/onlinedocs/gcc/Gcov.html) and [llvm-cov](https://clang.llvm.org/docs/SourceBasedCodeCoverage.html).
By definition, every unique source-level coverage identifier are distinguished by a uniqe set of **AST** tokens (or source-code regions) in C++.

It has the following features:
- **Readability**: The coverage can be directly mapped to the source code (e.g., line number and branch ID) that can be interpreted by human easily. 
- **Offline**: It (gcov or llvm-cov) is often implemented in a way that you cannot get the coverage information while the program is running; This means you cannot use it for coverage-guided fuzzing out of the box.
- **Slow**: It could be slow. There are a few reasons:
   1. It covers all and exact source code coverage, and is often not compressed for readability in coverage reports.
   2. The coverage reports can be flushed into the disk to be persistent, which could be slow. But in practice, the flushing is registered as a callback in [`atexit`](https://en.cppreference.com/w/cpp/utility/program/atexit) which could only be called when the program exits.
   3. The coverage counter are mapped to the source code, which puts more constraints on compiler optimization;

Also see [here](https://releases.llvm.org/15.0.0/tools/clang/docs/SourceBasedCodeCoverage.html#impact-of-llvm-optimizations-on-coverage-reports):

> llvm optimizations (such as inlining or CFG simplification) should have no impact on coverage report quality. This is due to the fact that the mapping from source regions to profile counters is immutable, and is generated before the llvm optimizer kicks in. The optimizer can’t prove that profile counter instrumentation is safe to delete (because it’s not: it affects the profile the program emits), and so leaves it alone.

### CFG coverage

CFG here stands for the [control flow graph](https://en.wikipedia.org/wiki/Control_flow_graph) in [LLVM IR](https://llvm.org/docs/LangRef.html).
That is, instead of looking at the source files, we look at it in a graph of basic blocks.

How do we implement this? From the high-level, the most naive implementation is to insert a tiny counter function at the beginning of each basic block (it could also be edge but in this article we only say "basic block" for simplicity) so that the counter can be incremented when the corresponding basic block is executed.
This can be implemented by writing a LLVM pass or simply use LLVM's [Coverage Sanitizer](https://clang.llvm.org/docs/SanitizerCoverage.html).

Fuzzing is also a game of speed.
However, the overhead of tracking all basic blocks can be expensive (like [TVM](https://github.com/apache/tvm) has 1~2 million edges), especially the program could be optimimized via very aggressive function inlining.
Therefore, in practice, instead of regarding every as an entry, LibFuzzer use a [hash bit map](https://github.com/llvm/llvm-project/blob/92fb310151d2b1e349695fc0f1c5d5d50afb3b52/compiler-rt/lib/fuzzer/FuzzerValueBitMap.h) to only track at most `1 << 16` bits.

But you may ask why we need CFG coverage for fuzzing?
In my opinion, the source coverage is designed for coverage evaluating which focuses on interpretibility, while CFG coverage is tailored for fuzzing which focuses on performance.
We can of course use source-level coverage while still having runtime feedback and hashing tricks, but it is just not implemented in that way.
Meanwhile, keeping coverage at the CFG level could allow us to do more customization/extension.
For example, we can easily adding coverage support for a new LLVM-based language.
Meanwhile, writing a pass or extension under the LLVM IR abstraction should be easier than doing that in the source level (say Clang).

However, it might not be a good idea to use CFG coverage for coverage evaluation because it is not accurate if the bit map is compressed.
Even if we use uncompressed bit map, the "side" effect of compiler passes will transform the original naive CFG to an optimized but sophisticated one, making the result hard to interpret.
For example, aggressive function inlining could be applied to make the coverage distribution biased.
Meanwhile, using source-level coverage brings normalization by default (but it could also be biased the software has a lot of copy-n-paste code).

## Takeaways: When and which?

Here's my current rules of thumb:
- To perform coverage benchmarking, use source coverage.
- To develop coverage-guided fuzzing, use CFG coverage.

That said, if you build a coverage-guided fuzzer and want to evaluate its coverage in, say 24 hours:

- Compile the fuzz target with **CFG-only** coverage instrumentation and run the fuzzer for 24 hours. Save all the intermediate tests and their correpsonding time stamp.
- Compile the target with **source-only** coverage instrumentation and replay the tests to get the source-level coverage report. Then use its original time stamp to represent the coverage when the test is executed.

## Closing

The title of this article is "stolen" from Haruki Murakami's book: [What I talk when I talk about running](https://en.wikipedia.org/wiki/What_I_Talk_About_When_I_Talk_About_Running).
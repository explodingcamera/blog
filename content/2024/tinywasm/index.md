---
title: "TinyWasm: How I wrote my own WebAssembly Runtime"
description: "Looking back at the development of TinyWasm, a small WebAssembly runtime written in Rust."
date: 2024-10-13
---

After a short hiatus from writing on this blog, I'm back with an update on what I've been working on lately (or rather slowly catching up on the backlog of posts I wanted to write).

I finally finished writing my bachelor's thesis on WebAssembly and Edge Computing this summer. More on that in a later post,
but for now, I wanted to talk about another project I worked on earlier this year that inspired much of the work I did
for my thesis: [TinyWasm](https://github.com/explodingcamera/tinywasm), a fully compliant WebAssembly runtime written in Rust.

## <u>**TinyWasm**</u>

When writing my posts on [OS Development](https://blog.henrygressmann.de/series/rust-os/) last year, I got interested in WebAssembly
and wanted to try it out inside the kernel. I was fed up with writing context-switching and memory management code,
and WebAssembly looked like an easy way to run existing code in the operating system.
I looked at the existing interpreters and compilers for WebAssembly, but they were all either too complex or had too many dependencies for my taste (I was going for embedded systems, so it had to be lightweight).

With just a bit of prior experience with WebAssembly and compilers/interpreters, I now had the topic for my capstone project:
Building a WebAssembly runtime. I've been pretty burned on a lot of (unnecessarily) complex projects in the past, so to keep myself on track,
I decided to set out some constraints at the start to finish it on time:

1. **No Platform-Specific Code**: My first goal was to remove all dependencies on platform-specific code so
   everything could work in Rust's `no_std` environment (and potentially in my OS). This also meant I was limited in the libraries I could use.
   Thankfully, an excellent crate for parsing WebAssembly binaries already existed:
   [`wasmparser`](https://github.com/bytecodealliance/wasm-tools). At the time, it didn't support `no_std`, but I was able to fork it and make it work.

2. **Build the MVP**: Focus on the initial version of WebAssembly, so no threads, no SIMD, no garbage collection, etc.

3. **Keep it simple**: I wanted the codebase to be as small and readable as possible to make it easier to integrate into other projects,
   such as my OS. No premature optimization (There has already been a [fork of TinyWasm](https://github.com/reef-runtime) used as a base for a distributed WebAssembly runtime).

4. **No Unsafe Code**: This came a bit later, but I decided to avoid unsafe Rust code entirely (maybe something for another post). While this excludes some optimizations like using virtual memory to optimize bounds checking in Wasm memory, it also forces me to write simpler code.

I started by taking a simple "Hello World" WebAssembly program and tried to infer everything I needed without looking at the specification.
Surprisingly, this worked well, and in a short time, I had a simple interpreter that could run very basic programs.

{{ figure(caption = "The first test version of the interpreter.", position="center", src="./assets/code.jpg", link="https://github.com/explodingcamera/tinywasm/blob/93f8e10a8c15cbcf0d09517869016c32c6bd47eb/crates/tinywasm/src/module/mod.rs#L131-L185") }}

With this newly gained confidence, I scrapped the initial codebase and started from scratch.
Starting by defining the structure of the interpreter and the different components it would need, I quickly realized that
I would need a lot of tests to make sure everything worked as expected.
Thankfully, I didn't have to write all of these tests myself, as the reference interpreter conveniently already has
[thousands of them](https://github.com/WebAssembly/testsuite) covering a lot of edge cases. Plumbing these tests into my test suite was a bit
of a pain, but in the end, I had a script that would run all of the relevant tests and give me a nice graph of
how many tests I had passed (and some dopamine when the number went up).

{{ figuresvg(caption = "", position="center", src="content/2024/tinywasm/assets/progress-mvp.svg") }}

Now that I had a good test suite, I started implementing the interpreter. The WebAssembly specification is thorough,
but it's also dense with abstract concepts and mathematical notation.
I spent a lot of time looking at different interpreters and their APIs to get a better understanding of how things were supposed
to work (I can recommend the trusty [grep.app](https://grep.app/) for this).

For the actual implementation, I mainly started by taking a couple of tests from one of the test suites and trying to get them to pass,
which worked surprisingly well. Slowly but surely, the numbers went up, and more and more tests passed.

Predictably, once I reached only about 80/2000+ test cases left, I still had about 20% of my work and
a couple of long nights ahead of me. Finally, once all the tests passed, I compiled the interpreter to WebAssembly
and ran it using TinyWasm. It worked on the first try. I was completely surprised, but LLVM randomly did the right optimizations that made it work,
and its code didn't trigger any of the remaining edge cases/bugs.

## <u>**Optimization**</u>

Once I had a (mostly) working interpreter, I started looking into profiling and optimizing the code. I had a few ideas on how to make it faster,
but I wanted to optimize only the parts that were slow and not add any additional complexity to the codebase.
I started by profiling the interpreter using `perf`, `cargo-flamegraph` and later `samply` to understand where the bottlenecks were. To keep things going in the right direction,
I also added some basic benchmarks using `criterion` to ensure I didn't accidentally make things slower.

{{ figure(caption = "A flamegraph using Firefox's profiler & samply", src="./assets/flamegraph.jpg") }}

Initially, the biggest overhead was matching opcodes in the interpreter loop. Without using unsafe code,
I had to nudge the compiler in the right direction to generate jump tables for the opcodes. Thankfully, a couple of
`#[inline(always)]` annotations and some moving code around did the trick, giving me a nice +50% speedup.
There's not a lot of information on how to do this in Rust, but a [post](https://pliniker.github.io/post/dispatchers/)
hints at this probably being the easiest cross-platform way to do it.

From there, I also looked into reducing the size of the bytecode and the interpreter itself. TinyWasm uses a custom bytecode format
that's a bit easier to execute than the standard WebAssembly format and can be zero-copy deserialized (powered by [`rkyv`](https://github.com/rkyv/rkyv)).
This bytecode is represented by a big enum with all the different opcodes and their arguments, and without any optimizations, it was about 32 bytes per instruction.
To reduce this, I removed some redundant information that could be inferred from the context and added more specialized opcodes for common patterns (Super Instructions).
Currently, the bytecode is about 16 bytes per instruction, which is a nice improvement to memory usage and performance due to better memory alignment.

Currently, The biggest bottleneck is the stack, mainly `push` and `pop` operations. I'm currently looking into ways to optimize this, but it's tricky without using unsafe code. Other runtimes, such as [wasmi](https://wasmi-labs.github.io/blog/posts/wasmi-v0.32/), show that register-based interpreters are much faster. However, I'm not sure if I want to go down that route yet, as it would add a lot of complexity to parsing, and I'd like to stay
as close to the original WebAssembly model as possible.

For actual performance, I'm currently at about 1/3 of the speed of wasmi, which is pretty good considering the size of the codebase. These benchmarks are not available online yet as this was part of my thesis, but whenever I get around to cleaning them up, I'll publish them on GitHub as well.

## <u>**Conclusion**</u>

I was super happy with the results, and I'm still pushing the odd update here and there. The next step is SIMD support (currently in the works), for which I recently refactored the stack to use a more efficient representation (SoA for differently sized types). After that, I'll look into adding threads and moving to support the WebAssembly System Interface (WASI). However, I'm waiting for the spec to stabilize before I start implementing it.

As of now, TinyWasm supports WebAssembly V2 (without SIMD and threads) and several other proposals, such as reference types and bulk memory operations, so most programs should work fine. After submitting it as my capstone project, I also posted it on HN and Reddit, where I got some nice feedback and a few stars on GitHub (obviously the most important part).

{{ figure(caption = "Internet points are important.", position="center", src="./assets/hn.jpg", link="https://news.ycombinator.com/item?id=39627410") }}

If you're interested in checking it out or maybe even contributing, TinyWasm is up
on [GitHub](https://github.com/explodingcamera/tinywasm) and also on [crates.io](https://crates.io/crates/tinywasm).
Feel free to poke around, open issues, or even submit a PR (I recently improved the test suite and added a small contribution guide).

## <u>**Further Reading**</u>

[Crafting Interpreters](https://craftinginterpreters.com/) by Robert Nystrom is probably the best introduction to the field.
Going from there, I can also recommend the [Writing an Interpreter in Go](https://interpreterbook.com/)/[Writing a Compiler in Go](https://compilerbook.com/)
books or the [Writing Interpreters in Rust Guide](httphttps://rust-hosted-langs.github.io/book/). I mostly looked at the source code of other interpreters, though, so don't be scared by all the theory. Simple interpreters are surprisingly easy to write, and a small one can be a great weekend project.

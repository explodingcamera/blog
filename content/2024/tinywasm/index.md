---
title: "TinyWasm: How I wrote my own WebAssembly Runtime"
date: 2024-10-06
draft: true
---

<!--
Talk about how big scary words and jargon don't matter. Bruteforceing your way through works. don't be afraid to throw away code. **Be** afraid to ask for help: with the right mindset, you can figure it out yourself (might not work for everyone ?).
 -->

After a long hiatus from writing on this blog, I'm back with a small update on what I've been working on lately (or rather slowly catchin op on the backlog of posts I wanted to write).

This summer, I finally finished writing my bachelor's thesis on the topic of WebAssembly and Edge Computing. More on that (probably) in a later post, but for now, I wanted to talk about another project I worked on earlier this year that inspired a lot of the work I did for my thesis: TinyWasm.

## <u>**TinyWasm**</u>

When writing my posts on [OS Development](https://blog.henrygressmann.de/series/rust-os/) last year, I got really interested in the topic of WebAssembly and wanted to try it out inside of the kernel. I was kind of got fed up with writing context switching and memory management code and WebAssembly it looked like an easy way to run existing code in the operating system.
I had a look at the existing interpreters and compilers for WebAssembly, but they were all either too complex or had too many dependencies for my taste (I was going for embedded systems, so it had to be lightweight).

So, without only minimal prior experience with both WebAssembly and compilers/interpreters, I now had the topic for my capstone project: A tiny WebAssembly runtime. I've been pretty burned on a lot of (unneccessarily) complex projects in the past, so, to keep myself on track, I decided to set out some constraints at the start to actually finish it on time:

1. **No Platform-Specific Code**: My first goal was to remove all dependencies on platform-specific code, so everything could work in Rust's `no_std` environment. This also meant that I couldn't use many of the existing libraries for WebAssembly. Thankfully, a really good crate for parsing WebAssembly binaries already existed: [`wasmparser`](https://github.com/bytecodealliance/wasm-tools) (however with no `no_std` support at the time).

2. **Build the MVP**: Just focus on the initial version of WebAssembly, so no threads, no SIMD, no garbage collection, etc.

3. **Keep it simple**: I wanted the codebase to be as small and readable as possible to make it easier to integrate into other projects such as my OS. No premature optimization.

4. **No Unsafe Code**: This one came a bit later, but I decided to avoid unsafe Rust code entirely (maybe something for another post).

I started out by taking a simple "Hello World" WebAssembly program and tried to infer everything I needed from there. Surprisingly, this worked pretty well and in a super short amount of time. This gave me some slightly missplaced confidence that everything was going to be smooth sailing from here on out.

{{ figure(caption = "I couldn't find a good XKCD comic for this, so here's one on graphs instead.", position="center", src="./assets/image.png", invert=true, mixblend=true, link="https://www.xkcd.com/688/") }}

With this newly gained confidence, I scrapped the initial codebase and started from scratch. Starting with a simple public API, I did something very unlike me: TDD. Now, I'm sorry for subjecting you to this, but essentially following a long document that outlines everything about WebAssembly (the [specification](https://webassembly.github.io/spec/core/index.html)), you just have to write tests for everything. Thankfully, I didn't actually have to write any of these tests myself, as the reference interpreter conveniently already has thousands of them covering a lot of edge cases (that are also often not clear from the specification). Plumbing these tests into my own test suite was a bit of a pain, but in the end I had a sweet script that would run all of the relevant tests and give me a nice graph of how many tests I had passed (and some dopamine hits when the number went up).

{{ figuresvg(caption = "", position="center", src="content/2024/tinywasm/assets/progress-mvp.svg") }}

Now was about the time my newly found confidence started to decline.

Not to throw the blame on the specification, but it's hard to grasp as someone from the outside. A lot of abstraction, (probably?) needless complexity and leaving things up to the implementer. I spent about a week just looking at different interpreters and their APIs to get a better understanding of what I was supposed to do. Most of the time, I just ended up taking a couple of tests from one of the test suites and trying to get them to pass, which worked suprisingly well. Slowly but surely, the numbers went up and more and more tests passed.

Predictably, once I reached only about 80/2000+ tests left, I still had about 20% of my work and a couple of long nights ahead of me. But finally, once all tests passed, I just compiled the interpreter to WebAssembly and ran it using TinyWasm itself. It worked on the first try. I was completely confused, but I guess LLVM randomly did the right optimizations that made it work.

I was super happy with the results, and I'm still pushing the odd update here and there. As of now, it's at the point of supporting WebAssembly V2 (without SIMD and threads) and a couple of proposals. I also posted it on HN and Reddit, where I got some nice feedback and a couple of stars on GitHub (obviously the most important part).

{{ figure(caption = "Internet points are important.", position="center", src="./assets/hn.jpg", link="https://news.ycombinator.com/item?id=39627410") }}

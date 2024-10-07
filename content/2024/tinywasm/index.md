---
title: "TinyWasm: How I wrote my own WebAssembly Runtime"
date: 2024-10-06
draft: true
---

<!--
Talk about how big, scary words and jargon don't matter. Brute forcing your way through works. Don't be afraid to throw away code. **Be** afraid to ask for help: with the right mindset, you can figure it out yourself (might not work for everyone ?).
 -->

After a long hiatus from writing on this blog, I'm back with a small update on what I've been working on lately (or rather slowly catching up on the backlog of posts I wanted to write).

I finally finished writing my bachelor's thesis on WebAssembly and Edge Computing this summer. More on that in a later post, but for now, I wanted to talk about another project I worked on earlier this year that inspired much of the work I did for my thesis: TinyWasm.

## <u>**TinyWasm**</u>

When writing my posts on [OS Development](https://blog.henrygressmann.de/series/rust-os/) last year, I got really interested in WebAssembly and wanted to try it out inside the kernel. I was fed up with writing context-switching and memory management code, and WebAssembly looked like an easy way to run existing code in the operating system.
I looked at the existing interpreters and compilers for WebAssembly, but they were all either too complex or had too many dependencies for my taste (I was going for embedded systems, so it had to be lightweight).

With minimal prior experience with WebAssembly and compilers/interpreters, I now had the topic for my capstone project: A tiny WebAssembly runtime. I've been pretty burned on a lot of (unnecessarily) complex projects in the past, so, to keep myself on track, I decided to set out some constraints at the start to actually finish it on time:

1. **No Platform-Specific Code**: My first goal was to remove all dependencies on platform-specific code so everything could work in Rust's `no_std` environment. This also meant that I could only use a few of the existing libraries for WebAssembly. Thankfully, an excellent crate for parsing WebAssembly binaries already existed: [`wasmparser`](https://github.com/bytecodealliance/wasm-tools) (however, with no `no_std` support at the time).

2. **Build the MVP**: Focus on the initial version of WebAssembly, so no threads, no SIMD, no garbage collection, etc.

3. **Keep it simple**: I wanted the codebase to be as small and readable as possible to make it easier to integrate into other projects, such as my OS. No premature optimization.

4. **No Unsafe Code**: This came a bit later, but I decided to avoid unsafe Rust code entirely (maybe something for another post).

I started by taking a simple "Hello World" WebAssembly program and tried to infer everything I needed. Surprisingly, this worked well and worked in a short amount of time. This gave me some slightly misplaced confidence that everything would be smooth sailing from here on out.

<!--
```rust
let mut local_values = vec![];
for (i, arg) in args.iter().enumerate() {
 let (val, ty) = arg.to_bytes();
 if locals[i] != ty {
 return Error::other(&format!("Invalid argument type for {}, index {}: expected {:?}, got {:?}" func_name, i, locals[i], ty));
 }
 local_values.push(val);
}

let mut stack: Vec<Vec<u8>> = Vec::new();
while let Some(op) = body.next() {
 match op.unwrap() {
 Operator::LocalGet { local_index } => stack.push(local_values[local_index as usize].clone()),
 Operator::I64Add => {
 let a = i64::from_le_bytes(stack.pop().unwrap().try_into().unwrap());
 let b = i64::from_le_bytes(stack.pop().unwrap().try_into().unwrap());
 stack.push((a + b).to_le_bytes().to_vec());
 }
 Operator::I32Add => {
 let a = i32::from_le_bytes(stack.pop().unwrap().try_into().unwrap());
 let b = i32::from_le_bytes(stack.pop().unwrap().try_into().unwrap());
 stack.push((a + b).to_le_bytes().to_vec());
 }
 Operator::End => {
 info!("stack: {:#?}", stack);
 return Ok(returns.iter().map(|ty| WasmValue::from_bytes(&stack.pop().unwrap(), ty)).collect::<Vec<_>>());
 }
 _ => {}
 }
}
``` -->

{{ figure(caption = "The first test version of the interpreter.", position="center", src="./assets/code.jpg", link="https://github.com/explodingcamera/tinywasm/blob/93f8e10a8c15cbcf0d09517869016c32c6bd47eb/crates/tinywasm/src/module/mod.rs#L131-L185") }}

With this newly gained confidence, I scrapped the initial codebase and started from scratch. Beginning with a simple public API, I did something unlike me: TDD. Essentially, following a lengthy document that outlines everything about WebAssembly (the [specification](https://webassembly.github.io/spec/core/index.html)), you just have to write tests for everything. Thankfully, I didn't actually have to write any of these tests myself, as the reference interpreter conveniently already has [thousands of them](https://github.com/WebAssembly/testsuite) covering a lot of edge cases (that are also often not clear from the specification). Plumbing these tests into my own test suite was a bit of a pain, but in the end, I had a script that would run all of the relevant tests and give me a nice graph of how many tests I had passed (and some dopamine when the number went up).

{{ figuresvg(caption = "", position="center", src="content/2024/tinywasm/assets/progress-mvp.svg") }}

Now was about the time my newly found confidence started to decline.

Around this time, my initial overconfidence started to wane. The WebAssembly specification is thorough, but it's also dense and filled with abstract concepts that aren't immediately helpful when you're trying to write actual code. I spent a lot of time looking at different interpreters and their APIs to get a better understanding of how something was supposed to work (I can recommend the trusty [grep.app](https://grep.app/) for this).

Most of the time, I just took a couple of tests from one of the test suites and tried to get them to pass, which worked surprisingly well. Slowly but surely, the numbers went up and more and more tests passed.

Predictably, once I reached only about 80/2000+ test cases left, I still had about 20% of my work and a couple of long nights ahead of me. Finally, once all the tests passed, I just compiled the interpreter to WebAssembly and ran it using TinyWasm itself. It worked on the first try. I was completely confused, but LLVM randomly did the right optimizations that made it work, and its code didn't trigger any of the remaining edge cases/bugs.

## <u>**(Premature) Optimization**</u>

- i wanted good perf
- profiling tools
- simplifying the code
- a lot of array indexing due to Rust's memory model
- minimizing copies
- reducing struct sizes so they fit into pointers
- reducing the bytecode size (custom bytecode format)
- reducing reference counted values
- nudging the compiler in the right direction (jump tables for opcodes)
- not competing with optimized runtimes - focus is on simplicity and size
- AoS vs SoA - no great support in Rust, but could speed some things up. E.g stack is a SoA right now. Minimizes memory usage.
- A lot of cache misses due to the interpreter loop without unsafe code
- Accessing the store is slow, but it's a tradeoff for safety
- Small memory accesses are slow, but a lot of the issues disappear with the bulk memory proposal enabled
- Register based interpreters are a lot faster right now, pushing/popping from the stack is extremely expensive

## <u>**Conclusion**</u>

I was super happy with the results, and I'm still pushing the odd update here and there. Currently, TinyWasm supports WebAssembly V2 (without SIMD and threads) and several of other proposals. I also posted it on HN and Reddit, where I got some nice feedback and a few stars on GitHub (obviously the most important part).

{{ figure(caption = "Internet points are important.", position="center", src="./assets/hn.jpg", link="https://news.ycombinator.com/item?id=39627410") }}

If you're interested in checking it out or maybe even contributing, TinyWasm is up on [GitHub](https://github.com/explodingcamera/tinywasm) and also on [crates.io](https://crates.io/crates/tinywasm). Feel free to poke around, open issues, or even submit a PR (I recently improved the test suite and added a small contribution guide).

## <u>**Further Reading**</u>

After finishing TinyWasm, I started working on my thesis, which was a lot of fun and work. I'll probably write a post about that in the future. Still, for now, I can recommend the following resources if you're interested in WebAssembly/Interpreters/Compilers: [Crafting Interpreters](https://craftinginterpreters.com/) by Robert Nystrom is probably the best introduction to the entire field.
Going from there, I can also recommend the [Writing an Interpreter in Go](https://interpreterbook.com/)/[Writing a Compiler in Go](https://compilerbook.com/) books or the [Writing Interpreters in Rust Guide](httphttps://rust-hosted-langs.github.io/book/). But my biggest takeaway from this project was that you don't need to understand everything to start and can infer many basic principles from looking at other projects and just trying things.

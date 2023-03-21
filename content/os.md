+++
title = "os-dev in rust"
date = 2023-03-21
draft = true

[taxonomies]
tags = ["qoi", "koi", "image"]
+++

A couple of years ago, I started learning a bit about low-level dev things through [nand2tetris](https://www.nand2tetris.org/) but recently I've gotten more interested in kernel and OS development. Since in rustland, we move fast and break things, I thought it would be a good idea to document my journey through this process since many of the resources I've found are either outdated or not very accessible. I will be using the [risc-v](https://riscv.org/) architecture and qemu for emulation to start with, and hopefully I will be able to get a kernel to boot on a real board later on (my MangoPi MQ-Pro recently arrived and I'm excited to try it out).

{{toc}}

# Setting up the environment

# Getting the first kernel to boot

# Resources

To end of this page, I will have a list of resources that I have found useful in learning about OS development and a list of OSes that are written in rust and run on risc-v for reference.

## Guides

- [https://github.com/rcore-os/rCore-Tutorial-v3](https://github.com/rcore-os/rCore-Tutorial-v3)
- [https://osblog.stephenmarz.com/index.html](https://osblog.stephenmarz.com/index.html)
- [https://os.phil-opp.com/](https://os.phil-opp.com/)
- [https://wiki.osdev.org/Rust](https://wiki.osdev.org/Rust)
- [https://interrupt.memfault.com/blog/zero-to-main-rust-1](https://interrupt.memfault.com/blog/zero-to-main-rust-1)
- [https://docs.rust-embedded.org/embedonomicon/main.html](https://docs.rust-embedded.org/embedonomicon/main.html)

## Other risc-v rust OSes

- hobby OSes

  - [octox](https://github.com/o8vm/octox/tree/main)
  - [osmium](https://github.com/moratorium08/osmium) (no longer maintained)

- xv6-inspired OSes

  - [xv6-rust](https://github.com/Ko-oK-OS/xv6-rust) - xv6 ported to rust (learning OS)
  - [core-os-riscv](https://github.com/skyzh/core-os-riscv) - (no longer maintained)

- other OSes

  - [xous](https://github.com/betrusted-io/xous-core) - security focused OS
  - [r3](https://github.com/r3-os/r3) - realtime OS
  - [zCore](https://github.com/rcore-os/zCore) - zircon reimplementation in rust

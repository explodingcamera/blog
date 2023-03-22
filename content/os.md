+++
title = "os-dev in rust - part 1: getting started"
date = 2023-03-21
draft = true

[taxonomies]
tags = ["qoi", "koi", "image"]
+++

A couple of years ago, I started learning a bit about low-level dev things through [nand2tetris](https://www.nand2tetris.org/) but recently I've gotten more interested in kernel and OS development (mostly because I've been reading a lot of linux kernel code to understand how it works). I thought it would be a good idea to document my journey through this process since many of the resources I've found are either outdated or not very accessible. These posts will be kept as short as possible, and mostly serve as a reference for myself, but hopefully they will be useful to others as well. This first entry will be mainly about setting up the environment and getting the first kernel to boot. Most of the information here is taken from different guides and tutorials, and I will try to link to them as much as possible. The code for this project can be found [here](https://github.com/explodingcamera/pogos).

X86 is probably the most popular architecture for OS development, but comes with a lot of baggage and - in my opinion - is inevitably going to be replaced by something else in the future due to the high complexity of the architecture and historical baggage. [risc-v](https://riscv.org/) has been getting a lot of buzz on hackernews & co lately, so I decided to give it a try.
For now, I'll be using QEMU to emulate a CPU, and (hopefully) on a real board later on (my MangoPi MQ-Pro recently arrived and I'm excited to give it a try). All of the code will be written in rust with the goal of using as little unsafe code as possible.

{{toc}}

# Setting up the environment

Setting up low-level risc-v dev in rust is pretty straightforward, and can be done in a few steps:

TODO

# Booting on RISC-V

The whole bootloader landscape if pretty in flux for risc-v right now, however it is not too complicated if you follow the right steps. A decent overview can be found [here](https://riscv.org/wp-content/uploads/2019/12/Summit_bootflow.pdf). We'll be using the Supervisor Binary Interface, specifically [RustSBI](https://github.com/rustsbi/rustsbi) to boot our kernel, which is essentially a binary that is loaded by the bootloader and then jumps to the kernel entry point. Theres no real reason to use RustSBI over other SBI implementations, but I've decided to use it to stick to rust as much as possible. When running something like linux, you'll probably want to use something like u-boot on top, but for now we'll just jump straight to the kernel from RustSBI (We're using RustSBI as our _M-mode RUNTIME firmware_ together with [_FW_JUMP_](https://riscv.org/wp-content/uploads/2019/06/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf#page=16) to jump to the kernel which is super easy with qemu).

SBI is kind of like a kernel for our kernel, which gives our us a standard interface to interact with low-level hardware. This runs in M-mode, which is the highest privilege level on risc-v. The kernel will run in S-mode, which is the second highest privilege level. To interact with SBI, we will be using the `ecall` instruction, which is a trap instruction that will cause the CPU to jump to the SBI handler. The SBI handler will then handle the trap and call the appropriate function in RustSBI and it will then return to the kernel and continue execution. To call the SBI, we will be using the [sbi](https://crates.io/crates/sbi) crate, which provides a safe interface for ecalls. SBI can also be used to print to the qemu console, which is what we will be doing in this post.

# Linking and setting up our runtime

I'll quickly go over how you could setup the runtime for rust on risc-v, but we'll use the [riscv-rt](https://crates.io/crates/riscv-rt) crate to do all of this automatically for us in the end. The first thing we need to do is to setup our linker script. The linker is a program that takes a bunch of object files and combines them into a single binary and in our case, we need to tell it where to put the different sections of our binary so SBI can find them. The linker script for our kernel will look something like this:

```linker
ENTRY(_start)

SECTIONS
{
    . = 0x80000000;
    .text : {
        *(.text)
    }
    . = ALIGN(0x1000);
    .rodata : {
        *(.rodata)
    }
    . = ALIGN(0x1000);
    .data : {
        *(.data)
    }
    . = ALIGN(0x1000);
    .bss : {
        *(.bss)
    }
}
```

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

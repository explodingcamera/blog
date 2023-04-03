+++
title = "os-dev part 1: getting started"
slug = "osdev-1-getting-started"
date = 2023-03-21
wip = true

[taxonomies]
tags = ["os-dev", "rust"]
series = ["os-dev"]
+++

> This is a series of posts about my journey through OS development in rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos) and all of the posts in this series [here](/series/os-dev/).

## ! **This post is still a work in progress** !

A couple of years ago, I started learning a bit about low-level dev things through [nand2tetris](https://www.nand2tetris.org/) but recently I've gotten more interested in kernel and OS development (mostly because I've been reading a lot of linux kernel code to understand how it works). I thought it would be a good idea to document my journey through this process since many of the resources I've found are either outdated or not very accessible. These posts will be kept as short as possible, and mostly serve as a reference for myself, but hopefully they will be useful to others as well. This first entry will be mainly about setting up the environment and getting the first kernel to boot. Most of the information here is taken from different guides and tutorials, and I will try to link to them as much as possible. I recommend also reading Phil Oppermann's [blog](https://os.phil-opp.com/) which is a bit less opinionated and goes a bit more into detail.

X86 is probably the most popular architecture for OS development, but comes with a lot of baggage and - in my opinion - is inevitably going to be replaced by something else in the future due to the high complexity of the architecture. [risc-v](https://riscv.org/) has been getting a lot of buzz on hackernews & co lately, so I decided to give it a try.
For now, I'll be using QEMU to emulate a CPU, and (hopefully) on a real board later on (my MangoPi MQ-Pro recently arrived and I'm excited to give it a try). All of the code will be written in rust with the goal of using as little unsafe code as possible.

{{toc}}

# Setting up the environment

Setting up low-level risc-v dev in rust is pretty straightforward, and can be done in a few steps:

First of, I'll be using the nightly version of rust, so we can build parts of the standard library ourselves and use some unstable features which can be useful for OS development.

{{ file(name = "terminal")}}

```bash

# install rust nightly
rustup toolchain install nightly
# add the rust-src component (needed for building the standard library)
rustup component add rust-src --toolchain nightly
# add the risc-v target
# - gc stands for the generic (IMAFD extensions) and compressed extensions
# - elf is the executable format we'll be using
rustup target add riscv64gc-unknown-none-elf --toolchain nightly

cargo init --bin --name os
# simple & safe api for interacting with the supervisor binary interface (SBI)
cargo add sbi
```

Next, we'll create some config files to tell cargo how to build our project.

{{ file(name = "rust-toolchain.toml")}}

```toml

[toolchain]
channel = "nightly"
components = ["rust-src"]
```

{{ file(name = ".cargo/config.toml")}}

```toml

[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
# start our executable with qemu when running `cargo run`.
# This will be explained in more detail later
# `-bios ./bootloader/rustsbi-qemu.bin` can be omitted if you want to use OpenSBI instead
# which is bundled with qemu, but I prefer RustSBI
runner = "qemu-system-riscv64 -m 2G -machine virt -nographic -bios ./bootloader/rustsbi-qemu.bin -serial mon:stdio -kernel"

# linker flags with a custom linker script, we'll be using
# the one provided by `riscv-rt` later instead
rustflags = [
  "-Clink-arg=-T./src/linker.ld",
  "-Cforce-frame-pointers=yes",
]

[unstable]
# build the standard library ourselves
build-std = ["core", "alloc"]
```

If you want to use RustSBI too, you can download the binary [here](https://github.com/rustsbi/rustsbi-qemu/releases/tag/v0.1.1).

# Booting on RISC-V

The whole bootloader landscape if pretty in flux for risc-v right now, however it is not too complicated if you follow the right steps. A decent overview can be found [here](https://riscv.org/wp-content/uploads/2019/12/Summit_bootflow.pdf). We'll be using the Supervisor Binary Interface, specifically [RustSBI](https://github.com/rustsbi/rustsbi) to boot our kernel, which is essentially a binary that is loaded by the bootloader and then jumps to the kernel entry point. Theres no real reason to use RustSBI over other SBI implementations, but I've decided to use it to stick to rust as much as possible. When running something like linux, you'll probably want to use something like u-boot on top, but for now we'll just jump straight to the kernel from RustSBI (We're using RustSBI as our _M-mode RUNTIME firmware_ together with [_FW_JUMP_](https://riscv.org/wp-content/uploads/2019/06/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf#page=16) to jump to the kernel which is super easy with qemu).

SBI is kind of like a kernel for our kernel, which gives our us a standard interface to interact with low-level hardware. This runs in M-mode, which is the highest privilege level on risc-v. The kernel will run in S-mode, which is the second highest privilege level. To interact with SBI, we will be using the `ecall` instruction, which is a trap instruction that will cause the CPU to jump to the SBI handler. The SBI handler will then handle the trap and call the appropriate function in RustSBI and it will then return to the kernel and continue execution. To call the SBI, we will be using the [sbi](https://crates.io/crates/sbi) crate, which provides a safe interface for ecalls. SBI can also be used to print to the qemu console, which is what we will be doing in this post.

# Linking and setting up our runtime (optional)

I'll quickly go over how you could setup the runtime for rust on risc-v, but we'll use the [riscv-rt](https://crates.io/crates/riscv-rt) crate to do all of this automatically for us in the end, so you can skip this section if you want.

The first thing we need to do is to setup our linker script. The linker is a program that takes a bunch of object files and combines them into a single binary and in our case, we need to tell it where to put the different sections of our binary so SBI can find them. The linker script for our kernel will look something like this:

{{ file(name = "src/linker.ld")}}

```ld

OUTPUT_ARCH(riscv) #
ENTRY(_start)

/* The base address of the program. RAM starts at 0x80000000 in QEMU
   and SBI takes the first 2MB. */
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        . = ALIGN(4K);
        strampoline = .;
        *(.text.trampoline);
        . = ALIGN(4K);
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    sbss_with_stack = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

Since we're using the ELF format for our kernel, we'll be distinguishing different regions of memory TEXT, DATA and BSS.

- **TEXT**: This is where our code will be stored. This is where the CPU will execute instructions from.
- **DATA**: This is where global variables will be stored
- **BSS**: This is where we will setup our stack

Now, we need to tell cargo to use this linker script. We can do this by adding the following to our `.cargo/config.toml` file:

{{ file(name = ".cargo/config.toml")}}

```toml

[target.riscv64gc-unknown-none-elf]
# ...
rustflags = [
  "-Clink-arg=-T./os/src/linker.ld",
  "-Cforce-frame-pointers=yes",
]
```

And now, we'll write a small assembly file that will be used as our entry point. This will
setup the stack and jump to our rust entry point. We'll call this `entry.asm` and it will look like this:

{{ file(name = "src/entry.asm")}}

```riscv

    .section .text.entry
    .globl _start
_start:
    # set stack pointer to the top of the stack
    la sp, boot_stack_top
    # call the start function in the kernel (defined in ./main.rs)
    call __main

    # set the memory for the stack
    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

# Using RISCV-RT

RISCV-RT ia a crate that will automatically setup our runtime for us so we don't have to do any of the stuff we did in the previous section.
Additionally, it also provides us with a trap handler which will be very useful for handling interrupts and exceptions later on.
We can add it to our project by adding the following to our `Cargo.toml` file:

{{ file(name = "cargo.toml")}}

```toml

[dependencies]
# we'll be running our kernel in S-mode which is needed for SBI
riscv-rt = {version = "0.11", features = ["s-mode"]}
```

# Resources

To end this post, I have a list of resources that I have found useful in learning about OS development and a list of OSes that are written in rust and run on risc-v for reference.

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

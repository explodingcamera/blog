+++
title = "os-dev part 1: getting started"
slug = "osdev-1"
date = 2023-05-07
draft = true

[taxonomies]
tags = ["os-dev", "rust"]
series = ["os-dev"]
+++

> This is a series of posts about my journey creating a kernel in rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos) and all of the posts in this series [here](/series/os-dev/).

A couple of years ago, I started learning a bit about low-level development through [nand2tetris](https://www.nand2tetris.org/) but recently I've gotten more interested in kernel and OS development through reading a lot ow lwn.net and because of the inclusion of rust support for writing linux kernel modules.
I've found that a lot of blogs and guides on kernels are either outdated or not very accessible, so this will be a different (and hopefully more fun) approach, fully utilizing the rust ecosystem to get up and running quickly. To dive in deeper, I also reading Phil Oppermann's amazing [blog](https://os.phil-opp.com/) on the subject. To get the most out of this series, prerequisite knowledge of rust or a sililar programming language is required. All commands throughout the series will also be expecting a linux terminal and might need to be slightly adjusted for osX or Windows.

{{toc}}

# RISC-V

X86 is currently the dominant CPU architecture, and only has recently lot a bit of marketshare to emerging ARM CPUs like the Apple M1 or AWS Graviton. For this series, however, we'll be writing the kernel for RISC-V. RISC-V is, as the name tells us, a Reduced Instruction Set Computer, which was released in 2015 under royalty-free open-source licenses. By focusing on small and modular instruction extensions and being standardazed so recently, it avoids the large historycal baggage and weird design decisions plaguing x86 (check out [this](https://mjg59.dreamwidth.org/66109.html) article to see the horrendous boot process in action).

I'll be using QEMU to emulate a CPU, and (hopefully) on a real board later on (my MangoPi MQ-Pro recently arrived and I'm excited to give it a try).

# Setting up the environment

Recent versions of rust have (mostly) made setting up bare metal development a breeze. First of, I'll be using the nightly version of rust, so we can build parts of the standard library ourselves and use some unstable features.

{{ file(name = "terminal")}}

```bash
# install qemu, this will be different depending on your os
sudo pacman -S qemu-system-riscv64

# install rust nightly
rustup toolchain install nightly
# add the rust-src component (needed for building the standard library)
rustup component add rust-src --toolchain nightly
# add the risc-v target
# - gc stands for the generic (IMAFD extensions) and compressed extensions
# - elf is the executable format we'll be using
rustup target add riscv64gc-unknown-none-elf --toolchain nightly

cargo init --bin --name os
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
# (qemu bundles opensbi, so we don't need to delare a bootloader)
runner = "qemu-system-riscv64 -m 2G -machine virt -nographic -serial mon:stdio -kernel"

# linker flags
rustflags = [
  "-Clink-arg=-Tmemory.x",
  "-Clink-arg=-Tlink.x",
]

# build the standard library ourselves, required to use the alloc crate
[unstable]
build-std = ["core", "alloc"]
```

# Booting on RISC-V

The whole bootloader landscape if pretty in flux for risc-v right now, however it is not too complicated if you follow the right steps. A decent overview can be found [here](https://riscv.org/wp-content/uploads/2019/12/Summit_bootflow.pdf). We'll be using the Supervisor Binary Interface as out bootloader. We're using OpenSBI (Supervisor Binary Interface) as our _M-mode RUNTIME firmware_. The version shipping with qemu uses a Jump Address ([_FW_JUMP_](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_jump.md)), in this case `0x80200000`, which is where we'll be putting our kernel.

SBI runs in Machine-mode, which is the highest [privilege level](http://docs.keystone-enclave.org/en/latest/Getting-Started/How-Keystone-Works/RISC-V-Background.html#risc-v-privilieged-isa) on risc-v. The kernel will run in Supervisor-mode, and user programs will run in User-mode, which is the lowest privilege level.

This architecture has a lot of benefits: SBI puts an abstraction layer between the kernel and the hardware, which allows us to write a single kernel that can run on any risc-v CPU, regardless of the extensions it supports, as long as it has a SBI implementation. SBI also provides a lot of useful functions like printing to and reading from the console, and it loads a device tree blob into memory, which we'll also be using later on to get information about the hardware.

To interact with SBI, we will be using the `ecall` instruction, which is a trap instruction that will cause the CPU to jump to the SBI handler. The SBI handler will then handle the trap and call the appropriate function, and return to the kernel to continue execution. The SBI specification can be found [here](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/riscv-sbi.adoc).

# Setting up the runtime

To make a binary that is able to be loaded as a kernel, we'll be using the [riscv-rt](https://crates.io/crates/riscv-rt) crate which provides a runtime for risc-v. It also provides us with a trap handler which will be very useful for handling interrupts and exceptions. Later on, we'll be writing our own runtime, but for now, we'll use this to get up and running quickly. We can add it to our project by adding the following to our `Cargo.toml` file:

{{ file(name = "cargo.toml")}}

```toml

[dependencies]
# we'll be running our kernel in S-mode which is needed for SBI
riscv-rt = {version = "0.11", features = ["s-mode"]}
```

No we need to configure out linker. A linker is a program that takes a bunch of object files and combines them into a single binary and in our case, we need to tell it where to put the different sections of our binary so SBI can find them. `riscv-rt` already ships with a linker script, where we only need to change the memory addresses. We'll be using the following linker script to set up an appropriate layout for qmeu:

{{ file(name = "link.x")}}

```ld
MEMORY
{
  RAM : ORIGIN = 0x80200000, LENGTH = 16M
  /* 16MB ought to be enough for anyone */
}

REGION_ALIAS("REGION_TEXT", RAM);
REGION_ALIAS("REGION_RODATA", RAM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
REGION_ALIAS("REGION_STACK", RAM);
/* we're skipping the heap as it's a bit easier to implement it using a rust array */
```

Since we're using the ELF format for our kernel, we'll be distinguishing different regions of memory TEXT, DATA and BSS.

- **TEXT**: This is where our code will be stored. This is where the CPU will execute instructions from.
- **DATA**: This is where global variables will be stored
- **BSS**: This is where uninitialized global variables will be stored
- **STACK**: The stack is used to store local variables and function arguments. It grows downwards, so it starts at the top of the memory region and grows towards the bottom (so it starts at `STACK_END` and grows towards `STACK_START`)

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

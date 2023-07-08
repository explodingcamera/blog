+++
title = "Operating Systems in Rust #1: Hello RISC-V"
description = "In this first post, we'll set up our environment and write a simple program that prints 'Hello World' to the screen."
date = 2023-05-07
updated = 2023-05-12
aliases = ["osdev-1"]
transparent = true

[taxonomies]
tags = ["rust", "riscv", "kernel"]
series = ["rust-os"]
+++

{% quote (class="info")%}

This is a series of posts about my journey creating a kernel in rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos/tree/part-1) and all of the posts in this series [here](/series/rust-os/).

{% end %}

# Background

I've been interested in operating systems for a while now and, with many of the recent advancements in rust's role in the OS ecosystem, I thought it would be fun to try and write a kernel in rust.
I've found that many blogs and guides on writing kernels and operating systems are either pretty outdated or not very accessible, so this will be a different (and hopefully more fun) approach, fully utilizing the Rust ecosystem to get up and running quickly and minimizing the use of unsafe and assembly code.

This series requires pre-requisite knowledge of Rust or a similar programming language. All commands throughout the series will also be expecting a Linux terminal and might need to be adjusted slightly for macOS or Windows. I'll be using Arch Linux, but any distro should work fine. I'm trying to keep everything approachable, so if you have any questions or suggestions, feel free to reach out to me at [mail@henrygressmann.de](mailto:mail@henrygressmann.de)

To follow along, I've also created a [GitHub Repo](https://github.com/explodingcamera/pogos) with a branch for each part of the series. You can find the code for this part [here](https://github.com/explodingcamera/pogos/tree/part-1).

{{toc}}

# CPU Architectures and RISC-V

X86 is currently the dominant CPU architecture and has recently lost a bit of market share to emerging ARM CPUs like the [Apple M Series](https://en.wikipedia.org/wiki/Apple_M2) or [AWS Graviton](https://en.wikipedia.org/wiki/AWS_Graviton). For this series, however, we'll be targeting RISC-V. RISC-V is a CPU Architecture released in 2015 under royalty-free open-source licenses. By focusing on small and modular instruction extensions and being standardized so recently, it avoids the sizeable historical baggage and weird design decisions plaguing x86 (check out [this](https://mjg59.dreamwidth.org/66109.html) article to see the horrendous boot process in action). RISC-V was also designed to be extensible, allowing for custom instruction extensions to be added to the base ISA. This enables us to use the same kernel on a wide range of CPUs, from small embedded devices to high-performance servers.

I'll be using [QEMU](https://www.qemu.org/) to run our kernel. QEMU is a virtual machine that can emulate a wide range of CPUs and devices, including RISC-V. At the end of this series, we'll also run it on an actual board (my [MangoPi MQ-Pro](https://mangopi.org/mangopi_mqpro) recently arrived, and I'm excited to try it).

# Setting up the environment

Recent versions of Rust have (mostly) made setting up bare metal development a breeze. I'll be using Rust nightly, so we can build parts of the standard library ourselves using `rust-src` and use some unstable features that will be useful for OS development.

{% quote (class="info")%}

Because the standard library depends on an operating system to provide memory allocation, threading, and other things, we need to later mark our crate as `#![no_std]` and implement these ourselves.

{% end %}

To do this, we'll use the `core` library, a subset of the standard library that doesn't depend on an operating system. With it, we won't have access to things like `println!` or `Vec`, but we can still use types like `Option` and `Result` and many other useful APIs.
In the next post, we'll also use the `alloc` create to enable us to use language features that require heap allocations, such as `Vec` and `Box`.

Before we start, we'll need to install a couple of things:

{{ file(name = "terminal")}}

```bash

# install qemu, this will be different depending on your os
sudo pacman -S qemu qemu-system-riscv

# install Rust nightly
rustup toolchain install nightly

# add the rust-src component (needed for building the alloc crate and other things)
rustup component add rust-src --toolchain nightly

# add the RISC-V target
# - GC stands for the generic (IMAFD extensions) and compressed extensions
#   These are the most common extensions which are required for most applications
# - ELF is the executable format we'll be using
#   This is the format Linux and most other UNIX-like operating systems use for executables.
#   Other common formats are PE (windows) and Mach-O (macOS).
#   (On x86, we'd have to endure the pain of dealing with PE binaries)
rustup target add riscv64gc-unknown-none-elf --toolchain nightly

# create a new cargo project (this will be our kernel)
cargo init --bin --name kernel
```

Next, we'll create config files to tell cargo what version of Rust to use and what target to build for.

{{ file(name = "rust-toolchain.toml")}}

```toml

[toolchain]
channel = "nightly" # use the nightly version of Rust
components = ["rust-src"] # we need this to build the alloc crate
```

{{ file(name = ".cargo/config.toml")}}

```toml

[build]
target = "riscv64gc-unknown-none-elf" # build an ELF executable, more on this later

[target.riscv64gc-unknown-none-elf]
# start our executable with qemu when running `cargo run`.
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

## RISC-V Privilege Levels

To better understand how we'll be booting our kernel, we'll first have to understand how RISC-V's privilege levels work. RISC-V has three privilege levels, sometimes called _rings_ or _modes_.

Firmware runs in Machine mode, the highest [privilege level](http://docs.keystone-enclave.org/en/latest/Getting-Started/How-Keystone-Works/RISC-V-Background.html#RISC-V-privilieged-isa) on RISC-V. The kernel will run in Supervisor-mode, and user programs will run in User-mode, the lowest privilege level.

{{ figure(caption = "The three privilege levels of RISC-V", position="center", src="./assets/rings.svg") }}

## SBI and OpenSBI

Compared to other CPU Architectures, RISC-V's boot process is straightforward.
We're using OpenSBI as our Supervisor Execution Environment (SEE), our _M-mode RUNTIME firmware_.

{% quote (class="info")%}
SBI (Supervisor Binary Interface) is a standard interface for interacting with the SEE, and OpenSBI is an implementation of this standard.
{% end %}

The version shipping with QEMU uses a Jump Address ([_FW_JUMP_](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_jump.md)), in this case, `0x80200000`, which is where we'll be putting our kernel. QEMU will load our kernel into memory and jump to `0x80000000`, from where OpenSBI will then jump to `0x80200000`, where our kernel is located.

{{ figure(caption = "Traditional Boot Flow", position="center", src="./assets/boot1.svg") }}

{{ figure(caption = "QEMU RISC-V Boot Flow", position="center", src="./assets/boot.svg") }}

This architecture has a lot of benefits: SBI puts an abstraction layer between the kernel and the hardware, which allows us to write a single kernel that can run on any RISC-V CPU, regardless of the extensions it supports, as long as it has an SBI implementation. SBI also provides many functions like printing to and reading from the console, and it loads a flattened device tree (FDT) into memory, which we'll also be using later on to get information about the hardware.

To interact with SBI, we will use the `ecall` instruction, a trap instruction that will cause the CPU to jump to the Supervisor Execution Environment. The SEE handler will then handle the trap and call the appropriate function, and return to the kernel to continue execution. The SBI specification can be found [here](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/riscv-sbi.adoc).

{{ figure(caption = "Calling SBI", position="center", src="./assets/sbi.svg") }}

# Setting up the runtime

To make a binary that can be loaded as a kernel, we'll use the [riscv-rt](https://crates.io/crates/riscv-rt) crate, which provides a runtime for RISC-V. It also provides us with a trap handler which will be very useful for handling interrupts and exceptions and a linker script which we'll be using to set up the memory layout of our kernel.

Later on, we'll be writing our own runtime, but for now, we'll use this to get up and running quickly. We can add it to our project by adding the following to our `Cargo.toml` file:

{{ file(name = "cargo.toml")}}

```toml

[package]
edition = "2021"
name = "kernel"
version = "0.1.0"

[dependencies]
# enable the s-mode feature to use the supervisor mode runtime (opensbi will run in machine mode and load our kernel into supervisor mode)
riscv-rt = {version = "0.11", features = ["s-mode"]}
sbi = "0.2" # provides a wrapper around the SBI functions to make them easier to use
```

Now we need to configure our linker. A linker is a program that takes a bunch of object files and combines them into a single binary, and - in our case - we need to tell it where to put the different sections of our binary so SBI can find them. `riscv-rt` already ships with a linker script. We will only need to tell it about the memory layout of our kernel.

Since we're using the Executable and Linking Format (ELF) for our kernel, we'll be distinguishing different regions of memory TEXT, DATA, and BSS.

{{ figure(caption = "ELF Memory Layout", position="center", src="./assets/elf.svg") }}

To make sure that our kernel is loaded at the correct address, we need to add the following to our `memory.x` file:

{{ file(name = "memory.x")}}

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
REGION_ALIAS("REGION_HEAP", RAM);
REGION_ALIAS("REGION_STACK", RAM);
```

`link.x` will be provided by `riscv-rt`, and we don't need to change it for now.
To make sure that the linker can find our script, we need to add the following to our `build.rs` file:

{{ file(name = "build.rs")}}

```rust

use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

    // Put the linker script somewhere the linker can find it.
    fs::write(out_dir.join("memory.x"), include_bytes!("memory.x")).unwrap();
    println!("cargo:rustc-link-search={}", out_dir.display());
    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rerun-if-changed=build.rs");
}
```

# Hello world

Now that we have the linker configured, we can start writing our kernel. We'll start by writing a simple hello world program that prints "Hello world!" to the console and then shuts down the machine.

## Printing

Since we're in a `no_std` environment, we can't use the standard library and must implement a print function ourselves. We'll be using the `sbi` crate to interact with our Supervisor Execution Environment (SEE) OpenSBI, which provides a `console_putchar` function that we can use to print a single character to the console. We can use this to implement a simple print function that prints a string to the console. This print function iterates over the characters in the string and prints them to the QEMU debug console.

{{ file(name = "src/utils.rs")}}

```rust

pub fn print(t: &str) {
    t.chars().for_each(
        // FUTURE: replace with the new SBI debug extension once it's available in all SBI implementations
        |c| sbi::legacy::console_putchar(c.try_into().unwrap_or(b'?')),
    );
}
```

To get super fancy with our print function, we can also implement a macro that allows us to print to the console using the same syntax as the standard library's `println!` macro. All we need to do for this is implement the `core::fmt::Write` for our print function.

{{ file(name = "src/utils.rs")}}

```rust

struct Writer {}
pub fn print_args(t: core::fmt::Arguments) {
    use core::fmt::Write;
    let mut writer = Writer {};
    writer.write_fmt(t).unwrap();
}

impl core::fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        print(s);
        Ok(())
    }
}

#[macro_export]
macro_rules! print {
    ($fmt:literal$(, $($arg: tt)+)?) => {
        $crate::utils::print_args(format_args!($fmt $(,$($arg)+)?))
    }
}

#[macro_export]
macro_rules! println {
    ($fmt:literal$(, $($arg: tt)+)?) => {{
        $crate::print!($fmt $(,$($arg)+)?);
        $crate::utils::print("\n");
    }};
    () => {
        $crate::utils::print("\n");
    }
}
```

While we're at it, let's also add a quick method to shut down the system:

{{ file(name = "src/utils.rs")}}

```rust

pub fn shutdown() -> ! {
    let _ = sbi::system_reset::system_reset(
        sbi::system_reset::ResetType::Shutdown,
        sbi::system_reset::ResetReason::NoReason,
    );
    unreachable!("System reset failed");
}
```

## Panic handler

Notice the `.unwrap()` call in the `print_args` function? Since we're using `no_std`, we can't use the standard library's panic handler. Instead, we'll write our panic handler to print the panic message to the console and then halt the CPU. We can do this by implementing the `panic_handler` lang item. This is a special function that is called when a panic occurs. We can implement it like this:

{{ file(name = "src/panic_handler.rs")}}

```rust

use crate::println;
use core::{hint::unreachable_unchecked, panic::PanicInfo};
use sbi::system_reset::{ResetReason, ResetType};

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{info}");

    let _ = sbi::system_reset::system_reset(
      ResetType::Shutdown,
      ResetReason::SystemFailure
    );

    unsafe {
        println!("System reset failed");

        // this can pretty much only happen if there is a bug in the sbi implementation
        // or if sbi is not present, unreachable_unchecked so we don't panic again
        unreachable_unchecked();
    }
}
```

## Entry point

Now we can finally write our hello world program. We'll be using the `entry` macro from `riscv-rt` to mark our main function as the entry point of our program. `riscv-rt` will load some assembly to set up a basic c-runtime environment and then call our main function. The main function will be passed the hart id of the hart that is executing it, which is passed to us by OpenSBI through the `a0` register.

{% quote (class="info")%}

Hart is the RISC-V term for a CPU core. A RISC-V system can have multiple harts, each with its own register state and program counter.

{% end %}

Something you might not have seen before is the `!` return type. This is a special type that means that the function never returns. Since we're writing a kernel, there's nowhere for our program to return to, so we'll need to either loop forever or shut down the machine.

{{ file(name = "src/main.rs")}}

```rust

#![no_std]
#![no_main]
#![feature(panic_info_message)]
#![feature(lazy_cell)]
#![allow(unused)]

extern crate riscv_rt;

use riscv_rt::entry;
mod panic_handler;
mod utils;

#[entry]
fn main(a0: usize) -> ! {
    println!("Hello world from hart {}\n", a0);

    utils::shutdown();
}
```

# Review

Once we run `cargo run`, we should see "Hello world!" printed on the console:

{{ file(name = "terminal")}}

```

$ cargo run

OpenSBI v1.2
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Hello world from hart 0
```

In the next few posts, we'll start handling interrupts and exceptions, allocate data on the heap, set up a page table, and much more, so stay tuned!

To dive in deeper, I also recommend reading Phil Oppermann's fantastic [blog](https://os.phil-opp.com/), where he creates a kernel in Rust for the x86 architecture, and Stephen Marz's [blog](https://osblog.stephenmarz.com/) about RISC-V and Rust.

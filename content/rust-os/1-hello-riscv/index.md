+++
title = "Operating Systems in Rust #1: Hello RISC-V"
description = "In this first post, we'll set up our environment and write a simple program that prints 'Hello World' to the screen."
date = 2023-05-07
updated = 2024-03-14
aliases = ["osdev-1"]
transparent = true

[taxonomies]
tags = ["rust", "riscv", "kernel"]
series = ["rust-os"]
+++

{% quote (class="info")%}

This is a series of posts about my journey creating a kernel in Rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos/tree/part-1) and all of the posts in this series [here](/series/rust-os/).

{% end %}

# Background

I've been interested in operating systems for a while now, and with many of the recent advancements in Rust's role in the OS ecosystem, I thought it would be fun to try and write a kernel in Rust.
I've found that many blogs and guides on writing kernels and operating systems are either pretty outdated or not very accessible, so this will be a different (and hopefully more fun) approach, fully utilizing the Rust ecosystem to get up and running quickly and minimizing the use of unsafe and assembly code.

This series requires a basic understanding of Rust or similar languages. You'll see some commands for a Linux terminal—Arch Linux in my case—but don't worry if you're on macOS, Windows, or using another Linux distro; a little tweaking should get things running smoothly.

To follow along, I've also created a [GitHub Repo](https://github.com/explodingcamera/pogos) with a branch for each part of the series. You can find the code for this part [here](https://github.com/explodingcamera/pogos/tree/part-1).

{{toc}}
<br/>

## RISC-V and other CPU Architectures

X86 is currently the dominant CPU architecture and has recently lost a bit of market share to emerging ARM CPUs like the [Apple M Series](https://en.wikipedia.org/wiki/Apple_M2) or [AWS Graviton](https://en.wikipedia.org/wiki/AWS_Graviton). For this series, however, we'll be targeting RISC-V. RISC-V is a CPU Architecture released in 2015 under royalty-free open-source licenses. By focusing on small and modular instruction extensions and being so new, it avoids the sizeable historical baggage and weird design decisions plaguing x86 (check out [this](https://mjg59.dreamwidth.org/66109.html) article to see the horrendous boot process in action). RISC-V was also designed to be extensible, allowing custom instruction extensions to be added to the base ISA. This enables us to use the same kernel on a wide range of CPUs, from small embedded devices to high-performance servers.

I'll use [QEMU](https://www.qemu.org/) to run our kernel. QEMU is a virtual machine that can emulate various CPUs and devices, including RISC-V. At the end of this series, we'll also run it on an actual board (my [MangoPi MQ-Pro](https://mangopi.org/mangopi_mqpro) recently arrived, and I'm excited to try it).

# Setting up the environment

Recent versions of Rust have made all the setup around building bare metal applications a lot easier than it used to be. We can now do most of our development with stable Rust and only need to use nightly for a few features later.

Before we start writing code, we'll need to install some things, including Rust and QEMU. The following commands will install Rust and QEMU on Arch Linux, but you can find instructions for other operating systems [here](https://www.rust-lang.org/tools/install) and [here](https://www.qemu.org/download/).

{{ file(name = "terminal")}}

```bash

# Install rust if you haven't already
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install qemu and the riscv64 toolchain
sudo pacman -S qemu qemu-system-riscv

# Create a new cargo project (this will be our kernel)
cargo init --bin --name kernel

# To help us get rid of some boilerplate, we'll add `riscv-rt` to our project.
# This crate provides a small runtime, including a linker script and a trap handler.
# We also need to enable s-mode to use the supervisor mode runtime, more on this later
cargo add riscv-rt --features s-mode
```

Next, we need to create config files to tell cargo more about our project. We'll start by creating a `rust-toolchain.toml` file to specify the version of Rust we want to use, and a `.cargo/config.toml` file to specify the target we want to build for and some linker flags.

{{ file(name = "rust-toolchain.toml")}}

```toml

[toolchain]
channel="stable" # We'll just use the most recent stable version of Rust for now
targets=["riscv64gc-unknown-none-elf"] # Build a riscv ELF executable, more on this later
```

{{ file(name = ".cargo/config.toml")}}

```toml

[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
# Pass our executable to qemu when running `cargo run`.
runner = "qemu-system-riscv64 -m 2G -machine virt -nographic -serial mon:stdio -kernel"

# Linker flags
rustflags = [
  "-Clink-arg=-Tmemory.x",
  "-Clink-arg=-Tlink.x",
]
```

The executable format we'll use is ELF, as you can see from the `riscv64gc-unknown-none-elf` target we specified above. ELF is the format used by Linux and most other UNIX-like operating systems for storing executables. Other common formats are PE (windows) and Mach-O (macOS). On x86, we'd have to endure the pain of dealing with PE binaries.

{{ figuresvg(caption = "ELF Memory Layout", position="center", src="content/rust-os/1-hello-riscv/assets/elf.svg") }}

One thing you might also have noticed is the `-Tmemory.x` and `-Tlink.x`, our linker scripts. For typical applications, these configuration files are generated by the compiler automatically, but for bare metal applications like ours, we need to specify how our program should be laid out in memory.

`riscv-rt` already ships with a basic linker script, so we will only need to tell it some basic information about the memory layout of the device we want to run it on. In this case, we'll put the entire kernel in RAM and give it a size of 16MB. The address `0x80200000` is the start of the RAM in QEMU's virtual machine, and `16M` is the amount of RAM we want to use.

{{ file(name = "memory.x")}}

```ld

MEMORY
{
  RAM : ORIGIN = 0x80200000, LENGTH = 16M
}

REGION_ALIAS("REGION_TEXT", RAM);
REGION_ALIAS("REGION_RODATA", RAM);
REGION_ALIAS("REGION_DATA", RAM);
REGION_ALIAS("REGION_BSS", RAM);
REGION_ALIAS("REGION_HEAP", RAM);
REGION_ALIAS("REGION_STACK", RAM);
```

With this file in place, we need to tell the linker where to find it. We can do this by adding a `build.rs` file to our project:

{{ file(name = "build.rs")}}

```rust

use std::env;
use std::fs;
use std::path::PathBuf;

fn main() {
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());

    fs::write(out_dir.join("memory.x"), include_bytes!("memory.x")).unwrap();
    println!("cargo:rustc-link-search={}", out_dir.display());
    println!("cargo:rerun-if-changed=memory.x");
    println!("cargo:rerun-if-changed=build.rs");
}
```

We can now write our first program to ensure everything works as expected. For now, we'll write a simple program that loops forever. We'll also need to add a [custom panic handler](https://doc.rust-lang.org/nomicon/panic-handler.html) to get our program to compile.

{{ file(name = "src/main.rs")}}

```rust

#![no_std]
#![no_main]

use riscv_rt::entry;
mod utils;

// We need to specify a panic handler for no_std programs to compile,
// for now this is just a placeholder
#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! { loop {} }

#[entry]
fn main() -> ! {
    loop {} // Busy loop forever
}
```

Now we can build and run our program:

{{ file(name = "terminal")}}

```bash

cargo run
```

And there we go! We've got a working environment set up!
To stop the program, press `Ctrl + A` followed by `X`.

<br />

# Booting on RISC-V

## RISC-V Privilege Levels

To better understand how this works behind the scenes, we first have to know a bit more about RISC-V's privilege levels. RISC-V has three privilege levels, sometimes called _rings_ or _modes_.

Firmware runs in Machine mode, the highest [privilege level](http://docs.keystone-enclave.org/en/latest/Getting-Started/How-Keystone-Works/RISC-V-Background.html#RISC-V-privilieged-isa).
This is where the bootloader and the Supervisor Execution Environment (SEE) run. This SEE is a piece of software that provides a small abstraction layer between the kernel and the hardware, loads the kernel into memory, and jumps to it. Our kernel will run in Supervisor-mode, which is the second-highest privilege level. Finally, applications will run in User-mode, the lowest privilege level.

{{ figuresvg(caption = "The three privilege levels of RISC-V", position="center", src="/content/rust-os/1-hello-riscv/assets/rings.svg") }}

## SBI and OpenSBI

Compared to other CPU Architectures, RISC-V's boot process is relatively straightforward (If you don't try to force UEFI onto it).
We use OpenSBI as our Supervisor Execution Environment (SEE), our _M-mode RUNTIME firmware_.

The version shipping with QEMU uses a Jump Address ([_FW_JUMP_](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_jump.md)), in this case, `0x80200000`.
This is a location in memory where we'll put our kernel, using the `-kernel` flag we set in our `.cargo/config.toml` file earlier. From there, OpenSBI will run some initialization code and jump to our kernel.

{{ figure(caption = "Traditional Boot Flow", position="center", src="./assets/boot1.svg") }}

{{ figure(caption = "QEMU RISC-V Boot Flow", position="center", src="./assets/boot.svg") }}

This architecture has a lot of benefits: SBI puts an abstraction layer between the kernel and the hardware, which allows us to write a single kernel that can run on any RISC-V CPU, regardless of the extensions it supports, as long as it has an SBI implementation. SBI also provides many functions, such as printing and reading from the console. It also loads a flattened device tree (FDT) into memory, which we'll use later to get information about the hardware.

{{ figure(caption = "Calling SBI", position="center", src="./assets/sbi.svg") }}

To interact with SBI, we will use the `ecall` instruction, a trap instruction that will cause the CPU to jump to the Supervisor Execution Environment. The SEE handler will then handle the trap, call the appropriate function, and return to the kernel to continue execution. The SBI specification can be found [here](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/riscv-sbi.adoc).

<br />

# Hello world

Let's write some actual code! First, let's check off a simple hello world, print "Hello world!" to the console, and then shut down the machine.

## Printing

Since we're in a `no_std` environment, we can't use the standard library and must implement a print function ourselves. We'll be using the `sbi` crate to interact with our Supervisor Execution Environment (SEE) OpenSBI, which provides a `console_putchar` function that we can use to print a single character to the console. Using this crate, we can create a simple print function that prints a string to the console. This print function iterates over the characters in the string and prints them to the QEMU debug console.

{% quote (class="info")%}
`console_putchar` is now part of the SBI Debug Extension, which is not yet available in all SBI implementations. We'll use the legacy console putchar function for now, but we'll switch to the debug extension once it's more widely available.
{% end %}

{{ file(name = "src/utils.rs")}}

```rust

pub fn print(t: &str) {
    for c in t.chars() {
        sbi::legacy::console_putchar(c.try_into().unwrap_or(b'?'))
    }
}
```

To get super fancy with our print function, we can also implement a macro that allows us to print to the console using the same syntax as the standard library's `println!` macro. All we need to do for this is implement the `core::fmt::Write` trait for our print function.

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

Notice the `.unwrap()` call in the `print_args` function? With our current panic handler, this will cause our program to halt the CPU and tell us nothing about what went wrong. Instead, we'll change our panic handler to print the panic message to the console and then shut down the system (more on this can be found in the [rust nomicon](https://doc.rust-lang.org/nomicon/panic-handler.html)).

{{ file(name = "src/panic_handler.rs")}}

```rust

use crate::println;
use core::{hint::unreachable_unchecked, panic::PanicInfo};
use sbi::system_reset::{ResetReason, ResetType};

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("A panic occurred: {info}");

    let _ = sbi::system_reset::system_reset(
        ResetType::Shutdown,
        ResetReason::SystemFailure
    );

    println!("System reset failed");
    // We need to loop forever to satisfy the `!` return type,
    // since `!` effectively means "this function never returns".
    loop {}
}
```

## Entry point

Now, let's put it all together. We'll import our newly created `utils` module and add a new argument to our `main` function. This argument is passed to us by OpenSBI using the `a0` register and contains the hart id of the hart that is executing our program.

{% quote (class="info")%}
Hart is the RISC-V term for a CPU core. A RISC-V system can have multiple harts, each with a register state and program counter.
{% end %}

Another thing of note is the `#[entry]` macro we used to define the entry point of our program. This macro is provided by the `riscv-rt` crate and is used to define the entry point of our program, which is called after the runtime has set up the stack and other things for us.

This is done using some inline assembly in there, which is pretty well documented in their [source code](https://github.com/rust-embedded/riscv/blob/master/riscv-rt/src/asm.rs). It's a great resource to check out if you're interested in how it works or want to write your own runtime or a custom linker script as your kernel grows.

{{ file(name = "src/main.rs")}}

```rust

#![no_std]
#![no_main]

extern crate riscv_rt;

use riscv_rt::entry;
mod panic_handler;
mod utils;

#[entry]
fn main(a0: usize) -> ! {
    println!("Hello world from hart {}!", a0);
    utils::shutdown();
}
```

<br />

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

Hello world from hart 0!
```

In the next few posts, we'll start handling interrupts and exceptions, allocate data on the heap, set up a page table, and much more!

To dive in deeper, I also recommend reading Phil Oppermann's fantastic [blog](https://os.phil-opp.com/), where he creates a kernel in Rust for the x86 architecture, and Stephen Marz's [blog](https://osblog.stephenmarz.com/) about RISC-V and Rust.

{% quote (class="info")%}

The next post in this series is available here: [Operating Systems in Rust #2: Shell](/rust-os/2-shell/).

{% end %}

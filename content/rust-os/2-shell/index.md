+++
transparent = true
title = "Operating Systems in Rust #2: Shell"
description = "Continuing with the kernel we started in the previous post, we'll add a simple shell and a global allocator to use heap allocated data structures."
date = 2023-07-08
updated = 2024-03-01 

[taxonomies]
tags = ["rust", "riscv", "kernel"]
series = ["rust-os"]
+++

{% quote (class="info")%}

This is a series of posts about my journey creating a kernel in Rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos/tree/part-2) and all of the posts in this series [here](/series/rust-os/).

{% end %}

Now that we have a basic kernel that can print to the screen, we can start building out some more functionality.
I first want to create a simple shell that will allow us to run some commands and more easily interact with our system.

As I mentioned in the previous post, we can't yet use heap-allocated data structures, so we'll start with implementing a Global Allocator. This will allow us to use APIs like `Box` and `Vec` anywhere in our kernel, making our lives much easier.

<!-- {{toc}} -->

# Memory Allocators

To better understand global allocators, we'll create a simple linear allocator to allocate memory from a fixed-size buffer. This allocator will only be able to allocate memory, not free it, but it will be enough to get us started.

A linear allocator - sometimes also called an arena allocator - just keeps track of the current index of the buffer and allocates memory from there - just as simple as it can get. These allocators are very fast but also very limited in their use cases. In the real world, they are often used where you need to allocate a lot of memory and then free it all at once, like in a game engine. They can also be used where you know you will only need a small amount of memory and want to avoid dealing with the overhead of a more complex allocator, like in embedded systems.

{{ figure(caption = "Linear Allocators", position="center", src="./assets/linear-allocator.svg") }}

First, we'll create a new file, `src/linear-allocator.rs` and will create the basic data structure for our allocator:

{{ file(name = "src/linear-allocator.rs") }}

```rust

use core::sync::atomic::{AtomicUsize};

pub struct LinearAllocator {
    head: AtomicUsize, // the current index of the buffer
    // AtomicUsize is a special type that allows us to safely share data
    // between threads without using locks

    start: *mut u8, // raw pointer to the start of the heap
    end: *mut u8,   // raw pointer to the end of the heap
}

// allow our allocator to be shared between threads
unsafe impl Sync for LinearAllocator {}

impl LinearAllocator {
    // create a new, empty allocator
    pub const fn empty() -> Self {
        Self {
            head: AtomicUsize::new(0),
            start: core::ptr::null_mut(),
            end: core::ptr::null_mut(),
        }
    }

    // initialize the allocator with a pre-allocated buffer of memory
    // - start is a raw pointer to the start of the buffer
    // - size is the length of the buffer
    pub fn init(&mut self, start: usize, size: usize) {
        self.start = start as *mut u8;
        self.end = unsafe { self.start.add(size) };
    }
}
```

## Global Allocator

We'll also need to implement the `GlobalAlloc` trait, so Rust's `#[global_allocator]` compile time built-in knows how to use our allocator.

This trait has two methods: `alloc` and `dealloc`. We'll only implement `alloc` for now since we won't be able to free memory with our implementation.

The trait also requires marking our implementation as `unsafe` since we are dealing with raw pointers and memory addresses.

{{ file(name = "src/linear-allocator.rs") }}

```rust

use core::alloc::{GlobalAlloc, Layout};
use core::ptr::NonNull;
use core::sync::atomic::{Ordering};

unsafe impl GlobalAlloc for LinearAllocator {
        unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        /* The byte multiple that our allocated memory must start at
           most hardware architectures perform better when reading/writing
           data at aligned addresses (e.g. 4 bytes, 8 bytes, etc.) so we
           need to make sure that our memory is aligned properly
        */
        let align = layout.align();

        // The size is the number of bytes we need to allocate
        let size = layout.size();

        let mut head = self.head.load(Ordering::Relaxed);

        // Align the head to the required alignment
        // e.g. if head is 1 and align is 4, we need to add 3 to head to get 4
        if head % align != 0 {
            head += align - (head % align);
        }

        // Move the head forward by the size of the allocation
        let new_head = head + size;

        // are we out of memory?
        if self.start.add(new_head) > self.end {
            return core::ptr::null_mut();
        }

        self.head.store(new_head, Ordering::Relaxed);
        NonNull::new_unchecked(self.start.add(head) as *mut u8).as_ptr()
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // no-op
    }
}
```

Before we can start using our allocator, we need to give it a region of memory use. We'll do this in our `src/heap.rs` file:

{{ file(name = "src/heap.rs") }}

```rust

#[global_allocator]
static mut KERNEL_HEAP_ALLOCATOR: LinearAllocator = LinearAllocator::empty();

// this will allocate 128kb of memory in the .bss section
static mut KERNEL_HEAP: [u8; 0x20000] = [0; 0x20000];

/// Initialize the heap allocator.
pub unsafe fn init_kernel_heap() {
  let heap_start = KERNEL_HEAP.as_ptr() as usize;
  let heap_size = KERNEL_HEAP.len();
  KERNEL_HEAP_ALLOCATOR.init(heap_start, heap_size);
}
```

{{ file(name = "src/main.rs") }}

```rust

#![no_std]
#![no_main]
#![feature(panic_info_message)]
#![feature(allocator_api)] // new
#![feature(lazy_cell)]
#![allow(unused)]

extern crate alloc; // new
extern crate riscv_rt;

use riscv_rt::entry;
mod panic_handler;
mod utils;

mod linear_allocator; // new
mod heap; // new

#[entry]
fn main(a0: usize) -> ! {
    println!("Hello world from hart {}!\n", a0);

    // Setup everything required for the kernel to run
    unsafe {
        heap::init_kernel_heap(); // new
    }

    utils::shutdown();
}
```

Now, we can use our allocator to allocate some memory! Let's create a `Vec` and push some values to it to make sure everything works as expected:

{{ file(name = "src/main.rs") }}

```rust

let mut v = Vec::new();

v.push(1);
v.push(2);
v.push(3);

println!("{:?}", v);

// [1, 2, 3]
```

This is great, but we can only do a little with this allocator since we can't free memory. Alternative allocation strategies are, for example, [linked list allocators](https://os.phil-opp.com/allocator-designs/#linked-list-allocator), [binary buddy allocators](https://www.kernel.org/doc/gorman/html/understand/understand009.html), and [slab allocators](https://www.kernel.org/doc/gorman/html/understand/understand011.html). We won't be implementing any of these in here, but I encourage you to read about them and try to implement them yourself! Some of these are also available as crates on crates.io to use them as drop-in replacements for our naive implementation here ([linked_list_allocator](https://crates.io/crates/linked-list-allocator), [buddy_system_allocator](https://crates.io/crates/buddy_system_allocator) and [slabmalloc](https://crates.io/crates/slabmalloc)).

# Shell

With most of the essential rust features available, we can now start building our shell. This shell will allow us to interact with our kernel and inspect its state.
The shell will be a simple loop that reads characters from SBIs `console_getchar function and executes some basic commands.

{{ file(name = "src/main.rs") }}

```rust

pub const ENTER: u8 = 13;
pub const BACKSPACE: u8 = 127;

pub fn shell() {
    print!("> ");

    let mut command = String::new(); // heap allocated strings!

    loop {
        match sbi::legacy::console_getchar() {
            Some(ENTER) => {
                println!();
                process_command(&command);
                command.clear();
                print!("> ");
            }
            Some(BACKSPACE) => {
                if command.len() > 0 {
                    command.pop();
                    print!("{}", BACKSPACE as char)
                }
            }
        }
    }
}

fn process_command(command: &str) {
    match command {
        "help" | "?" | "h" => {
            println!("available commands:");
            println!("  help      print this help message  (alias: h, ?)");
            println!("  shutdown  shutdown the machine     (alias: sd, exit)");
        }
        "shutdown" | "sd" | "exit" => util::shutdown(),
        "" => {}
        _ => {
            println!("unknown command: {command}");
        }
    };
}
```

Now, when we run our kernel, we'll be greeted with a prompt, and we can type in commands like `help` and `shutdown` to see the available commands and shut down the machine.

```
> help
available commands:
  help      print this help message  (alias: h, ?)
  shutdown  shutdown the machine     (alias: sd, exit)
> exit
```

## Debugging

Being able to shut down the machine is great and all, but let's add some more functionality. We'll start by adding commands to trigger different exceptions so we can test our exception handler from the previous chapter.

{{ file(name = "src/main.rs") }}

```rust

match command {
    // ...
    "pagefault" => {
        // read from an invalid address to trigger a page fault
        unsafe { core::ptr::read_volatile(0xdeadbeef as *mut u64); }
    }
    "breakpoint" => {
        // ebreak triggers a breakpoint exception, a trap that can be used for debugging with gdb or similar tools
        unsafe { asm!("ebreak") };
    }
    // ...
}
```

When we run the `pagefault` command, we'll see that nothing happens. This is because we still need to set up a page table, and the kernel is still running in physical memory, something we want to change in the next chapter.

Our breakpoint command, however, will trigger a breakpoint exception, and we'll see the following output:

```
Exception handler called
Trap frame: TrapFrame { ra: 2149605116, t0: 9, t1: 2149632682, t2: 22, t3: 5760, t4: 48, t5: 12288, t6: 1, a0: 8, a1: 0, a2: 2149663232, a3: 2149593222, a4: 110, a5: 2149663744, a6: 4, a7: 1 }
panicked at 'Exception cause: Exception(Breakpoint)', kernel/src/trap.rs:33:5
```

We can now see the trap frame and the exception cause, so we can start debugging our kernel. You can also use an external debugger like [gdb](https://www.gnu.org/software/gdb/), which allows you to set breakpoints, inspect memory, step through code, and much more.
Setting up gdb is a bit more involved, so if you run into issues, check out this [guide](https://os.phil-opp.com/set-up-gdb/).

There are a lot of improvements we can make to this shell. For some ideas, check out my `simple_shell` crate on [crates.io](https://crates.io/crates/simple_shell). I recommend you try implementing some of these yourself, as it's a fun exercise. Something that might be helpful is this ANSI escape code [cheat sheet](https://gist.github.com/fnky/458719343aabd01cfb17a3a4f7296797) for processing user input like arrow keys, backspace, etc.

After this relatively short chapter, we'll add multi-tasking to our kernel using async Rust so we can run multiple programs simultaneously.

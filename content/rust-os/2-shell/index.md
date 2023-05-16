+++
transparent = true
title = "Creating a Kernel in Rust #2: Shell [Work in Progress]"
description = "Creating a simple shell for our kernel to run commands and help us debug"
date = 2023-05-14

[taxonomies]
tags = ["rust", "riscv", "kernel"]
series = ["rust-os"]
+++

{% quote (class="info")%}

This is a series of posts about my journey creating a kernel in rust. You can find the code for this project [here](https://github.com/explodingcamera/pogos/tree/part-1) and all of the posts in this series [here](/series/os-dev/).

This post isn't finished yet, but I wanted to get it out so I stop procrastinating on it. More content will follow next week.

{% end %}

Now that we have a basic kernel that can print to the screen, we can start building out some more functionality.
The first thing I want to do is create a simple shell that will allow us to run some commands and more easily interact with our system.

Like I mentioned in the previous post, we can't yet use heap allocated data structures, so we'll start with implementing a Global Allocator. This will allow us to use APIs like `Box` and `Vec` anywhere in our kernel which will make our lives much easier.

{{toc}}

# Global Allocator

To better understand what a global allocator is, we'll create a simple linear allocator that will allocate memory from a fixed size buffer. This allocator will only be able to allocate memory, not free it, but it will be enough to get us started.

A linear allocator - sometimes also called a arena allocator - just keeps track of the current index of the buffer and allocates memory from there - just as simple as it can get. These allocators are very fast, but they are also very limited in their use cases. In the real world, they are often used in places where you need to allocate a lot of memory and then free it all at once, like in a game engine. They can also be used in places where you know you will only need a small amount of memory and you don't want to deal with the overhead of a more complex allocator like in embedded systems.

{{ figure(caption = "Linear Allocators", position="center", src="./assets/linear-allocator.svg") }}

First, we'll create a new file `src/linear-allocator.rs` and will create the basic data structure for our allocator:

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
    pub const fn empty() -> Self {
        Self {
            head: AtomicUsize::new(0),
            start: core::ptr::null_mut(),
            end: core::ptr::null_mut(),
        }
    }

    pub fn init(&mut self, start: usize, size: usize) {
        self.start = start as *mut u8;
        self.end = unsafe { self.start.add(size) };
    }
}
```

We'll also need to implement the `GlobalAlloc` trait, so Rust's `#[global_allocator]` compile built-in knows how to use our allocator. This trait has two methods: `alloc` and `dealloc`. We'll only implement `alloc` for now, since we won't be able to free memory until we implement a more complex allocator.

The trait also requires that we mark our implementation as `unsafe` since we are dealing with raw pointers and memory addresses.

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

Now that we have our allocator, we can use it to allocate memory in our kernel. We'll start by allocating a buffer for it to use:

{{ file(name = "src/heap.rs") }}

```rust

#[global_allocator]
static mut KERNEL_HEAP_ALLOCATOR: LinearAllocator = LinearAllocator::empty();
static mut KERNEL_HEAP: [u8; 0x20000] = [0; 0x20000]; // this will allocate 128kb of memory in the .bss section

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
    println!("Hello world from hart {}\n", a0);

    // Setup everything required for the kernel to run
    unsafe {
        heap::init_kernel_heap(); // new
    }

    utils::shutdown();
}
```

# Understanding Zig's SMP Allocator

repo snapshot: https://github.com/ziglang/zig/blob/42dbd35d3e16247ee68d7e3ace0da3778a1f5d37/lib/std/heap/SmpAllocator.zig

## Rough sketch:

- Terminologies
    - slab == page
    - slot == class size
    - class == category
    - thread == core in some places

- Bird eye of overall architecture
    - Drawing showing each core having a current page under it  
    - And a big allocation box where pages go after allocator on a core allocates a new one and hence forgets it i.e. programmer needs to manage it now :)
    - How `next_addrs` and `frees` looks like in a page? If possible show a simple case and a complex case
    - How do `slot`, `slab` and `class` relate to a page and and in turn a core?
    - Important constraints mentioned in topmost comment of file
        - No resizes allowed
        - To be used in smp scenarios
        - How to handle allocations which are large?
        - thread local metadata array is the same number as CPU count and importance of it -> better use of `free`d memory and hence less "wastage"
    - Importance of cycling through cores

- Understanding code: https://github.com/ziglang/zig/blob/0.14.0/lib/std/heap/SmpAllocator.zig
    - `global` Global variable
    - threadlocal `thread_index` variable
    - mathy section, give example of answers of equations assuming 64-bit system
    - `Thread` class
        - importance of avoiding false sharing, and how that `_` achieves that
        - meaning of mutex, next_addrs and frees, draw a simple diagram for showing overall architecture
        - lock method: Why does lock have two places where lock is getting acquired
    - `alloc` method
        - Mentioning case of big allocations and early exits
        - Finding appropriate class to fit allocation in, give an example by solving equations in `sizeClassIndex` and `slotSize`
        - Why `free` list is the first thing we check? -> encourage reuse of resources
        - Next checking `next_addrs`, for slot if not found in free list
        - Core switching part
        - Importance of why `max_alloc_search` is set at 1 i.e. 2 iterations: 0 and 1
        - Why locking logic from `lock` method of `Thread` shows up again?
    - simple `free` implementation

- Bonus
    - What is vtable doing here?
        - Use of `resize` and `remap` methods
        - Their special handling of big size alloc/free cases

## Discussing benchmarks given in devlog:

- https://ziglang.org/devlog/2025/#2025-02-07

## Discussing comments on lobsters:

- https://lobste.rs/s/itcqle/good_memory_allocator_200_lines_code
- deep dive into matkald's comment, reasoning about it is important along side benchmark results on your machine
    - https://github.com/andrewrk/CarmensPlayground/pull/1
    - Try to include what `snmalloc` and `mimalloc` actually do to solve more producers, less consumers scenario
- TODO: check other comments and discuss them here if needed

## Conclusion:

- Why building allocators is hard?
- Give shoutout to Andrew's repo of allocator playground

## Doubts:

- Is this a core or not? We know for sure this does get executed by a thread but most probably a thread scheduled on that core
- highlight exact places where thread and cores can be confused

Before starting let's clear some terminology which will be used throught the blog and code, few things are confusing and I have written down their equivalent to understand them better.

## Terminology:

A `slab` described in code is equal to a OS `page`.

There are different allocation size classes, classes being zig's `mem.Alignment` [enum](https://ziglang.org/documentation/master/std/#std.mem.Alignment).

Each variant of this enum when mapped to smp allocator is called `class` or `category`.

So, a `slab` will belong to a `class`, and will contain array of `class` sized elements, each of this element is called `slot`.

<!-- // RECHECK(feniljain): Do we wanna keep this point about thread == core in some places? -->
And, a thread and core are used interchangebly throughout, they are definitely not the same! But a thread gets scheduled on a core at a time, so when a thread allocates something,
that is allocation on that core. They are used interchanbly in this sense. I will try to make sure to disambiguate this throughout the blog post.

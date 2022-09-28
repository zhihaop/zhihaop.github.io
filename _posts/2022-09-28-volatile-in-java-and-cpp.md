---
layout: post
title:  "Volatile in Java and C/C++"
date:   2022-09-28 17:25:36 +0800
categories: jekyll update
---
## Volatile in Java and C/C++

The `volatile` keyword are available in many programming language, including C, C++ and Java.

But the C/C++ and Java `volatile` keywords has the different semantics.

## Reordering

On modern platforms, code is frequently not executed in the order it was written. It is reordered by the compiler, the processor and the memory subsystem to achieve maximum performance. On multiprocessor architectures, individual processors may have their own local caches that are out of sync with main memory. It is generally undesirable to require threads to remain perfectly in sync with one another because this would be too costly from a performance point of view. This means that at any given time, different threads may see different values for the same shared data. reference: [wikipedia](https://en.wikipedia.org/wiki/Java_memory_model).

Reordering means the reader thread may see those writes in an order other than the actual program order. For example, thread(1) writes `x=1,y=2` sequentially, but after threads(2) see `x=1`, the variable `y` may have not changed.

## Memory Visbility

The writes may be not visiable by the other processor, because individual processors may have their own local caches that are out of sync with main memory.

When the thread(1) update `x=0` to `x=1`, there is no guarantees about what the other threads may see. In other words, the other threads may see `x=0` after thread(1) updated.

## The Java `volatile`

The Java `volatile` keyword is a way to control `memory order`. It provide the memory visbility between multiple threads and prevent the reordering. It guarantees the `happens-before` relationship between the writer and the reader.

Let's take an example.

```Java
class Example {
    private static int number;
    private static volatile boolean ready
}
```

The writer thread do the following instructions below.

```Java
    Example.ready = true;
    Example.number = 1;
```

The reader thread will get the corrent `number` after the `ready` was set to `true` in the writer thread, because the strength of `happens-before` memory ordering and the memory visbility of `volatile`.

```Java
    while (!Example.ready) {
        Thread.yield();
    }

    System.out.println(number);  // output: 1
```

## The C/C++ `volatile`

The volatile in C/C++ is **not** guarunteed the memory order of the modified variable, it just tells the compiler not to optimize the variable since the variable may be changed unexpectedly:

- memory-mapped I/O devices
- uses of variables between `setjmp` and `longjmp`
- uses of `sig_atomic_t` variables in signal handlers.

Operations on volatile variables are not atomic, nor do they establish a proper `happens-before` relationship for threading.

## The internal of Java `volatile`

The Java `volatile` is similar to C11 (C++11) `memory_order`. It use `memory_barrier` to guarantee `happens-before` semantics.

The reading of Java `volatile` variable acts as `acquire` sematics.

- equivalent to `std::memory_order_acquire`.
- A load operation with this memory order performs the acquire operation on the affected memory location: no reads or writes in the current thread can be reordered before this load. All writes in other threads that release the same atomic variable are visible in the current thread. reference [cppreference](https://en.cppreference.com/w/cpp/atomic/memory_order).

The writing of Java `volatile` variable acts as
`release` sematics.

- equivalent to `std::memory_order_release`.
- A store operation with this memory order performs the release operation: no reads or writes in the current thread can be reordered after this store. All writes in the current thread are visible in other threads that acquire the same atomic variable (see Release-Acquire ordering below) and writes that carry a dependency into the atomic variable become visible in other threads that consume the same atomic. reference [cppreference](https://en.cppreference.com/w/cpp/atomic/memory_order).

## Conclution

The C/C++ and Java `volatile` keywords has the completely different semantics. The C/C++ `volatile` just to tells the compiler not to optimize the variable since the variable may be changed unexpectedly. The Java `volatile` is more like a C/C++ atomic operation, and provides `happens-before` semantics.

## Recommended Links

[volatile vs. volatile](https://www.drdobbs.com/parallel/volatile-vs-volatile/212701484)

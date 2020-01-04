# Memory ordering

When writing code for multiple threads there are several subtle things we need to consider. You see, both compilers and CPU's reorder the code we write if they think it will lead to faster execution. In single threaded programs, this is not something we need to consider, but once we start writing multithreaded programs the compiler reordering can get us into truble.

However, while the compiler ordering is possible to check by looking at the disassembled code, things get much more difficult on systems with multiple CPUs.

When threads are run on different CPUs, the internal reordering of instruction on the CPU can lead to some very hard to debug problems since we mostly observe the side effects of CPU reordering, speculative execution, pipelining and caching. 

Let's start at the bottom and work our way up to a better understanding.

{% hint style="info" %}
I'll skip compiler reordering since it's pretty easy to understand. Just know that your code most likely will not be compiled chronologically the way you write it. However, this only applies to code that can correctly be reordered. Code that depends on previous steps will of course not be reordered at random.
{% endhint %}

### CPU Caches

We normally assume a CPU has three levels of caching. L1, L2, and L3. While L2, and L3 are shared between cores, L1 is a per core cache. Our challenges start here.

The L1 cache uses a sort of [MESI caching protocol](https://en.wikipedia.org/wiki/MESI_protocol). While the name might sound mysterious, it's actually pretty simple. It is an acronym for the different state an item in the cache can find themselves in:

```text
(M) Modified - modified (dirty). Need to write back data to main memory.
(E) Exclusive - only exists in this cache. Does not ned to be synced (clean).
(S) Shared - might exist in other caches. Is current with main memory (clean).
(I) Invalid - cache line is invalid. Another cache has modified it.
```

{% hint style="info" %}
**Does this sound familiar?**

In rust we have two kind of references `&`shared references, and `&mut`exclusive refrences. This does indeed map very well to the E and S, two of the possible states data can have in the L1 Cache. Modelling this in the language can \(and does\) provide the possibility of optimizations which languages without these semantics can't do. 

In Rust, only memory which is `Exclusive`can be modified by default. 
{% endhint %}

Ok, so we can model this for ourselves by thinking that every cache line in the CPU has an `enum`with four states attached to them.

### Inter Processor Communication

So, if a cache line is invalidated if the data exists in a different cache and is modified there, there must be some way for the processors to communicate with each other?

The answer is yes. However, it's pretty hard to find documentation about the exact details, each CPU has what we can think of as a _mailbox_.

This mailbox is what's important for us when we're going to talk about atomics later on. This mailbox can buffer a certain number of messanges. Each message is saved here to avoid interrupting the CPU all the time. Now, at some point the CPU either checks this mailbox, or I've also seen talks about designs which issues an interrupt to the CPU when it needs to process messages.

Let's take an example of a cache line which is marked as `Shared`. If a CPU modifies this cache line, it is invalid, so the rest of the CPU's gets a message in their inbox to mark this cache line as invalid, and fetch the correct value from main memory \(or L2/L3 cache\).

### Memory Ordering

Now that we have some idea of how the CPU's are designed to coordinate between them, we can talk about different memory orderings and what they mean:

In Rust the memory ordering is represented by the  [std::sync::atomic::Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) enum, which has 5 possible values:

### Relaxed

No messages are sent or retrieved and proccessed from the inbox before this operation. This is therefore the weakest of the possible memory orderings. It implies that the operation does not do any specific synchronization with the other CPUs.

### Acquire

All messages in the inbox are read and proccessed from the inbox before the operation. This means that all cache lines any other CPU has modified, is marked as `Invalidated`meaning that we'll need to fetch new values from memory if our operation involves reading such a value. For this exact reason, `Acquire`only makes sense in _load_ operations. **Most `atomic`methods in rust which involves stores will panic if you pass inn `Acquire`as the memory ordering of a `store`operation.**

### Release

All pending messages on the CPU are flushed and sent to the other CPUs mailboxes. This is most likely values which was `Shared`but we have modified, so they're now marked as `Modified`in our cache. Before we proceed with any operation, we flush the changes to this data to main memory and sends a message to all the others that they'll need to mark this cache line as `Invalid`. `Release`memory ordering only makes sense on `store`operation. **For this reason, and opposite of the `Acquire`ordering, most `load`methods in Rust will panic if you pass in a `Release`ordering.**

### AcqRel

First process all messages in the inbox \(as we do in `Acquire`\) and them flush changes and notify the other CPUs about changes we have made \(as we do in `Release`\).

### SeqCst

Same as `AcqRel`but it also preserves an sequentially consistent order between operations that are marked with `SeqCst`. It's a bit hard to understand, but think of it like special messages which are marked with a timestamp. The timestamp is only attached with messages sent to the Mailbox of the CPU from a thread which did an `AcqRel`as a part of `SeqCst`. This timestamp is only read if you are are performing a `AcqRel`as a part of a `SeqCst`operation yourselves. You order them from frist to last and process them that way. 

If you have three threads, A, B and C. All performing `SeqCst`operations on the same memory, they'll all read each other messages in order. Thereby agreeing on what happened in which order.

One important thing to note is that if there is any `Acquire`, `Release`or `Relaxed`operations on this memory, the sequential consistency is lost since there is no way to know when that operation happened and thereby agree on a total order of operations.

**SeqCst is the strongest of the memory orderings, it also has a slightly higher cost than the others.**

{% hint style="info" %}
I have a hard time coming up with a good example where this ordering is the only one that solves the problem, and which can't be solved by using the weaker orderings. 

_However, reasoning about `Acquire`and `Release`in complex scenarios can be hard. If you only use `SeqCst`on a part of memory you'll know that you have the strongest memory ordering and you're likely on the "safe side". It makes working with atomics a lot more convenient._
{% endhint %}

To be able to solve this we need to learn a bit about multithreaded code and synchronization.

### Atomics

Atomics live in the `std::sync::atomic`module in the Rust standard library. There is especially two kind of atomics which is of interest for us: `AtomicUsize`and `AtomicBool`. To be able to use atomic operations in Rust we also need to know how to use `std::sync::atomic::Ordering`enum which we use to indicate what kind of memory fence we want to use.

But before we go into that, let's start at the beginning.

### Why do we need atomic operations?

There are two main reasons for this:

1. Compiler reordering
2. CPU caching and reordering

#### Compiler reordering

The compiler doesn't promise to execute your code exactly in the order you wrote it. It does however, promise to not reorder the code in a way which changes the end result. This is true even for single threaded code. It can optimize your code in many ways, and therefore, problems related to compiler reordering are often most pronounced on optimized builds.

#### CPU reordering

However, the CPU is no better. While we can inspect the assembly outputted by the compiler, the CPU can reorder these instructions as well. It can even reorder the order in which data is fetched from memory. All this is done to improve efficiency and performance.

{% hint style="info" %}
Diving into exactly how this works quickly deserves an entire book itself. Fortunately, a lot of excellent material is already written about this subject. Here are some of the best references I've found while researching this:

_A series of articles explaining how memory ordering, fences, barriers and atomic operations work:_  
[https://preshing.com/20120612/an-introduction-to-lock-free-programming/](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)[https://preshing.com/20120625/memory-ordering-at-compile-time/](https://preshing.com/20120625/memory-ordering-at-compile-time/)[  
https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)[https://preshing.com/20120913/acquire-and-release-semantics/](https://preshing.com/20120913/acquire-and-release-semantics/)  
[https://preshing.com/20120930/weak-vs-strong-memory-models/](https://preshing.com/20120930/weak-vs-strong-memory-models/)  
[https://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu/](https://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu/)

_A short introduction on explaining how to use the different memory orderings `Relaxed`, `Acquire`, `Release`and `SeqCst`in practice:_  
[https://vorner.github.io/2018/03/25/Atomics.html](https://vorner.github.io/2018/03/25/Atomics.html)

_A paper discussing the how these instructions is implemented in the hardware:_  
[http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.07.23a.pdf](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.07.23a.pdf)

_A long an extensive explanation on how the Linux kernel handles memory barriers:_  
[https://www.kernel.org/doc/Documentation/memory-barriers.txt](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
{% endhint %}

We'll focus on a brief and quick introduction in how and why this matters to us:

### A bad counter - unsynchonized memory access

Let's start with a very naive and unsafe way of doing this and have a look at the different options we have and why we use them. This will be the example that we improve in a few steps while trying to explain along the way.

```rust
#![feature(asm)]

use std::thread;
static mut LOCKED: bool = false;
static mut COUNTER: usize = 0;

pub fn test(inc: usize) -> usize {
    while unsafe { LOCKED } {}
    unsafe { LOCKED = true };
    unsafe { COUNTER += inc};
    unsafe { LOCKED = false };
    unsafe { COUNTER }
}

fn main() {
    let mut handles = vec![];
    for i in 0..5 {
        let handle = thread::spawn(move || {
            let mut prevent_optimization = Vec::with_capacity(100_000);
            for _ in 0..100_000 {
                let res = test(1);
                prevent_optimization.push(res);
            }
            let last_val = prevent_optimization.last().unwrap();
            println!("{} finished. Last value: {}", i + 1, last_val);
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("{}", unsafe { COUNTER });
}

```

The code simply providing two global constants. A flag indicating if we have a "lock" on the `COUNTER`and a counter we want to increase. We expect this counter to be at 200 000 once the program is finished. 

{% hint style="info" %}
We need to let the function return a value and store that value to prevent the compiler to optimizing away the entire loop in `release`builds. If we don't do that the compiler is smart enough to simply replace the entire loop with a simple `100_000*1`instruction. This results in two threads adding 100\_000 to our `COUNTER` and that's not enoguh to show what we want.
{% endhint %}

Let's run this with both a debug build and a release build:

{% tabs %}
{% tab title="Debug" %}
```text
Finished dev [unoptimized + debuginfo] target(s) in 0.01s
Running `target\debug\memory_ordering.exe`
126093
```
{% endtab %}

{% tab title="Release" %}
    Finished release [optimized] target(s) in 0.01s
    Running `target\release\memory_ordering.exe`
    180810
{% endtab %}
{% endtabs %}

OK, so the debug build gave us a count of 126 093. Pretty far off. The release build gives us the count of 180 810. Better but still not correct. 

The reason for this is that both thread reads and writes to the same memory address. You see, each core has a small cache called the `L1`cache which is not shared with the other cores. What happens is basically what I try to show in this figure:

![](../.gitbook/assets/bilde%20%288%29.png)

Both caches holds the same value for the `COUNTER`and increments that, which of course results in the exact same number. Then in random order they overwrite each others data with the same value. This happens most of the time but not all of the time, so that's why we'll see the number above 100 000, but less than 200 000.

When we compile with `cargo run --debug`we get this assembly output \(abbreviated\)

Let's try to make our code a little bit better by introducing atomics:

### A slightly better counter - introducing atomics

Atomics are not special data types in the eyes of the CPU. CPU's mostly deal with integers of different sizes. However, when we use atomics the compiler knows we are dealing with data which it can't reorder freely, and in addition we get access to some special instructions on the CPU we'll use later on. These instructions introduce memory barriers which can help us safely share and synchronize data between cores.

{% hint style="info" %}
In the following examples, the `main`function stays the same as in the first one so I omit that here.
{% endhint %}

```rust
use std::thread;
use std::sync::atomic::{AtomicBool, Ordering};

static LOCKED: AtomicBool = AtomicBool::new(false);
static mut COUNTER: usize = 0;

pub fn test(inc: usize) -> usize {
    while LOCKED.load(Ordering::Relaxed) {}
    LOCKED.store(true, Ordering::Relaxed);
    unsafe { COUNTER += inc };
    LOCKED.store(false, Ordering::Relaxed);
    unsafe { COUNTER }
}

fn main() {
 ...
}
```

The results from running this code is:

We know the compiler knows we're dealing with atomics, but let's have a look at what instructions the CPU get when it comes to aquire the lock:

![](../.gitbook/assets/bilde%20%285%29.png)

{% hint style="info" %}
The assembly output is retrived by using the [Rust Playground](https://play.rust-lang.org/). We only look at the assembly output from the `release`builds.
{% endhint %}

### An even better counter - using cmpex instruction

```rust
use std::thread;
use std::sync::atomic::{AtomicBool, Ordering};

const LOCKED: AtomicBool = AtomicBool::new(false);
static mut COUNTER: usize = 0;

pub fn test(inc: usize) -> usize {
    while LOCKED.compare_and_swap(true, false, Ordering::Relaxed) { }
    unsafe { COUNTER += inc};
    LOCKED.store(false, Ordering::Relaxed);
    unsafe { COUNTER }
}

fn main() {
  ...
}
```

The assemby output explaines why

![](../.gitbook/assets/bilde%20%287%29.png)

### A working counter - using Acquire/Release

```rust
use std::thread;
use std::sync::atomic::{AtomicBool, Ordering, fence};

static LOCKED: AtomicBool = AtomicBool::new(false);
static mut COUNTER: usize = 0;

pub fn test(inc: usize) -> usize {
    while LOCKED.compare_and_swap(false, true, Ordering::Acquire) {}
    unsafe { COUNTER += inc };
    LOCKED.store(false, Ordering::Release);
    unsafe { COUNTER }
}

fn main() {
    ...
}
```

Let's have a look at the assembly output

![](../.gitbook/assets/bilde%20%2813%29.png)


# Memory ordering

In our library we have the following code to avoid registering interest in new events to an event queue which has recieved a close signal:

```rust
pub fn register(
    &self,
    soc: &mut TcpStream,
    token: usize,
    interests: Interests,
) -> io::Result<()> {
    if self.is_poll_dead.load(Ordering::SeqCst) {
        return Err(io::Error::new(
            io::ErrorKind::Interrupted,
            "Poll instance is dead.",
        ));
    }
    
    // WHAT HAPPENS IF ANOTHER THREAD CLOSES THE LOOP AT THIS EXACT MOMENT?
    
    ffi::create_io_completion_port(soc.as_raw_socket(), self.completion_port, 0)?;

    let op = ffi::Operation::new(token);
    soc.operations.push_back(op);

    if interests.is_readable() {
        ffi::wsa_recv(
            soc.as_raw_socket(),
            &mut soc.wsabuf,
            soc.operations.back_mut().unwrap(),
        )?;
    } else {
        unimplemented!();
    }

    Ok(())
}
```

When writing code for multiple threads there are several subtle things we need to consider. In the case of this exact code, what happens if we were to have several registrators on different threads and one is closing the loop at the same time another is registering an interest?

Well, while we do check that the `Poll`instance is not dead before we register an event there is nothing in our code which prevents another thread to aquire the `is_poll_dead`flag before we're finised with registering an event. In which case we'll wait for a notification that never occurs.

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


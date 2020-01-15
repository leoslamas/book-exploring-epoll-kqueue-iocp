# Public API

So let's get starting, first out is our public API in `lib.rs`. Let's implement method by method what we presented in the previous chapter:

### Event and Token

We start off easy. `Events`is just an ordinary `Vec`of events which are defined in our platform specific modules. `Token`is just a type alias for `usize`. I only include it since it's very common to use `Token`instead of an `usize`to identify an event. The reason for this is that contrary to what we do, another implementation could pass inn a pointer to som data or anything else than a simple number to identify a specific event.

```rust
pub type Events = Vec<Event>;
pub type Token = usize;
```

### Poll

Let's start by defining the `Poll`struct itself.

```rust
#[derive(Debug)]
pub struct Poll {
    registry: Registry,
    is_poll_dead: Arc<AtomicBool>,
}
```

So `Poll`is pretty simple. It contains a `Registry`which we'll define below, and a flag `is_poll_dead`which indicates if this `Poll`instance has recieved a close signal or not. This needs to be an `AtomicBool`wrapped in an `Arc`since we'll pass on a reference to this flag when we create a `Registrator`. 

{% hint style="info" %}
`Arc`is an atomic reference counted smart pointer in Rust. It's similar to the smart pointer `Rc`but it's thread safe. 
{% endhint %}

The interesting part is the implementation of `Poll`:

```rust
impl Poll {
    pub fn new() -> io::Result<Poll> {
        Selector::new().map(|selector| Poll {
            registry: Registry { selector },
            is_poll_dead: Arc::new(AtomicBool::new(false)),
        })
    }

    pub fn registrator(&self) -> Registrator {
        self.registry
            .selector
            .registrator(self.is_poll_dead.clone())
    }
    
    pub fn poll(&mut self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<usize> {
        // A negative timout is converted to a 0 timeout
        let timeout = timeout_ms.map(|n| if n < 0 { 0 } else { n });
        loop {
            let res = self.registry.selector.select(events, timeout);
            match res {
                Ok(()) => break,
                Err(ref e) if e.kind() == io::ErrorKind::Interrupted => (),
                Err(e) => return Err(e),
            };
        }

        if self.is_poll_dead.load(Ordering::SeqCst) {
            return Err(io::Error::new(io::ErrorKind::Interrupted, "Poll closed."));
        }

        Ok(events.len())
    }
}
```

We have three important methods here. 

First `new`instanciates a new instance of Poll. If you're not very familiar with Rust yet, I'll explain what we do here since it can be hard to understand:

`Selector::new()`returns a `io::Result<Selector>`. The `Result`enum in Rust has a method called `map`. This method is only called if the result is `Result::Ok( ... )`, and passes the value `...`into the closure we provide. So if the call to `Selector::new()`does not return an `Err`we create a `Poll`instance in the closure:

```rust
|selector| Poll {
    registry: Registry { selector },
    is_poll_dead: Arc::new(AtomicBool::new(false)),
}
```

Next method is the `registrator`method. This method returns a `Registrator`. This is the only way to create a `Registrator`since a `Registrator`which is not tied to a `Poll`instance makes no sense.

We create a registrator by calling the platform specific `registrator`method on the platform specific `Select`instance. By calling `clone`on our `Arc<AtomicBool>`we increase the reference count and recieve a reference to the `is_poll_dead`flag in our `Poll`instance. 

The last method on the `Poll`instance is `poll`and let's walk through this code as well.

`timeout`is an `Option<i32>`and since a negative timeout makes no sense we use the same technique as we used in the `new`method to convert a negative timeout to `0`if timeout is `Some( ... )`.

```rust
let timeout = timeout_ms.map(|n| if n < 0 { 0 } else { n });
```

Next we create a loop which actually waits for events to be ready. If the `select`call returns `Ok(())`we break out of the loop. However, you might wonder why we call this in a loop at all?

The answer is on the next line. If the error is of `kind` `io::ErrorKind::Interrupted`we actually do nothing an just call `select`again. On any other error type we exit the loop and return the error.

{% hint style="info" %}
The reason for special casing `Interrupted`is whats called  [Spurious Wakeup](https://en.wikipedia.org/wiki/Spurious_wakeup), and it's expected by all platforms that we account for this condition to happen. The OS doesn't guarantee that it only wakes up the thread on an event, it could happen if certain conditions occur on the same time with the result that our thread is woken up and no event has occured.

In this case we get an error of kind `Interrupted`and we simply call `select`again which will tell the OS to suspend our thread again and wake it up when an event occurs.
{% endhint %}

```rust
loop {
    let res = self.registry.selector.select(events, timeout);
    match res {
        Ok(()) => break,
        Err(ref e) if e.kind() == io::ErrorKind::Interrupted => (),
        Err(e) => return Err(e),
    };
}
```

If we get an `Ok()` we proceed to check if we've recieved a close signal while our thread was suspended. If we did we return an `Error`of kind `Interrupted`to the user of our library. We document this so the user knows to check for this error type and act accordingly.

{% hint style="info" %}
Since `Interrupted`is special cased in the `select`call there is no way for `select`to return an `Interrupted`error kind. That means we know that the only way the `poll`method will return this kind of error is in the case of a closed event queue.
{% endhint %}

Lastly we return the number of events returned by accessing the `len`of our event array.

```rust
if self.is_poll_dead.load(Ordering::SeqCst) {
    return Err(
    io::Error::new(io::ErrorKind::Interrupted, "Poll closed.")
    );
}

Ok(events.len())
```

### `Selector`

Our `Registry`is a simple struct just wrapping a `Selector`instance. All functionality is in the platform specific Selector instance.

```rust
#[derive(Debug)]
pub struct Registry {
    selector: Selector,
}
```

### `Interests`

This struct is just provides a simple way for us to to let a user express what kind of event they want to register interest in. This struct uses a somewhat uncommon technique of defining constants inside the struct itself.

The advantage of this is that we expose these constants namespaced by the `Interests`struct so it's called like `Interests::READABLE`by the user.

We also provide methods to check what kind of interests were registered. Using bitflags like this lets us register interest in multiple events by simply OR'ing different interests like `let read_write = Interests::READABLE | Interests::WRITABLE`.

{% hint style="info" %}
We only implement the `READABLE`in this library, but a nice reader excercise is to try to implement the neccesary methods to be able to use the event queue for users that are interested in `WRITABLE`events as well.
{% endhint %}

```rust
const WRITABLE: u8 = 0b0000_0001;
const READABLE: u8 = 0b0000_0010;

pub struct Interests(u8);
impl Interests {
    pub const READABLE: Interests = Interests(READABLE);
    pub const WRITABLE: Interests = Interests(WRITABLE);

    pub fn is_readable(&self) -> bool {
        self.0 & READABLE != 0
    }

    pub fn is_writable(&self) -> bool {
        self.0 & WRITABLE != 0
    }
}
```

### Full code for lib.rs

```rust
use std::io;
use std::sync::{
    atomic::{AtomicBool, AtomicUsize, Ordering},
    Arc,
};

#[cfg(target_os = "windows")]
mod windows;
#[cfg(target_os = "windows")]
pub use windows::{Event, Registrator, Selector, TcpStream};

#[cfg(target_os = "macos")]
mod macos;
#[cfg(target_os = "macos")]
pub use macos::{Event, Registrator, Selector, TcpStream};

#[cfg(target_os = "linux")]
mod linux;
#[cfg(target_os = "linux")]
pub use linux::{Event, Registrator, Selector, TcpStream};

pub type Events = Vec<Event>;
pub type Token = usize;

/// `Poll` represents the event queue. The `poll` method will block the current thread
/// waiting for events. If no timeout is provided it will potentially block indefinately.
/// 
/// `Poll` can be used in one of two ways. The first way is by registering interest in events and then wait for
/// them in the same thread. In this case you'll use the built-in methods on `Poll` for registering events.
/// 
/// Alternatively, it can be used by waiting in one thread and registering interest in events from
/// another. In this case you'll ned to call the `Poll::registrator()` method which returns a `Registrator`
/// tied to this event queue which can be sent to another thread and used to register events.
#[derive(Debug)]
pub struct Poll {
    registry: Registry,
    is_poll_dead: Arc<AtomicBool>,
}

impl Poll {
    pub fn new() -> io::Result<Poll> {
        Selector::new().map(|selector| Poll {
            registry: Registry { selector },
            is_poll_dead: Arc::new(AtomicBool::new(false)),
        })
    }

    pub fn registrator(&self) -> Registrator {
        self.registry
            .selector
            .registrator(self.is_poll_dead.clone())
    }

    /// Polls the event loop. The thread yields to the OS while witing for either
    /// an event to retur or a timeout to occur. A negative timeout will be treated
    /// as a timeout of 0.
    pub fn poll(&mut self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<usize> {
        // A negative timout is converted to a 0 timeout
        let timeout = timeout_ms.map(|n| if n < 0 { 0 } else { n });
        loop {
            let res = self.registry.selector.select(events, timeout);
            match res {
                Ok(()) => break,
                Err(ref e) if e.kind() == io::ErrorKind::Interrupted => (),
                Err(e) => return Err(e),
            };
        }

        if self.is_poll_dead.load(Ordering::SeqCst) {
            return Err(io::Error::new(io::ErrorKind::Interrupted, "Poll closed."));
        }

        Ok(events.len())
    }
}

#[derive(Debug)]
pub struct Registry {
    selector: Selector,
}


const WRITABLE: u8 = 0b0000_0001;
const READABLE: u8 = 0b0000_0010;

/// Represents interest in either Read or Write events. This struct is created 
/// by using one of the two constants:
/// 
/// - Interests::READABLE
/// - Interests::WRITABLE
pub struct Interests(u8);
impl Interests {
    pub const READABLE: Interests = Interests(READABLE);
    pub const WRITABLE: Interests = Interests(WRITABLE);

    pub fn is_readable(&self) -> bool {
        self.0 & READABLE != 0
    }

    pub fn is_writable(&self) -> bool {
        self.0 & WRITABLE != 0
    }
}

```


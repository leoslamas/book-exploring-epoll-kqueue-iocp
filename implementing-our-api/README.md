# Implementing our API

The first thing we do is to describe and implement the platform-indipendant functionality and the public surface of our library. 

Navigate to `lib.rs`and open the file.

```text
minimio
   |
   +--> src
   |     |
   |     +--> lib.rs

```

Let's just start by naming the structs and write out the function definitions we implement the API we designed in the last chapter just to get a starting point. 

```rust
pub type Events = Vec< ... >;
pub type Token = usize;

pub struct Poll { ... }

pub struct Registry { ... }

impl Poll {
    pub fn new() -> io::Result<Poll> { ... }

    pub fn registrator(&self) -> Registrator { ... }

    pub fn poll(&mut self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<usize> { ... }
}

const WRITABLE: u8 = 0b0000_0001;
const READABLE: u8 = 0b0000_0010;

pub struct Interests(u8);
impl Interests {
    pub const READABLE: Interests = Interests(READABLE);
    pub const WRITABLE: Interests = Interests(WRITABLE);
}
impl Interests {
    pub fn is_readable(&self) -> bool { ... }

    pub fn is_writable(&self) -> bool { ... }
}

```

As you might recognize from our API design we have the definitions of `Events`, `Poll`, `Registry` and `Interests` right here.

As you might notice there is no `Registrator`or `TcpStream`yet which is part of our public API, the reason for that is that we need to rely on some platform specifics for those. In addition, we need some platform specifics to actually implement our API.

It's not that much really, let's make two more changes to accomondate for the platform specific code:

### Platform specific modules

Let's create two more platform specific modules so our project structure looks like this:

```text
minimio
   |
   +--> src
   |     |
   |     +--> lib.rs
   |     +--> linux.rs
   |     +--> macos.rs
   |     +--> windows.rs
   +--> tests
   |      |
   |      +--> api.rs
```

And in our `lib.rs`define the modules:

```rust
#[cfg(target_os = "linux")]
mod linux;
#[cfg(target_os = "macos")]
mod macos;
#[cfg(target_os = "windows")]
mod windows;
```

The `[cfg(target_os = "...")]`is a conditional compilation flag. The[ cfg attrbute ](https://doc.rust-lang.org/reference/conditional-compilation.html)lets us define certain condtions that needs to be true for this part of the code to compile. In this case we compile the module if the `target_os`matches.

We'll also need some platform specific functionality to actually implement our event queue:

```rust
#[cfg(target_os = "linux")]
pub use linux::{Event, Registrator, Selector, TcpStream};
#[cfg(target_os = "macos")]
pub use macos::{Event, Registrator, Selector, TcpStream};
#[cfg(target_os = "windows")]
pub use windows::{Event, Registrator, Selector, TcpStream};
```

As you see here we pull inn exactly the same methods from each of the modules. They need to have the same API and the same signatures to work.

{% hint style="info" %}
An alternative to manually making sure that we have the same API on each arcitecture is to define `Traits`for Event, Registrator, TcpStream and make sure our platform specific structs implement these. We keep it simple though. Actually, the way we do it here is a pretty common pattern when dealing with platform specific code in Rust. 
{% endhint %}

You might wonder what these methods do, and we'll go through that in detail in the following chapters, but we'll quickly introduce them here:

#### `Event`

 Event is the representation of an event. It implements only one public method `id() -> Token`. This mehod lets us identify each event.

#### `Registrator`

The `Registrator`let's us register interest in new events to out evet queue. It publicly exposes a method called `register()`and a method `close_loop()`. The last one lets us close down the loop created on our `Poll`method.

#### `Selector`

Selector is really the heart of the whole event queue. This is what drives our `Poll`implementation. It exposes the `select()`method which actually blocks the current thread and returns once an event is ready, and a `registrator()`method which returns a `Regstrator`uniquely tied to this specific event queue.

#### `TcpStream`

Is actually supposed to mimick `std::net::TcpStream`but we'll only implement the `connect`method. However, we will also provide our own implementation of the `Read`, `Write`and `AsRawFd/AsRawSocket`traits.






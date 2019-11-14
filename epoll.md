# Epoll

Epoll is the Linux way of providing a scalable event queue. 

## The raw materials in Epoll

### `epoll_create`

Similarely to `CreateCompletionPort`and `kqueue`, the `epoll_create`syscall creates an epoll instance and returns a file handle to us.

{% embed url="http://man7.org/linux/man-pages/man2/epoll\_create1.2.html" %}

### `epoll_ctl`

The `epoll_ctl`syscall is used to add, modify or remove events from the event queue.

{% embed url="http://man7.org/linux/man-pages/man2/epoll\_ctl.2.html" %}

### `epoll_wait`

The `epoll_wait`syscall suspends \(deschedules\) the thread it's called from, thereby blocking any further progress on that thread until either an event has occured or a provided `timeout`has expired.

{% embed url="http://man7.org/linux/man-pages/man2/epoll\_wait.2.html" %}

### `eventfd`

While not strictly a part of `epoll`this system call creates an `eventfd`object that can be used as an event "notify" mechanism. In other words, this lets us issue an timeout event which wakes up the thread which is waiting after calling the `epoll_wait`syscall. We are going to use this to be able to wake up our thread when we want to close the event queue and clean up after ourselves.

{% embed url="http://man7.org/linux/man-pages/man2/eventfd.2.html" %}

### `close`

The `close`syscall closes a file descriptor so it no longer points to a resource and can be reused.

{% embed url="http://man7.org/linux/man-pages/man2/close.2.html" %}

## Useful structures

### Event struct

```rust
#[repr(C)]
pub struct Event {
    /// The events we want to register interest in represented by a bitflag.
    events: u32,
    // User defined variable (we use this to identify this exact event)
    epoll_data: Data,
}
```

The `Event`struct represents information about an event we want to register interest about. Specifically, what kind of events we want to register interest for like `Read`or `Write`events.

### Data union

```rust
#[repr(C)]
pub union Data {
    void: *const c_void,
    fd: i32,
    uint32: u32,
    uint64: u64,
}
```

{% hint style="info" %}
This type might be new to you. It's in Rusts standard library but works like a C `union`. It does have a few similarities with Rusts enum in the sense that this type can represent either `c_void, i32, u32 or an u64`, but thats also where the similarities end. It's useful for us when we want to work with C APIs that expects a C-type union like we do here.
{% endhint %}

## Flags and constants

We need some flags and constants to actually work with `epoll`. We only need very few since we're just covering one small use case, but these will at least get you started to get something working.

```rust
pub const EPOLL_CTL_ADD: i32 = 1;
pub const EPOLL_CTL_DEL: i32 = 2;
pub const EPOLLIN: i32 = 0x1;
pub const EPOLLONESHOT: i32 = 0x40000000;
```

The flags are used by `or`ing different flags together. This is a pretty common way of describing a set of options in C. Read more about bitflags in the paragraph about them in the Appendix.


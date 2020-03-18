---
description: >-
  Kqueue is the relevant macanism on BSD based Operating Systems. One example of
  this is Macos. Kqueue, similarely to Epoll, is a readiness based model.
---

# Kqueue Reference

## Quick reference of important Kqueue syscalls

### `kqueue`

Kqueue is a system call which provides us with a generic method which lets us get notified when an kernel event has occured. Similarely to the `CreateCompletionPort`on Windows this returns a handle to an event queue file descriptor.

{% embed url="https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages\_iPhoneOS/man2/kqueue.2.html" %}

### `kevent`

In contrast to IOCP \(and Epoll\), we use the same syscall to register and wait for events. The `kqueue`syscall will know what to do based on the arguments we provide. As we'll see, this results in a pretty elegant API but it can be a bit hard to wrap your head around in the start. It's documented on the same manpage as `kqueue`.

### `close`

We need a way to close the file handle to our event queue. The `close`syscall lets us do just that.

### Important structures

### `Kevent(structure)`

Not to be confused with the `kevent`syscall. This structure is the most important way for us to provide information to our `kevent`syscall what we want to do.

```rust
#[derive(Debug, Clone, Default)]
#[repr(C)]
pub struct Kevent {
    /// Value used to identify this event.  The exact interpretation
    /// is determined by the attached filter, but often is a file descriptor.
    pub ident: u64,
    /// Identifies the kernel filter used to process this event.
    pub filter: i16,
    /// Actions to perform on the event
    pub flags: u16,
    /// Filter-specific flags
    pub fflags: u32,
    /// Filter-specific data value
    pub data: i64,
    /// Opaque user-defined value passed through the kernel unchanged
    pub udata: u64,
}
```

{% hint style="info" %}
We'll explain the filters and flags we need to use when we show how to use `kqueue`in practice, but please note the `udata`field. This lets us attach a value which is untouched by the OS. We need a way to identify which event that has occured and this field will be valuable for us.
{% endhint %}

### `Timespec`

 Timeouts on `BSD` is passed as a  `Timespec`struct. It's important for us since we want to be able to set a timeout for how long we want to wait for an event before the thread calling `poll`is woken up.

```rust
#[derive(Debug)]
#[repr(C)]
pub(super) struct Timespec {
    /// Seconds
    tv_sec: isize,
    /// Nanoseconds     
    v_nsec: usize,
}
```

## Useful flags and constants

```rust
pub const EVFILT_READ: i16 = -1;
pub const EVFILT_TIMER: i16 = -7;
pub const EV_ADD: u16 = 0x1;
pub const EV_ENABLE: u16 = 0x4;
pub const EV_ONESHOT: u16 = 0x10;
pub const EV_CLEAR: u16 = 0x20;
```

The exact meaning of these is described in the `Kqueue`manpage. However, getting the actual values can be pretty hard so I've gathered some of them here for you.

{% hint style="info" %}
We could have represented all of these as hex values but as I've gotten them from different sources I've left them as found.
{% endhint %}


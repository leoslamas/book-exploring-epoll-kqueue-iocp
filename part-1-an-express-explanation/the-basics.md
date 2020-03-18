# Epoll - the express version

Before we go on to create a cross platform library, let's play around with an example so it doesn't get boring too quickly.

Since `Epoll`, `Kqueue`and `IOCP`all have different API's \(and part of this book is showing a bit of all of them\), let's keep things simple in the start and start by just looking at `Epoll`.

{% hint style="info" %}
If you're on Windows I suggest you use [WSL ](https://docs.microsoft.com/en-us/windows/wsl/install-win10)to follow along on this part of the book.
{% endhint %}

Let's start by firing up a new project by creating a new folder and initialize a project. Move straight in to the `main.rs`file.

Just leave the `main` function for now and declare a new module beneath it. Next we add the `extern`function definitions we'll use to make the syscalls we need to use `epoll`on Linux:

```rust
mod ffi {
    #[link(name = "c")]
    extern "C" {
        pub fn epoll_create(size: i32) -> i32;
        pub fn close(fd: i32) -> i32;
        pub fn epoll_ctl(epfd: i32, op: i32, fd: i32, event: *mut Event) -> i32;
        pub fn epoll_wait(epfd: i32, events: *mut Event, maxevents: i32, timeout: i32) -> i32;
    }
}
```

We use `[link(name = "c")]`to tell the linker what library we want to link to, so we're able to call the functions on which is defined in the `extern "C"`block. The `C`in `extern "C"`tells the compiler that we'll use the "C" calling convention \(which we'll need to use when using the C API on Linux\).

However, we don't have everything set up yet. The syscalls expects us to pass in more than just primitives. It expects some data structures we need to define as well.

We're still writing in the `ffi`block. If we take a look at the manpage for the `epoll_ctl`function we see that we need two more definitions:

![Click to enlarge](../.gitbook/assets/bilde%20%2810%29.png)

Namely an `Event` struct and a `Data`union:

```rust
#[repr(C)]
pub union Data {
    void: *const std::os::raw::c_void,
    fd: i32,
    uint32: u32,
    uint64: u64,
}

#[repr(C)]
pub struct Event {
    events: u32,
    epoll_data: Data,
}
```

The `Event` struct is pretty familiar, right? However, the `Data` union is not something we use in Rust on a daily basis. What is that?

From the [Rust Reference on Unions](https://doc.rust-lang.org/reference/items/unions.html):

{% hint style="info" %}
The key property of unions is that all fields of a union share common storage. As a result writes to one field of a union can overwrite its other fields, and size of a union is determined by the size of its largest field.
{% endhint %}

So, this sounds a lot like `Enums`right? Turns out that they're not all that different. `Enums`can be thought of as a kind of "tagged" `union`. It requires slightly more space since it needs to carry the information about what kind was last written to the `union`but that's about all the difference there is.

Getting data from a C Union is always unsafe since we have no way of knowing that we have valid data for the type we read into. Fortunately, the `epoll_data`field is a field we provide the data for, so we can of course know what data we pass in.

Honestly, we could just use a plain `u64`here. A `C`union will write its data from the first byte anyway so the memory layout of a `Data.uint64`and a `u64`will be the same.

It's actually better for us to just pass in a concrete type since we decide what data we want to store with the `Event`object anyway. Since the `union`defined both `u32`and `u64`as valid data, we can just use an`usize` that should work. 

Let's avoid using a `union`here and change our `ffi`module to look like this:

```rust
mod ffi {
    #[link(name = "c")]
    extern "C" {
        pub fn epoll_create(size: i32) -> i32;
        pub fn close(fd: i32) -> i32;
        pub fn epoll_ctl(epfd: i32, op: i32, fd: i32, event: *mut Event) -> i32;
        pub fn epoll_wait(epfd: i32, events: *mut Event, maxevents: i32, timeout: i32) -> i32;
    }

    #[repr(C)]
    pub struct Event {
        events: u32,
        epoll_data: usize,
    }
}
```

Nice! So, let's create a `epoll`queue and use it to wait for a response to a slow server. This is the fun part!

```rust
use std::io::{self, Write};
use std::os::unix::io::AsRawFd;
use std::net::TcpStream;

fn main() {

    // A counter to keep track of how many events we're expecting to act on
    let mut event_counter = 0;

    // First we create the event queue.
    // The size argument is ignored but needs to be larger than 0
    let queue = unsafe { ffi::epoll_create(1) };
    // This is how we basically check for errors and handle them using most 
    // C APIs
    // We handle them by just panicing here in our example.
    if queue < 0 {
        panic!(io::Error::last_os_error());
    }

    // As you'll see below, we need a place to store the streams so they're 
    // not closed
    let mut streams = vec![];

    // We crate 5 requests to an an endpoint we control the delay on
    for i in 1..6 {
        // This site has an api to simulate slow responses from a server
        let addr = "slowwly.robertomurray.co.uk:80";
        let mut stream = TcpStream::connect(addr).unwrap();

        // The delay is passed in to the GET request as milliseconds. 
        // We'll create delays in decending order so we sould recieve 
        // them as `5, 4, 3, 2, 1`
        let delay = (5 - i) * 1000;
        let request = format!(
            "GET /delay/{}/url/http://www.google.com HTTP/1.1\r\n\
             Host: slowwly.robertomurray.co.uk\r\n\
             Connection: close\r\n\
             \r\n",
            delay
        );
        stream.write_all(request.as_bytes()).unwrap();

        // make this socket non-blocking. Well, not really needed since 
        // we're not using it in this example...
        stream.set_nonblocking(true).unwrap();

        // Then register interest in getting notified for `Read` events on 
        // this socket. The `Event` struct is where we specify what events 
        // we want to register interest in and other configurations using 
        // flags. 
        //
        // `EPOLLIN` is interest in `Read` events. 
        // `EPOLLONESHOT` means that we remove any interests from the queue 
        // after first event. If we don't do that we need to `deregister` 
        // our interest manually when we're done with the socket.
        //
        // `epoll_data` is user provided data, so we can put a pointer or 
        // an integer value there to identify the event. We just use 
        // `i` which is the loop count to indentify the events.
        let mut event = ffi::Event {
            events: (ffi::EPOLLIN | ffi::EPOLLONESHOT) as u32,
            epoll_data: i,
        };

        // This is the call where we actually `ADD` an interest to our queue. 
        // `EPOLL_CTL_ADD` is the flag which controls whether we want to 
        // add interest, modify an existing one or remove interests from 
        // the queue.
        let op = ffi::EPOLL_CTL_ADD;
        let res = unsafe { 
            ffi::epoll_ctl(queue, op, stream.as_raw_fd(), &mut event) 
        };
        if res < 0 {
            panic!(io::Error::last_os_error());
        }

        // Letting `stream` go out of scope in Rust automatically runs 
        // its destructor which closes the socket. We prevent that by 
        // holding on to it until we're finished
        streams.push(stream);
        event_counter += 1;
    }

    // Now we wait for events
    while event_counter > 0 {

        // The API expects us to pass in an arary of `Event` structs. 
        // This is how the OS communicates back to us what has happened.
        let mut events = Vec::with_capacity(10);

        // This call will actually block until an event occurs. The timeout 
        // of `-1` means no timeout so we'll block until something happens. 
        // Now the OS suspends our thread doing a context switch and work 
        // on someting else - or just perserve power.
        let res = unsafe { ffi::epoll_wait(queue, events.as_mut_ptr(), 10, -1) };
        // This result will return the number of events which occurred 
        // (if any) or a negative number if it's an error.
        if res < 0 {
            panic!(io::Error::last_os_error());
        };

        // This one unsafe we could avoid though but this technique is used 
        // in libraries like `mio` and is safe as long as the OS does 
        // what it's supposed to.
        unsafe { events.set_len(res as usize) };

        for event in events {
            println!("RECIEVED: {:?}", event);
            event_counter -= 1;
        }
    }

    // When we manually initialize resources we need to manually clean up 
    // after our selves as well. Normally, in Rust, there will be a `Drop` 
    // implementation which takes care of this for us.
    let res = unsafe { ffi::close(queue) };
    if res < 0 {
        panic!(io::Error::last_os_error());
    }
    println!("FINISHED");
}

mod ffi {
    pub const EPOLL_CTL_ADD: i32 = 1;
    pub const EPOLLIN: i32 = 0x1;
    pub const EPOLLONESHOT: i32 = 0x40000000;

    #[link(name = "c")]
    extern "C" {
        pub fn epoll_create(size: i32) -> i32;
        pub fn close(fd: i32) -> i32;
        pub fn epoll_ctl(epfd: i32, op: i32, fd: i32, event: *mut Event) -> i32;
        pub fn epoll_wait(epfd: i32, events: *mut Event, maxevents: i32, timeout: i32) -> i32;
    }

    #[derive(Debug)]
    #[repr(C)]
    pub struct Event {
        pub(crate) events: u32,
        pub(crate) epoll_data: usize,
    }
}
```

{% hint style="info" %}
If [bitflags](../appendix-1/bitflags.md) are new to you and you want to know what `ffi::EPOLLIN | ffi::EPOLLONESHOT`does and why it's done that way. Take a look at the [Bitflags](../appendix-1/bitflags.md) chapter in the appendix.
{% endhint %}

Now, I've commented the code to the best of my ability to answer any questions along the way, so I won't repeat that here.

If you have any questions, you can use the [Issue Tracker for this book](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/issues) and ask there. Unless you read this many years after I wrote it, chances are that I, or someone else, can answer you there.

Good job! We have actually created our own epoll-backed event queue which notifies us on read events on our sockets!

If we run the code we get:

```text
RECIEVED: Event { events: 1, epoll_data: 140587164499969 }
RECIEVED: Event { events: 1, epoll_data: 140587164499969 }
RECIEVED: Event { events: 1, epoll_data: 140587164499969 }
RECIEVED: Event { events: 1, epoll_data: 140587164499969 }
RECIEVED: Event { events: 1, epoll_data: 140587164499969 }
FINISHED
```

**Wait! What?** Our `epoll_data`is not 5, 4, 3, 2, 1 as expected but something else entirely? 

Oh, so you trusted the manpage for Linux, did you? Yeah, me too. It turns out it's written for users of the C library and not with people using `ffi`in mind. A valuable lesson to keep in mind. In this case that causes a big problem for us.

{% hint style="info" %}
After a little bit of searching \(well, to be honest it was a lot of searching\) I found out that the manpage doesn't tell the whole truth. The real definition looks like this _\(thanks to user @Talchas on the Rust discord channel for figuring this out\)_:
{% endhint %}

```c
struct epoll_event {
uint32_t events;    /* Epoll events */
epoll_data_t data;  /* User data variable */
} __EPOLL_PACKED;
```

The `__EPOLL_PACKED` directive was not in the manpage. This means that the struct is not padded which would normally be the case when declaring a 32-bit sized data type before a 64-bit sized datatype. The first 4 bytes of our `epoll_data`is written to the padding between our `u32`and our `u64`\(which is what an `usize`is on 64 bit systems\).

Fortunately FFI in Rust is really pleasant to work with, and we only need to make one small change to fix this:

```rust
#[derive(Debug, Clone, Copy)]
#[repr(C, packed)]
pub struct Event {
    pub(crate) events: u32,
    pub(crate) epoll_data: usize,
}
```

Notice the `#[repr(C, packed)]`attribute? This tells the Rust compiler to treat this as a packed struct which is what we need. We also need to derive `Clone`and `Copy`to be able to safely create a `Debug` display of a packed struct.

**Running our example again gives us what we expected:**

```text
RECIEVED: Event { events: 1, epoll_data: 5 }
RECIEVED: Event { events: 1, epoll_data: 4 }
RECIEVED: Event { events: 1, epoll_data: 3 }
RECIEVED: Event { events: 1, epoll_data: 2 }
RECIEVED: Event { events: 1, epoll_data: 1 }
FINISHED
```

Now that we have seen how `epoll`works in real life, let's move on and have a look at `kqueue`and `IOCP`so we'll learn the basics of how they work as well.


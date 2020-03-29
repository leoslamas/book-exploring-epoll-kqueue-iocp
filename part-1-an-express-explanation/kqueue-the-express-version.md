# Kqueue - the express version

Like in the express epoll example we initialize a cargo project and open `main.rs`.

Kqueue has a rather small API surface with only two syscalls to deal with `kqueue`and one for closing the queue. The API looks like this:

```rust
mod ffi {
    #[link(name = "c")]
    extern "C" {
        /// Returns: positive: file descriptor, negative: error
        pub(super) fn kqueue() -> i32;
        
        pub(super) fn kevent(
            kq: i32,
            changelist: *const Kevent,
            nchanges: i32,
            eventlist: *mut Kevent,
            nevents: i32,
            timeout: *const Timespec,
        ) -> i32;

        pub fn close(d: i32) -> i32;
    }
}
```

Most of our interactions with our `kqueue`instance happens through the `Kevent`struct. In this struct we pass in information about whether we want to `add`, `modify`or `delete`an interest from our queue. We also specify what kind of events we're interested in. Let's have a look at the other structures and constants in our `ffi`module:

```rust
pub const EVFILT_READ: i16 = -1;
pub const EV_ADD: u16 = 0x1;
pub const EV_ENABLE: u16 = 0x4;
pub const EV_ONESHOT: u16 = 0x10;

#[derive(Debug)]
#[repr(C)]
pub(super) struct Timespec {
    /// Seconds
    tv_sec: isize,
    /// Nanoseconds     
    v_nsec: usize,
}

impl Timespec {
    pub fn from_millis(milliseconds: i32) -> Self {
        let seconds = milliseconds / 1000;
        let nanoseconds = (milliseconds % 1000) * 1000 * 1000;
        Timespec {
            tv_sec: seconds as isize,
            v_nsec: nanoseconds as usize,
        }
    }
}

// https://github.com/rust-lang/libc/blob/c8aa8ec72d631bc35099bcf5d634cf0a0b841be0/src/unix/bsd/apple/mod.rs#L497
// https://github.com/rust-lang/libc/blob/c8aa8ec72d631bc35099bcf5d634cf0a0b841be0/src/unix/bsd/apple/mod.rs#L207
#[derive(Debug, Clone, Default)]
#[repr(C)]
pub struct Kevent {
    pub ident: u64,
    pub filter: i16,
    pub flags: u16,
    pub fflags: u32,
    pub data: i64,
    pub udata: u64,
}
```

Here we see the definition of the `Kevent`structure. If we look at the manpage for `kevent`we find some more information about this structure:

![Click to enlarge](../.gitbook/assets/bilde%20%2813%29.png)

You can find more information on the [macos manpage for `kevent`](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/kevent.2.html). One thing to note is that the `udata`field can be used to store any user-defined data. The OS leaves this untouched so it doesn't need to be a valid pointer. In our example code we use a regular `usize`to identify each event.

Let's put this all together and see how these syscalls are used:

```rust
use std::io::{self, Write};
use std::os::unix::io::AsRawFd;
use std::net::TcpStream;
use std::ptr;

fn main() {
    // A counter to keep track of how many events we're expecting to act on
    let mut event_counter = 0;

    // First we create the event queue.
    // The size argument is ignored but needs to be larger than 0
    let queue = unsafe { ffi::kqueue() };
    // We handle errors in this example by just panicking.
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
        // We'll create delays in decending order so we sould receive 
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
        // this socket. The `Kevent` struct is where we specify what events 
        // we want to register interest in and other configurations using 
        // flags. 
        //
        // `EVFILT_READ` indicates that this is a `Read` interest
        // `EV_ADD` indicates that we're adding a new event to the queue. 
        // `EV_ENABLE` means that we want the event returned when triggered
        // `EV_ONESHOT` mans that we want the vent deleted from the queue
        // on the first occurance. If we don't do that we need to `deregister` 
        // our interest manually when we're done with the socket (which is fine
        // but for this example it's easier to just delete it first time)
        //
        // You can read more about the flags and options here:
        // https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/kevent.2.html

        let event = ffi::Kevent {
                ident: stream.as_raw_fd() as u64,
                filter: ffi::EVFILT_READ,
                flags: ffi::EV_ADD | ffi::EV_ENABLE | ffi::EV_ONESHOT,
                fflags: 0,
                data: 0,
                udata: i,
        };

        let changelist = [event];

        // This is the call where we actually register an interest with our 
        // queue. The call to `kevent` behaves differently based on the 
        // parameters passed in. Passing in a null pointer as the timeout 
        // specifies an infinite timeout
        let res = unsafe {ffi::kevent(
            queue,               // the kqueue handle
            changelist.as_ptr(), // changes to the queue
            1,                   // length of changelist 
            ptr::null_mut(),     // event list (if we expect any returned)
            0,                   // len of event list (if we expect any)
            ptr::null()          // timeout (if any)
        )};

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

        // The API expects us to pass in an arary of `Kevent` structs. 
        // This is how the OS communicates back to us what has happened.
        let mut events: Vec<ffi::Kevent> = Vec::with_capacity(10);

        // This call will actually block until an event occurs. Passing in a
        // null pointer as the timeout waits indefinately
        // Now the OS suspends our thread doing a context switch and work 
        // on someting else - or just perserve power.
        let res = unsafe { 
            ffi::kevent(
                queue,                    // same kqueue
                ptr::null(),              // no changes this time
                0,                        // length of change array is 0
                events.as_mut_ptr(),      // we expect to get events back
                events.capacity() as i32, // how many events we can receive
                ptr::null(),              // indefinite timeout
            )
        };

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
            println!("RECEIVED: {}", event.udata);
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
    pub const EVFILT_READ: i16 = -1;
    pub const EV_ADD: u16 = 0x1;
    pub const EV_ENABLE: u16 = 0x4;
    pub const EV_ONESHOT: u16 = 0x10;

    #[derive(Debug)]
    #[repr(C)]
    pub(super) struct Timespec {
        /// Seconds
        tv_sec: isize,
        /// Nanoseconds     
        v_nsec: usize,
    }

    impl Timespec {
        pub fn from_millis(milliseconds: i32) -> Self {
            let seconds = milliseconds / 1000;
            let nanoseconds = (milliseconds % 1000) * 1000 * 1000;
            Timespec {
                tv_sec: seconds as isize,
                v_nsec: nanoseconds as usize,
            }
        }
    }

    // https://github.com/rust-lang/libc/blob/c8aa8ec72d631bc35099bcf5d634cf0a0b841be0/src/unix/bsd/apple/mod.rs#L497
    // https://github.com/rust-lang/libc/blob/c8aa8ec72d631bc35099bcf5d634cf0a0b841be0/src/unix/bsd/apple/mod.rs#L207
    #[derive(Debug, Clone, Default)]
    #[repr(C)]
    pub struct Kevent {
        pub ident: u64,
        pub filter: i16,
        pub flags: u16,
        pub fflags: u32,
        pub data: i64,
        pub udata: u64,
    }

    #[link(name = "c")]
    extern "C" {
        /// Returns: positive: file descriptor, negative: error
        pub(super) fn kqueue() -> i32;
        
        pub(super) fn kevent(
            kq: i32,
            changelist: *const Kevent,
            nchanges: i32,
            eventlist: *mut Kevent,
            nevents: i32,
            timeout: *const Timespec,
        ) -> i32;

        pub fn close(d: i32) -> i32;
    }
}
```

I've chosen to comment everything in the code so if you paste this into your own program you'll still have all information you need.

There are however something I want to point out specifically. The call to `kevent`behaves differently from the calls we see in `epoll`in that it does different things based on what arguments are passed in. 

A simplified way of thinking of this is that it has two modes. One is `register`mode, where we pass in a set of changes we want to make to our queue in the `changelist`field. 

The other is the `wait`mode where it will request the OS to suspend the thread it's called from and wake it up when some event\(s\) has happened. In this call we pass in an array of zeroed `Kevent`structs which the OS will fill with data about what has occurred while the thread was suspended.

{% hint style="info" %}
The fact that we can make multiple changes to the queue using only a single syscall can be an advantage in situations where you have many I/O operations and many calls. A syscall is not cheap and minimizing them can have a noticable effect on performance.
{% endhint %}

**Running this code on a system running `macos`should give the following result:**

```text
RECEIVED: 4
RECEIVED: 5
RECEIVED: 3
RECEIVED: 2
RECEIVED: 1
FINISHED
```

If you want to see what the `Kevent`structure we get in return looks like, change line 130 to `println!("RECEIVED: {:?}", event);`, and you'll notice that the OS has filled in the field with data about the event which occurred.


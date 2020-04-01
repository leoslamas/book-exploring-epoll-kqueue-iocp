# IOCP

{% hint style="info" %}
We're starting off with Windows and IOCP since, well, I did it the other way around on my first go at this project and that turned out to require a total rewrite when I tried to fit the completion based `IOCP` model to the readiness based models like `kqueue`and `epoll`.

It's also by far going to be the implementation requiring most lines of code and is arguably the most difficult to implement.
{% endhint %}

Let's move in to the `windows.rs`file in our project.

To implement an IOCP backed event queue you need to interact with the operating system through a set of syscalls. Let's create a sub module in the `windows.rs`file to contain these calls.

## FFI module

```rust
mod ffi {
    ...
}
```

Inside this sub module, let's define the `extern` functions we're going to call to make these syscalls:

```rust
#[link(name = "Kernel32")]
extern "stdcall" {
    // https://docs.microsoft.com/en-us/windows/win32/fileio/createiocompletionport
    fn CreateIoCompletionPort(
        filehandle: HANDLE,
        existing_completionport: HANDLE,
        completion_key: ULONG_PTR,
        number_of_concurrent_threads: DWORD,
    ) -> HANDLE;

    // https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsarecv
    fn WSARecv(
        s: RawSocket,
        lpBuffers: LPWSABUF,
        dwBufferCount: DWORD,
        lpNumberOfBytesRecvd: LPDWORD,
        lpFlags: LPDWORD,
        lpOverlapped: LPWSAOVERLAPPED,
        lpCompletionRoutine: LPWSAOVERLAPPED_COMPLETION_ROUTINE,
    ) -> i32;

    // https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus
    fn PostQueuedCompletionStatus(
        CompletionPort: HANDLE,
        dwNumberOfBytesTransferred: DWORD,
        dwCompletionKey: ULONG_PTR,
        lpOverlapped: LPWSAOVERLAPPED,
    ) -> i32;

    /// https://docs.microsoft.com/nb-no/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus
    /// Errors: https://docs.microsoft.com/nb-no/windows/win32/debug/system-error-codes--0-499-
    /// From this we can see that error `WAIT_TIMEOUT` has the code 258 which we'll
    /// need later on
    fn GetQueuedCompletionStatusEx(
        CompletionPort: HANDLE,
        lpCompletionPortEntries: *mut OVERLAPPED_ENTRY,
        ulCount: ULONG,
        ulNumEntriesRemoved: PULONG,
        dwMilliseconds: DWORD,
        fAlertable: BOOL,
    ) -> i32;

    // https://docs.microsoft.com/nb-no/windows/win32/api/handleapi/nf-handleapi-closehandle
    fn CloseHandle(hObject: HANDLE) -> i32;

    // https://docs.microsoft.com/nb-no/windows/win32/api/winsock/nf-winsock-wsagetlasterror
    fn WSAGetLastError() -> i32;
}
```

So this is the absolute minimum we need to implement a functioning `IOCP`event queue. I have linked to the documentation for each function, but I'll write a very short explanation of what each function does and why we need it.

{% hint style="info" %}
In Rust, all `ffi`calls are `unsafe`. It's a common pattern to create safe wrappers around these calls, so for each function below I 'll also show the safe wrapper we write for each function. Both the `ffi`definitions and the wrappers live in the `ffi`module.
{% endhint %}

### CreateIoCompletionPort

Creates an event queue, or a `CompletionPort`which it is called on Windows. This will be the first thing we call when we want to start an IOCP event queue.

The safe wrapper around this call looks like this:

```rust
/// Returns the file handle to the completion port we passed in
pub fn create_io_completion_port(
    s: RawSocket,
    completion_port: isize,
    token: usize,
) -> io::Result<isize> {
    let res = unsafe { 
        CreateIoCompletionPort(
            s as isize, 
            completion_port, 
            token as *mut usize, 
            0,
        )};

    if (res as *mut usize).is_null() {
        return Err(std::io::Error::last_os_error());
    }

    Ok(res)
}
```

### WSARecv

This method will register an interest in a `data received`event. Together with the [WSASend](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasend) method this lets us wait for completed `Read`and `Write`operations. This will be the method we call to indicate to the operating system that we're interested to get a notification when the `Receive`event has finished on our `Socket`. We call this from our `Registrator`.

Windows, calls an operation we can await in an event queue an `Overlapped Operation`. Knowing this is handy when it comes to understanding the names of data structures and functions on Windows.

{% hint style="warning" %}
This function can return `0`, which means no error occurred and that the `Receive`operation completed immediately. **This is however not what we expect this function to return!** 

So you thought the `Success`case was what we're expecting, yes? In this specific case we expect failure! 

In we get a result indicating success there is nothing for us to await and we should handle the case when data is already there \(we'll not cover this case in our implementation though\).

Turns out that we actually expect to get an error of the type [WSA\_IO\_PENDING](https://docs.microsoft.com/windows/desktop/WinSock/windows-sockets-error-codes-2) which has the error code `997`. This error means that the overlapped operation has been successfully initiated and that completion will be indicated at a later time.

Any other errors indicate an unsuccessful registration.

By the way, you need to call the function `WSAGetLastError`to get the error code which I explain further below.
{% endhint %}

**Safe wrapper:**

```rust
/// Creates a socket read event.
/// ## Returns
/// The number of bytes received
pub fn wsa_recv(
    s: RawSocket,
    wsabuffers: &mut [WSABUF],
    op: &mut Operation,
) -> Result<(), io::Error> {
    let mut flags = 0;
    let operation_ptr: *mut Operation = op;

    let res = unsafe {
        WSARecv(
            s,
            wsabuffers.as_mut_ptr(),
            1,
            ptr::null_mut(),
            &mut flags,
            operation_ptr as *mut WSAOVERLAPPED,
            ptr::null_mut(),
        )
    };
    if res != 0 {
        let err = unsafe { WSAGetLastError() };
        if err == WSA_IO_PENDING {
            // Everything is OK. We can wait this with GetQueuedCompletionStatus
            Ok(())
        } else {
            Err(std::io::Error::last_os_error())
        }
    } else {
        // The socket is already ready so we don't need to queue it
        // TODO: Avoid queueing this
        Ok(())
    }
}
```

{% hint style="success" %}
**The safe wrapper deserves some commenting. Do you see the cast `operation_ptr as *mut WSAOVERLAPPED`?** 

This is explained below, but it's a way for us to actually identify what event occurred. The way we do this is by wrapping `WSAOVERLAPPED`in an `Operation`struct. 

If we make sure that the `Operation`structs memory layout conforms with what the API expects we can attach additional context with the event by casting it to a `WSAOVERLAPPED`pointer and then later on dereference it back to an `Operation`struct and retrieve any additional information we attached identifying the event that occurred. It's complicated, but I'll explain more as we go along.

This technique is inspired by the [BOOST ASIO implementation of IOCP](https://www.boost.org/doc/libs/1_42_0/boost/asio/detail/win_iocp_io_service.hpp).
{% endhint %}

### PostQueuedCompletionStatus

This lets us post an event to our event queue. We only need this to register an event which lets us wake up the thread waiting for events to close it down.

**Safe wrapper:**

```rust
pub fn post_queued_completion_status(
    completion_port: isize,
    bytes_to_transfer: u32,
    completion_key: usize,
    overlapped_ptr: &mut WSAOVERLAPPED,
) -> io::Result<()> {

    let res = unsafe {
        PostQueuedCompletionStatus(
            completion_port,
            bytes_to_transfer,
            completion_key as *mut usize,
            overlapped_ptr,
        )
    };
    
    if res == 0 {
        Err(std::io::Error::last_os_error().into())
    } else {
        Ok(())
    }
}
```

### GetQueuedCompletionStatusEx

Ok, so this is the actually blocking call which waits for new events. There are two related functions called `GetQueuedCompletionStatusEx`and `GetQueuedCompletionStatus`. The difference between them is that `GetQueuedCompletionStatus`receives one and one event, while the `...Ex`version can receive multiple events simultaneously.

One thing to note is that we need to pass in a pre-allocated `OVERLAPPED_ENTRY`structures. This array will be filled with entries corresponding to an event.

The safe wrapper is pretty extensively documented so there is a lot of information on how I made this work.

**Safe wrapper:**

```rust
/// ## Parameters:
/// - *completion_port:* the handle to a completion port created by 
/// callingCreateIoCompletionPort
///
/// - *completion_port_entries:* a pointer to an array of 
/// OVERLAPPED_ENTRY structures
/// - *ul_count:* The maximum number of entries to remove
/// - *timeout:* The timeout in milliseconds, if set to NONE, 
/// timeout is set to INFINITE
/// - *alertable:* If this parameter is FALSE, the function does not 
/// return until the time-outperiod has elapsed or an entry is retrieved. 
/// If the parameter is TRUE and there are no available entries, the 
/// function performs an alertable wait. The thread returns when the 
/// system queues an I/O completion routine or APC to the thread and the 
/// thread executes the function.
///
/// ## Returns
/// The number of items actually removed from the queue
pub fn get_queued_completion_status_ex(
    completion_port: isize,
    completion_port_entries: &mut [OVERLAPPED_ENTRY],
    ul_count: u32,
    timeout: Option<u32>,
    alertable: bool,
) -> io::Result<u32> {

    let mut ul_num_entries_removed: u32 = 0;
    let timeout = timeout.unwrap_or(INFINITE);
    
    let res = unsafe {
        GetQueuedCompletionStatusEx(
            completion_port,
            completion_port_entries.as_mut_ptr(),
            ul_count,
            &mut ul_num_entries_removed,
            timeout,
            alertable,
        )
    };

    if res == 0 {
        Err(io::Error::last_os_error())
    } else {
        Ok(ul_num_entries_removed)
    }
}
```

### CloseHandle

This function simply closes a handle. Since we're communicating directly with the OS here, there is nobody that helps us clean up after us and release resources. We need this function to properly close our `IOCP`queue.

**Safe wrapper:**

```rust
pub fn close_handle(handle: isize) -> io::Result<()> {
    let res = unsafe { CloseHandle(handle) };

    if res == 0 {
        Err(std::io::Error::last_os_error().into())
    } else {
        Ok(())
    }
}
```

### WSAGetLastError

This function retrieves the last error we got. Normally it's enough to check if there was an error and use `std::io::Error::last_os_error()`from Rust's standard library to retrieve the error. However, in our `WSARecv`function we need to retrieve information about the error and get the correct error code.

We'll not create a safe wrapper around this since we only call this function in our other safe wrappers and never outside the `ffi`module.

## Structures and Constants:

We need to define some structures and some constants which are missing from our `ffi`module inside `windows.rs`so far. I'll present more constants than we actually use, but if you want to play around with this they might come in handy.

```rust
#[repr(C)]
#[derive(Clone, Debug)]
pub struct WSABUF {
    len: u32,
    buf: *mut u8,
}

impl WSABUF {
    pub fn new(len: u32, buf: *mut u8) -> Self {
        WSABUF { len, buf }
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct OVERLAPPED_ENTRY {
    lp_completion_key: *mut usize,
    lp_overlapped: *mut WSAOVERLAPPED,
    internal: usize,
    bytes_transferred: u32,
}

impl OVERLAPPED_ENTRY {
    pub fn id(&self) -> Token {
        // NOTE: this might be solvable wihtout sacrifising so 
        // much of Rusts safety guarantees
        let operation: &Operation = unsafe { &*(self.lp_overlapped as *const Operation) };
        operation.token
    }

    pub(crate) fn zeroed() -> Self {
        OVERLAPPED_ENTRY {
            lp_completion_key: ptr::null_mut(),
            lp_overlapped: ptr::null_mut(),
            internal: 0,
            bytes_transferred: 0,
        }
    }
}

// Reference: https://docs.microsoft.com/en-us/windows/win32/api/winsock2/ns-winsock2-wsaoverlapped
#[repr(C)]
#[derive(Debug)]
pub struct WSAOVERLAPPED {
    /// Reserved for internal use
    internal: ULONG_PTR,
    /// Reserved
    internal_high: ULONG_PTR,
    /// Reserved for service providers
    offset: DWORD,
    /// Reserved for service providers
    offset_high: DWORD,
    /// If an overlapped I/O operation is issued without an I/O completion routine
    /// (the operation's lpCompletionRoutine parameter is set to null), then this 
    /// parameter should either contain a valid handle to a WSAEVENT object or 
    /// be null. If the lpCompletionRoutine parameter of the call is non-null then 
    /// applications are free to use this parameter as necessary.
    h_event: HANDLE,
}

impl WSAOVERLAPPED {
    pub fn zeroed() -> Self {
        WSAOVERLAPPED {
            internal: ptr::null_mut(),
            internal_high: ptr::null_mut(),
            offset: 0,
            offset_high: 0,
            h_event: 0,
        }
    }
}

/// Operation is a way for us to attach additional context to the `WSAOVERLAPPED`
/// event. Inpired by [BOOST ASIO](https://www.boost.org/doc/libs/1_42_0/boost/asio/detail/win_iocp_io_service.hpp)
#[derive(Debug)]
#[repr(C)]
pub struct Operation {
    wsaoverlapped: WSAOVERLAPPED,
    token: usize,
}

impl Operation {
    pub(crate) fn new(token: usize) -> Self {
        Operation {
            wsaoverlapped: WSAOVERLAPPED::zeroed(),
            token,
        }
    }
}

// You can find most of these here: https://docs.microsoft.com/en-us/windows/win32/winprog/windows-data-types
/// The HANDLE type is actually a `*mut c_void` but windows preserves 
/// backwards compatibility by allowing a INVALID_HANDLE_VALUE which is `-1`. 
/// We can't express that in Rust so it's much easier for us to treat
/// this as an isize instead;
pub type HANDLE = isize;
pub type BOOL = bool;
pub type WORD = u16;
pub type DWORD = u32;
pub type ULONG = u32;
pub type PULONG = *mut ULONG;
pub type ULONG_PTR = *mut usize;
pub type PULONG_PTR = *mut ULONG_PTR;
pub type LPDWORD = *mut DWORD;
pub type LPWSABUF = *mut WSABUF;
pub type LPWSAOVERLAPPED = *mut WSAOVERLAPPED;
pub type LPWSAOVERLAPPED_COMPLETION_ROUTINE = *const extern "C" fn();

// https://referencesource.microsoft.com/#System.Runtime.Remoting/channels/ipc/win32namedpipes.cs,edc09ced20442fea,references
// read this! https://devblogs.microsoft.com/oldnewthing/20040302-00/?p=40443
/// Defined in `win32.h` which you can find on your windows system
pub const INVALID_HANDLE_VALUE: HANDLE = -1;

// https://docs.microsoft.com/en-us/windows/win32/winsock/windows-sockets-error-codes-2
pub const WSA_IO_PENDING: i32 = 997;

// This can also be written as `4294967295` if you look at sources on the internet.
// Interpreted as an i32 the value is -1
// see for yourself: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4b93de7d7eb43fa9cd7f5b60933d8935
pub const INFINITE: u32 = 0xFFFFFFFF;
```

{% hint style="info" %}
It's worth noting that the `OVERLAPPED` structure comes in two flavors: `OVERLAPPED`and `WSAOVERLAPPED`. Windows defines the `WSAOVERLAPPED`structure as equivalent to the Win32 `OVERLAPPED`structure \([reference](https://msdn.microsoft.com/en-us/ie/ff565952%28v=vs.94%29)\). This means that even though WinAPI sometimes defines a `OVERLAPPED`structure you can safely use the `WSAOVERLAPPED`structure instead.

We can avoid having to deal with two different structures and simplify our code a bit by using this information to only use `WSAOVERLAPPED`when dealing with this API.
{% endhint %}

## Event, Registrator, Selector and TcpStream

Now that we have wired up our `ffi`module it's time to implement the public API of the Windows module.

### The Event struct

We start of easy. Event is just a type alias for our `OVERLAPPED_ENTRY`struct. We have implemented one public methods on this: `fn id(&self) -> Token`and a private one `zeroed()`which just zero initializes all the fields.

```rust
pub type Event = ffi::OVERLAPPED_ENTRY;
```

### The TcpStream

The `TcpStream`is where we abstract over the many differences between `IOCP`and `kqueue/epoll`. On Windows we need to lend a buffer \(or array of buffers to be precise\) to the OS which it will use to read the data into.

{% hint style="warning" %}
Since `CompletionKey`is assigned on a _per resource basis_ there is no easy way for us to get information about what exact event occurred. 

We solve this by providing a struct which has t**he same memory layout** as `WSAOVERLAPPED`which is passed into the syscall that registers an event. If you look at the definition of the `ffi::Operation`struct it has a `#[repr(C)]`directive and the first field is a `ffi::WSAOVERLAPPED`struct. This means that if we cast this to a `*mut WSAOVERLAPPED`and pass it to the Windows API it will only touch this field of our `Operation`struct.

This property is usefull since we can then attach an arbitrary amount of context after the `wsaoverlapped`field which identifies this struct. When the event has occurred, we can cast it back into a `Operation`struct to get this additional information.

In our code we do this as a part of `OVERLAPPED_ENTRY` which has a pointer to the `WSAOVERLAPPED`struct we passed in when we registered the event. You can see this in the [Structures and Constants](iocp.md#structures-and-constants) section above.

We used a `LinkedList`to store these operations. The reason for preferring this over a `Vec`is that we could end up reallocating the `Vec`if we register many events on this resource thereby invalidating the address to the `Operation`struct we handed over to Windows. A linked list will not reallocate and is a suitable collection to use for this scenario.
{% endhint %}

To be able to use this socket as a normal `std::net::TcpStream`we implement the `Read`, `Write`and `AsRawSocket`traits. The interesting part is where we "hijack" the `read`method and read from the buffer we sent to `IOCP`instead.

If we had implemented `WSASend`we'd have to do the same for the `Write`methods.

```rust
#[derive(Debug)]
pub struct TcpStream {
    inner: net::TcpStream,
    buffer: Vec<u8>,
    wsabuf: Vec<ffi::WSABUF>,
    event: Option<ffi::WSAOVERLAPPED>,
    token: Option<usize>,
    pos: usize,
    operations: LinkedList<ffi::Operation>,
}

// On Windows we need to be careful when using IOCP on a server. Since we're 
//"lending" access to the OS over memory we crate (we're not giving over 
// ownership, but can't touch while it's borrowed either).

// It's easy to exploit this by issuing a lot of requests while delaying our
// responses. By doing this we would force the server to hand over so many 
// write/read buffers while waiting for clients to respond that it might run 
// out of memory. Now the way we would normally handle this is to have a counter
// and limit the number of outstandig buffers, queueing requests and only handle 
// them when the counter is below the high water mark. The same goes for using 
// unlimited timeouts. 
// http://www.serverframework.com/asynchronousevents/2011/06/tcp-flow-control-and-asynchronous-writes.html
impl TcpStream {
    pub fn connect(adr: impl net::ToSocketAddrs) -> io::Result<Self> {
        // This is a shortcut since this will block when establishing the connection.
        // There are several ways of avoiding this.
        // a) Obtrain the socket using system calls, set it to non_blocking before we connect
        // b) use the crate [net2](https://docs.rs/net2/0.2.33/net2/index.html) which
        // defines a trait with default implementation for TcpStream which allow us to set
        // it to non-blocking before we connect

        // Rust creates a WSASocket set to overlapped by default which is just what we need
        // https://github.com/rust-lang/rust/blob/f86521e0a33a2b54c4c23dbfc5250013f7a33b11/src/libstd/sys/windows/net.rs#L99
        let stream = net::TcpStream::connect(adr)?;
        stream.set_nonblocking(true)?;

        let mut buffer = vec![0_u8; 1024];
        let wsabuf = vec![ffi::WSABUF::new(buffer.len() as u32, buffer.as_mut_ptr())];
        Ok(TcpStream {
            inner: stream,
            buffer,
            wsabuf,
            event: None,
            token: None,
            pos: 0,
            operations: LinkedList::new(),
        })
    }
}

impl Read for TcpStream {
    fn read(&mut self, buff: &mut [u8]) -> io::Result<usize> {
        //   self.inner.read(buff)
        let mut bytes_read = 0;
        if self.buffer.len() - bytes_read <= buff.len() {
            for (a, b) in self.buffer.iter().skip(self.pos).zip(buff) {
                *b = *a;
                bytes_read += 1;
            }

            Ok(bytes_read)
        } else {
            for (b, a) in buff.iter_mut().zip(&self.buffer) {
                *b = *a;
                bytes_read += 1;
            }
            self.pos += bytes_read;
            Ok(bytes_read)
        }
    }
}

impl Write for TcpStream {
    fn write(&mut self, buff: &[u8]) -> io::Result<usize> {
        self.inner.write(buff)
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}

impl AsRawSocket for TcpStream {
    fn as_raw_socket(&self) -> RawSocket {
        self.inner.as_raw_socket()
    }
}
```

### The Registrator

The `registrator` needs to be "linked" to our completion port, and should be able to check if the event loop is alive or not so we don't use the `Registrator`after we sent a `close_signal`to our event loop.

We use an `Arc<AtomicBool>`to check if the `Poll`instance is alive or not. The "link" to the completion port is simply the completion port handle.

`Registrator` has two public methods: `register`and `close_loop`. 

`register` is interesting. It takes a `TcpStream`\(not the `std::net`one though\), a token and a bitflag indicating what interests we're interested in.

Now, the first thing we do is to check if the `Poll`instance is dead.

{% hint style="danger" %}
Warning! Remember that the `Registrator`is meant to just be used on **one** different thread from the one that our `Poll`instance is running on. Read the [relevant paragraph in the Appendix chapter "It's not that easy"](its-not-that-easy.md#thread-safety-and-race-conditions) for more information about this and potential ways to solve this limitation.
{% endhint %}

The next thing we do is to associate the "resource", in this case a `socket`with our `IOCP`instance. 

{% hint style="info" %}
This might seem strange but `create_io_completion_port`behaves diffrently based on the parameters we pass in. If we pass in both a valid `handle`and a valid `completion_port`we have now associated this `handle`with this `completion_port`. If we choose to pass in a`CompletionKey`it is also associated with this `handle`. We only pass in `0`here since we don't use the `CompletionKey`.
{% endhint %}

```rust
pub struct Registrator {
    completion_port: isize,
    is_poll_dead: Arc<AtomicBool>,
}

impl Registrator {
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

        // NOTE: We associate the same resource over and over here. Every time
        // we do that is a syscall we could avoid. The proper thing to do is to
        // implement some logic that makes sure thois is only set once
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

    /// NOTE: An alternative solution is to use the `CompletionKey` to signal that
    /// this is a close event. We don't use it for anything else so it is a
    /// good candidate to use for timers and special events like this
    pub fn close_loop(&self) -> io::Result<()> {
        if self
            .is_poll_dead
            .compare_and_swap(false, true, Ordering::SeqCst)
        {
            return Err(io::Error::new(
                io::ErrorKind::Interrupted,
                "Poll instance is dead.",
            ));
        }
        let mut overlapped = ffi::WSAOVERLAPPED::zeroed();
        ffi::post_queued_completion_status(self.completion_port, 0, 0, &mut overlapped)?;
        Ok(())
    }
}
```

{% hint style="danger" %}
We require a `&mut TcpStram` here, but we only need it for Windows. 

Now there are ways to deal with this so we don't require an API which needs `&mut TcpStream`, on the platforms where we don't pass in a buffer to the OS, but either way the reality is that we mutate data owned by our `TcpSocket`on Windows. 

If we implement this as part of a Runtime which we control there are ways for/ us to guarantee that the buffer is not touched or moved, so we could mutate it safely, but this is not a Runtime so we can't know that.
{% endhint %}

One interesting thing to note here is that in our `close_loop` function we need to wake the thread which is now suspended waiting for events. On Windows we do this by posting en empty completion packet using `PostQueuedCompletionStatus`, which will wake up our thread immediately. Since we get no events in return, we just continue until we check the flag which indicates that our instance is closed and exits. On the other platforms we use different methods to accomplish the same.

### The Selector

The selector is what's backing the `Poll`instance and is where the blocking call to wait for events occur.

As you see, `Selector`is really only a wrapper around the CompletionPort handle with a few important methods.

In `Selector::new()`we actually create a `CompletionPort` which is the first thing we need to get started with `IOCP`.

Next is a method to get a `Registrator`instance. The `is_poll_dead`argument is passed in from the public `Poll`instance so the `Registrator`will be tied to that.

The `Selector::select()`method is where we actually call `get_queued_completion_status_ex`which will ask the OS to suspend the thread it's called from and wake it up when an action has completed.

{% hint style="info" %}
An interesting thing to note here is that we need to handle the case of a `WAIT_TIMEOUT`error. This error has the error code `258`. We can retrieve the error code by calling the `raw_os_error()`which is provided to use by the [std::io::Error](https://doc.rust-lang.org/std/io/struct.Error.html#method.raw_os_error) type.
{% endhint %}

Lastly we implement `Drop`for `Selector`. When `Selector`goes out of scope and is dropped we should be done with our `CompletionPort`and we should take care to clean up after ourselves by calling `ffi::close_handle`to close the port properly.

```rust
#[derive(Debug)]
pub struct Selector {
    completion_port: isize,
}

impl Selector {
    pub fn new() -> io::Result<Self> {
        // set up the queue
        let completion_port = ffi::create_completion_port()?;

        Ok(Selector { completion_port })
    }

    pub fn registrator(&self, is_poll_dead: Arc<AtomicBool>) -> Registrator {
        Registrator {
            completion_port: self.completion_port,
            is_poll_dead,
        }
    }

    /// Blocks until an Event has occured. Never times out.
    pub fn select(
        &mut self,
        events: &mut Vec<ffi::OVERLAPPED_ENTRY>,
        timeout: Option<i32>,
    ) -> io::Result<()> {

        // Windows want timeout as u32 so we cast it as such
        let timeout = timeout.map(|t| t as u32);

        // first let's clear events for any previous events and wait until we 
        // get some more
        events.clear();
        let ul_count = events.capacity() as u32;

        let removed_res = ffi::get_queued_completion_status_ex(
            self.completion_port as isize,
            events,
            ul_count,
            timeout,
            false,
        );

        // We need to handle the case that the "error" was a WAIT_TIMEOUT error.
        // the code for this error is 258 on Windows. We don't treat this as an 
        // error but set the events returned to 0. (i tried to do this in the 
        // `ffi::get_queued_completion_status_ex` function but there was an error
        // I didn't manage to resolve).
        let removed = match removed_res {
            Ok(n) => n,
            Err(ref e) if e.raw_os_error() == Some(258) => 0,
            Err(e) => return Err(e),
        };

        unsafe {
            events.set_len(removed as usize);
        }

        Ok(())
    }
}

impl Drop for Selector {
    fn drop(&mut self) {
        match ffi::close_handle(self.completion_port) {
            Ok(_) => (),
            Err(e) => {
                if !std::thread::panicking() {
                    panic!(e);
                }
            }
        }
    }
}
```

### Conclusion

If you got all the way here to the conclusion by reading all this code, I'm impressed. Good job! 

`IOCP`supports a [variety of resources](https://docs.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports#supported-io-functions) which can be used in a similar way to what we show here. Just remember to read the [It's not that easy](its-not-that-easy.md) chapter to get some overview of the shortcuts and scenarios we haven't covered here.

If you haven't cloned the repo yet or want to have a look at the complete code presented in this chapter check out the [repository.](https://github.com/cfsamson/examples-minimio)

I'll leave you with a link to the full code presented here:

{% embed url="https://github.com/cfsamson/examples-minimio/blob/master/src/windows.rs" %}





\`\`


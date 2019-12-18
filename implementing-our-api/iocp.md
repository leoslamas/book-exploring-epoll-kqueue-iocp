# IOCP

{% hint style="info" %}
We're starting off with Windows and IOCP since, well, I did it the other way around on my first go at this project and that turned out to require a total rewrite when I tried to fit the completion based `IOCP` model to the readiness based models like `kqueue`and `epoll`.

It's also by far going to be the implementation requiring most lines of code.
{% endhint %}

Let's move in to the `windows.rs`file in our project.

To implement an IOCP backed event queue you need to interact with the operating system through a set of syscalls. Let's create a submodule in the `windows.rs`file to contain these calls.

## FFI module

```rust
mod ffi {
    ...
}
```

Inside this submodule, let's define the `extern` functions we're going to call to make these syscalls:

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

This method will registers an interest in a `data recieved`event. Together with the [WSASend](https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsasend) method this let's us wait for completed `Read`and `Write`operations. This will be the method we call to indicate to the operating system that we're interested to get a notification when the `Recieve`event has finished on our `Socket`. We call this from our `Registrator`.

Windows, calls an operation we can await in an event queue an `Overlapped Operation`. Knowing this is handy when it comes to understanding the names of data structures and functions on Windows.

{% hint style="info" %}
This function can return `0`, which means no error occurred and that the `Recieve`operation completed immediately. **This is however not what we expect this function to return!** 

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
/// The number of bytes recieved
pub fn wsa_recv(s: RawSocket, wsabuffers: &mut [WSABUF]) -> Result<WSAOVERLAPPED, io::Error> {
    let mut ol = WSAOVERLAPPED::zeroed();
    let mut flags = 0;

    let res = unsafe {
        WSARecv(
            s,
            wsabuffers.as_mut_ptr(),
            1,
            ptr::null_mut(),
            &mut flags,
            &mut ol,
            ptr::null_mut(),
        )
    };
    
    if res != 0 {
        let err = unsafe { WSAGetLastError() };
        
        if err == WSA_IO_PENDING {
            // Everything is OK, and we can wait this with GetQueuedCompletionStatusEx
            Ok(ol)
        } else {
            Err(std::io::Error::last_os_error())
        }
    } else {
        // The socket is already ready so we don't need to queue it
        // TODO: Avoid queueing this
        Ok(ol)
    }
}
```

### PostQueuedCompletionStatus

This let's us post an event to our event queue. We only need this to register an event which lets us wake up the thread waiting for events to close it down.

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

Ok, so this is the actually blocking call which waits for new events. There are two related functions called `GetQueuedCompletionStatusEx`and `GetQueuedCompletionStatus`. The difference between them is that `GetQueuedCompletionStatus`recieves one and one event, while the `...Ex`version can recieve multiple events simultaniously.

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

This function retrieves the last error we got. Normally it's enough to check if there was an error and use Rusts standard librarys `std::io::Error::last_os_error()`to retrieve the error. However, in our `WSARecv`function we need to retrieve information about the error and get the correct error code.

We'll not create a safe wrapper around this since we only call this function in our other safe wrappers and never outside the `ffi`module.

## Structures and Constants:

We need to define some structures and some constants which are missing from our `ffi`module inside `windows.rs`so far. I'll present more constants than we actually use, but if you want to play around with this they might come in handy.

```rust
 #[derive(Debug, Clone)]
    pub struct IOCPevent {}

    impl Default for IOCPevent {
        fn default() -> Self {
            IOCPevent {}
        }
    }

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
        // Normally a pointer but since it's just passed through we can 
        // store whatever valid usize we want. For our case an Id or Token 
        // is more secure than dereferencing som part of memory later.
        lp_completion_key: *mut usize,
        pub lp_overlapped: *mut OVERLAPPED,
        internal: usize,
        bytes_transferred: u32,
    }

    impl OVERLAPPED_ENTRY {
        pub fn id(&self) -> Token {
            self.lp_completion_key as usize
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
        /// If an overlapped I/O operation is issued without an I/O 
        /// completion routine (the operation's lpCompletionRoutine parameter 
        /// is set to null), then this parameter should either contain a 
        /// valid handle to a WSAEVENT object or be null. If the
        /// lpCompletionRoutine parameter of the call is non-null then 
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

    // https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped
    #[repr(C)]
    #[derive(Debug)]
    pub struct OVERLAPPED {
        pub internal: ULONG_PTR,
        internal_high: ULONG_PTR,
        dummy: [DWORD; 2],
        h_event: HANDLE,
    }

    #[repr(C)]
    pub struct WSADATA {
        w_version: u16,
        i_max_sockets: u16,
        i_max_u_dp_dg: u16,
        lp_vendor_info: u8,
        sz_description: [u8; WSADESCRIPTION_LEN + 1],
        sz_system_status: [u8; WSASYS_STATUS_LEN + 1],
    }

    impl Default for WSADATA {
        fn default() -> WSADATA {
            WSADATA {
                w_version: 0,
                i_max_sockets: 0,
                i_max_u_dp_dg: 0,
                lp_vendor_info: 0,
                sz_description: [0_u8; WSADESCRIPTION_LEN + 1],
                sz_system_status: [0_u8; WSASYS_STATUS_LEN + 1],
            }
        }
    }

    // You can find most of these here: 
    // https://docs.microsoft.com/en-us/windows/win32/winprog/windows-data-types
    // The HANDLE type is actually a `*mut c_void` but windows preserves 
    // backwards compatibility by allowing a INVALID_HANDLE_VALUE which 
    // is `-1`. We can't express that in Rust so it's much easier for us to treat
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
    pub type LPOVERLAPPED = *mut OVERLAPPED;
    pub type LPWSAOVERLAPPED_COMPLETION_ROUTINE = *const extern "C" fn();

    // https://referencesource.microsoft.com/#System.Runtime.Remoting/channels/ipc/win32namedpipes.cs,edc09ced20442fea,references
    // read this! https://devblogs.microsoft.com/oldnewthing/20040302-00/?p=40443
    /// Defined in `win32.h` which you can find on your windows system
    pub const INVALID_HANDLE_VALUE: HANDLE = -1;

    // https://docs.microsoft.com/en-us/windows/win32/winsock/windows-sockets-error-codes-2
    pub const WSA_IO_PENDING: i32 = 997;

    // This can also be written as `4294967295` if you look at sources on 
    // the internet. Interpreted as an i32 the value is -1
    // see for yourself: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=4b93de7d7eb43fa9cd7f5b60933d8935
    pub const INFINITE: u32 = 0xFFFFFFFF;

    pub const WSADESCRIPTION_LEN: usize = 256;
    pub const WSASYS_STATUS_LEN: usize = 128;

```

```rust
  // We require a `&mut TcpStram` here, but we only need it for Windows. Now
    // there are ways to deal with this, but either way we need to in reality
    // mutate our `TcpSocket` on Windows the way we have designed this. If we
    // implement this as part of a Runtime which we control there are ways for
    // us to guarantee that the buffer is not touched or moved, so we could
    // mutate it without the user knowing safely, but this is not a Runtime so
    // we can't know that.
    pub fn register_with_id(
        &self,
        stream: &mut TcpStream,
        interests: Interests,
        token: usize,
    ) -> io::Result<Token> {
        self.registry
            .selector
            .registrator(self.is_poll_dead.clone())
            .register(stream, token, interests)?;
        Ok(Token::new(token))
    }
```

```rust
#![allow(non_camel_case_types)]
#![allow(dead_code)]

use crate::{Interests, Token};
use std::io::{self, Read, Write};
use std::net;
use std::os::windows::io::{AsRawSocket, RawSocket};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

pub type Event = ffi::OVERLAPPED_ENTRY;
pub type Source = std::os::windows::io::RawSocket;

#[derive(Debug)]
pub struct TcpStream {
    inner: net::TcpStream,
    buffer: Vec<u8>,
    wsabuf: Vec<ffi::WSABUF>,
    event: Option<ffi::WSAOVERLAPPED>,
    token: Option<usize>,
    pos: usize,
    // status: TcpReadiness,
}

// On Windows we need to be careful when using IOCP on a server. Since we're "lending"
// access to the OS over memory we crate (we're not giving over ownership, 
// but can't touch while it's lent either),
// it's easy to exploit this by issuing a lot of requests while delaying our
// responses. By doing this we would force the server to hand over so many write
// read buffers while waiting for clients to respond that it might run out of memory. 
// Now the way we would normally handle this is to have a counter and limit the
// number of outstandig buffers, queueing requests and only handle them when the
// counter is below the high water mark. The same goes for using unlimited timeouts.
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
            //status: TcpReadiness::NotReady,
        })
    }

    pub fn source(&self) -> Source {
        self.inner.as_raw_socket()
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

        ffi::create_io_completion_port(soc.as_raw_socket(), self.completion_port, token)?;

        if interests.is_readable() {
            ffi::wsa_recv(soc.as_raw_socket(), &mut soc.wsabuf)?;
        } else {
            unimplemented!();
        }

        Ok(())
    }

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

// possible Arc<InnerSelector> needed
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

    /// Blocks until an Event has occured. Never times out. We could take a parameter
    /// for a timeout and pass it on but we'll not do that in our example.
    pub fn select(
        &mut self,
        events: &mut Vec<ffi::OVERLAPPED_ENTRY>,
        timeout: Option<i32>,
    ) -> io::Result<()> {
        // calling GetQueueCompletionStatus will either return a handle to a "port" ready to read or
        // block if the queue is empty.

        // Windows want timeout as u32 so we cast it as such
        let timeout = timeout.map(|t| t as u32);

        // first let's clear events for any previous events and wait until we get som more
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
        // the code for this error is 258 on Windows. We don't treat this as an error
        // but set the events returned to 0.
        // (i tried to do this in the `ffi` function but there was an error)
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

mod ffi {
    use super::*;
    use std::io;
    use std::os::windows::io::RawSocket;
    use std::ptr;

    #[derive(Debug, Clone)]
    pub struct IOCPevent {}

    impl Default for IOCPevent {
        fn default() -> Self {
            IOCPevent {}
        }
    }

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
        // Normally a pointer but since it's just passed through we can store whatever valid usize we want. For our case
        // an Id or Token is more secure than dereferencing som part of memory later.
        lp_completion_key: *mut usize,
        pub lp_overlapped: *mut OVERLAPPED,
        internal: usize,
        bytes_transferred: u32,
    }

    impl OVERLAPPED_ENTRY {
        pub fn id(&self) -> Token {
            self.lp_completion_key as usize
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
        /// (the operation's lpCompletionRoutine parameter is set to null), then this parameter
        /// should either contain a valid handle to a WSAEVENT object or be null. If the
        /// lpCompletionRoutine parameter of the call is non-null then applications are free
        /// to use this parameter as necessary.
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

    // https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-overlapped
    #[repr(C)]
    #[derive(Debug)]
    pub struct OVERLAPPED {
        pub internal: ULONG_PTR,
        internal_high: ULONG_PTR,
        dummy: [DWORD; 2],
        h_event: HANDLE,
    }

    #[repr(C)]
    pub struct WSADATA {
        w_version: u16,
        i_max_sockets: u16,
        i_max_u_dp_dg: u16,
        lp_vendor_info: u8,
        sz_description: [u8; WSADESCRIPTION_LEN + 1],
        sz_system_status: [u8; WSASYS_STATUS_LEN + 1],
    }

    impl Default for WSADATA {
        fn default() -> WSADATA {
            WSADATA {
                w_version: 0,
                i_max_sockets: 0,
                i_max_u_dp_dg: 0,
                lp_vendor_info: 0,
                sz_description: [0_u8; WSADESCRIPTION_LEN + 1],
                sz_system_status: [0_u8; WSASYS_STATUS_LEN + 1],
            }
        }
    }

    // You can find most of these here: https://docs.microsoft.com/en-us/windows/win32/winprog/windows-data-types
    /// The HANDLE type is actually a `*mut c_void` but windows preserves backwards compatibility by allowing
    /// a INVALID_HANDLE_VALUE which is `-1`. We can't express that in Rust so it's much easier for us to treat
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
    pub type LPOVERLAPPED = *mut OVERLAPPED;
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

    pub const WSADESCRIPTION_LEN: usize = 256;
    pub const WSASYS_STATUS_LEN: usize = 128;

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

        fn GetQueuedCompletionStatus(
            CompletionPort: HANDLE,
            lpNumberOfBytesTransferred: LPDWORD,
            lpCompletionKey: PULONG_PTR,
            lpOverlapped: LPOVERLAPPED,
            dwMilliseconds: DWORD,
        ) -> i32;

        // https://docs.microsoft.com/nb-no/windows/win32/api/handleapi/nf-handleapi-closehandle
        fn CloseHandle(hObject: HANDLE) -> i32;

        // https://docs.microsoft.com/nb-no/windows/win32/api/winsock/nf-winsock-wsagetlasterror
        fn WSAGetLastError() -> i32;
    }

    // ===== SAFE WRAPPERS =====

    pub fn close_handle(handle: isize) -> io::Result<()> {
        let res = unsafe { CloseHandle(handle) };

        if res == 0 {
            Err(std::io::Error::last_os_error().into())
        } else {
            Ok(())
        }
    }

    pub fn create_completion_port() -> io::Result<isize> {
        unsafe {
            // number_of_concurrent_threads = 0 means use the number of physical threads but the argument is
            // ignored when existing_completionport is set to null.
            let res = CreateIoCompletionPort(INVALID_HANDLE_VALUE, 0, ptr::null_mut(), 0);
            if (res as *mut usize).is_null() {
                return Err(std::io::Error::last_os_error());
            }

            Ok(res)
        }
    }

    /// Returns the file handle to the completion port we passed in
    pub fn create_io_completion_port(
        s: RawSocket,
        completion_port: isize,
        token: usize,
    ) -> io::Result<isize> {
        let res =
            unsafe { CreateIoCompletionPort(s as isize, completion_port, token as *mut usize, 0) };

        if (res as *mut usize).is_null() {
            return Err(std::io::Error::last_os_error());
        }

        Ok(res)
    }

    /// Creates a socket read event.
    /// ## Returns
    /// The number of bytes recieved
    pub fn wsa_recv(s: RawSocket, wsabuffers: &mut [WSABUF]) -> Result<WSAOVERLAPPED, io::Error> {
        let mut ol = WSAOVERLAPPED::zeroed();
        let mut flags = 0;

        let res = unsafe {
            WSARecv(
                s,
                wsabuffers.as_mut_ptr(),
                1,
                ptr::null_mut(),
                &mut flags,
                &mut ol,
                ptr::null_mut(),
            )
        };
        if res != 0 {
            let err = unsafe { WSAGetLastError() };
            if err == WSA_IO_PENDING {
                // Everything is OK, and we can wait this with GetQueuedCompletionStatus
                Ok(ol)
            } else {
                Err(std::io::Error::last_os_error())
            }
        } else {
            // The socket is already ready so we don't need to queue it
            // TODO: Avoid queueing this
            Ok(ol)
        }
    }

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

    /// ## Parameters:
    /// - *completion_port:* the handle to a completion port created by calling CreateIoCompletionPort
    /// - *completion_port_entries:* a pointer to an array of OVERLAPPED_ENTRY structures
    /// - *ul_count:* The maximum number of entries to remove
    /// - *timeout:* The timeout in milliseconds, if set to NONE, timeout is set to INFINITE
    /// - *alertable:* If this parameter is FALSE, the function does not return until the time-out period has elapsed or
    /// an entry is retrieved. If the parameter is TRUE and there are no available entries, the function performs
    /// an alertable wait. The thread returns when the system queues an I/O completion routine or APC to the thread
    /// and the thread executes the function.
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
        // can't coerce directly to *mut *mut usize and cant cast `&mut` as `*mut`
        // let completion_key_ptr: *mut &mut usize = completion_key_ptr;
        // // but we can cast a `*mut ...`
        // let completion_key_ptr: *mut *mut usize = completion_key_ptr as *mut *mut usize;
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

    pub fn get_queued_completion_status(
        completion_port: isize,
        bytes_transferred: &mut u32,
        completion_key: usize,
        overlapped: &mut OVERLAPPED,
        timeout: Option<u32>,
    ) -> io::Result<u32> {
        let ul_num_entries_removed: u32 = 0;
        // can't coerce directly to *mut *mut usize and cant cast `&mut` as `*mut`
        // let completion_key_ptr: *mut &mut usize = completion_key_ptr;
        // // but we can cast a `*mut ...`
        // let completion_key_ptr: *mut *mut usize = completion_key_ptr as *mut *mut usize;
        let timeout = timeout.unwrap_or(INFINITE);
        let res = unsafe {
            GetQueuedCompletionStatus(
                completion_port,
                bytes_transferred,
                completion_key as *mut *mut usize,
                overlapped,
                timeout,
            )
        };

        if res == 0 {
            Err(std::io::Error::last_os_error())
        } else {
            Ok(ul_num_entries_removed)
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn selector_new_creates_valid_port() {
        let selector = Selector::new().expect("create completion port failed");
        assert!(selector.completion_port > 0);
    }

    #[test]
    fn selector_register() {
        let selector = Selector::new().expect("create completion port failed");
        let poll_is_alive = Arc::new(AtomicBool::new(false));
        let registrator = selector.registrator(poll_is_alive.clone());
        let mut sock: TcpStream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
        let request = "GET /delay/1000/url/http://www.google.com HTTP/1.1\r\n\
                       Host: slowwly.robertomurray.co.uk\r\n\
                       Connection: close\r\n\
                       \r\n";
        sock.write_all(request.as_bytes())
            .expect("Error writing to stream");

        registrator
            .register(&mut sock, 1, Interests::READABLE)
            .expect("Error registering sock read event");
    }

    #[test]
    fn selector_select() {
        let mut selector = Selector::new().expect("create completion port failed");
        let poll_is_alive = Arc::new(AtomicBool::new(false));
        let registrator = selector.registrator(poll_is_alive.clone());
        let mut sock: TcpStream = TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
        let request = "GET /delay/1000/url/http://www.google.com HTTP/1.1\r\n\
                       Host: slowwly.robertomurray.co.uk\r\n\
                       Connection: close\r\n\
                       \r\n";
        sock.write_all(request.as_bytes())
            .expect("Error writing to stream");

        registrator
            .register(&mut sock, 2, Interests::READABLE)
            .expect("Error registering sock read event");
        let entry = ffi::OVERLAPPED_ENTRY::zeroed();
        let mut events: Vec<ffi::OVERLAPPED_ENTRY> = vec![entry; 255];
        selector.select(&mut events, None).expect("Select failed");

        for event in events {
            let ol = unsafe { &*(event.lp_overlapped) };
            println!("EVT_OVERLAPPED {:?}", ol);
            println!("OVERLAPPED_STATUS {:?}", ol.internal as usize);
            println!("COMPL_KEY: {:?}", event.id());
        }

        println!("SOCKET AFTER EVENT RETURN: {:?}", sock);

        let mut buffer = String::new();
        sock.read_to_string(&mut buffer).unwrap();

        println!("BUFFERS: {}", buffer);
    }
}

```


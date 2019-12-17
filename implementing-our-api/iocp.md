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
This function can return `0`, which means no error occurred and that the `Recieve`operation completed immediately. In this case there is nothing for us to await and we should handle the case when data is already there \(we'll not cover this edge case in our implementation though\). **This is however not what we expect this function to return!**

So you thought the `Success`case was what we're expecting, yes? Funny thing, in this specific case we expect failure. 

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


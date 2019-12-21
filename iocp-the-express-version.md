# IOCP - the express version

```rust
#![allow(non_camel_case_types)]

use std::io::{self, Write};
use std::net::TcpStream;
use std::os::windows::io::AsRawSocket;
use std::ptr;

fn main() {
    // A counter to keep track of how many events we're expecting to act on
    let mut event_counter = 0;

    // First we create the event queue.
    // The size argument is ignored but needs to be larger than 0
    let queue =
        unsafe { ffi::CreateIoCompletionPort(ffi::INVALID_HANDLE_VALUE, 0, ptr::null_mut(), 0) };
    // We handle any error here panicing.
    if (queue as *mut usize).is_null() {
        panic!(std::io::Error::last_os_error());
    }

    // As you'll see below, we need a place to store the streams so they're
    // not closed
    let mut streams = vec![];

    // This is a trick often used in IOCP to attach some additional context
    // to our Event
    #[derive(Debug)]
    #[repr(C)]
    struct Operation {
        wsaoverlapped: ffi::WSAOVERLAPPED,
        context: usize,
    };

    // we need a place to store these "Operations"
    let mut ops: Vec<Operation> = Vec::with_capacity(5);

    // and we need a place to store buffers
    let mut buffers: Vec<Vec<u8>> = Vec::with_capacity(5);

    // We crate 5 requests to an an endpoint we control the delay on
    for i in 1..6 {
        // This site has an api to simulate slow responses from a server
        let addr = "slowwly.robertomurray.co.uk:80";
        let mut stream = TcpStream::connect(addr).unwrap();

        let delay = (5 - i) * 1000;
        let request = format!(
            "GET /delay/{}/url/http://www.google.com HTTP/1.1\r\n\
             Host: slowwly.robertomurray.co.uk\r\n\
             Connection: close\r\n\
             \r\n",
            delay
        );
        stream.write_all(request.as_bytes()).unwrap();

        stream.set_nonblocking(true).unwrap();

        // we need to register this resource with the completion port
        let res = unsafe { ffi::CreateIoCompletionPort(
            stream.as_raw_socket() as isize, 
            queue, 
            ptr::null_mut(),
            0) 
        };

        if (res as *mut usize).is_null() {
            panic!(std::io::Error::last_os_error());
        }

        // crate a zeroed WSAOVERLAPPED struct
        let event = ffi::WSAOVERLAPPED::zeroed();
        // Wrap it in Operation, remember, first part of `Operation` will be
        // laid out exactly as `WSAOVERLAPPED` in memory
        let op = Operation {
            wsaoverlapped: event,
            context: i,
        };
        // store the operation so its possible for us to retrieve it afterwards
        ops.push(op);
        let op = ops.last_mut().unwrap();
        // we need to coerce it to a pointer tp `Operation` since we'll cast it
        // as a pointer to `WSAOVERLAPPED` later.
        let op_ptr: *mut Operation = op;

        // The buffer which IOCP will read data into
        let buffer: Vec<u8> = Vec::with_capacity(1024);
        buffers.push(buffer);
        let buffer = buffers.last_mut().unwrap();

        // To actually send the buffer to IOCP we need to get the inner
        // bytearray and specify the size and use the `WSABUF` struct.
        let wsabuf = &mut [ffi::WSABUF::new(1024, buffer.as_mut_ptr())];

        // This is where we actually register an interest to get notified when
        // the buffers are filled with data.
        let mut flags = 0;
        let res = unsafe {
            ffi::WSARecv(
                stream.as_raw_socket(),
                wsabuf.as_mut_ptr(),
                1,
                ptr::null_mut(),
                &mut flags,
                op_ptr as *mut ffi::WSAOVERLAPPED,
                ptr::null_mut(),
            )
        };

        if res != 0 {
            let err = unsafe { ffi::WSAGetLastError() };
            // This is not an "error", it's what we expect!
            if err == ffi::WSA_IO_PENDING {
                // Everything is OK, and we can wait this with GetQueuedCompletionStatus
                ()
            } else {
                panic!(std::io::Error::last_os_error());
            }
        } else {
            // The socket is already ready so we don't need to queue it
            // we won't handle that case here!
            ()
        };

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
        let mut events: Vec<ffi::OVERLAPPED_ENTRY> = Vec::with_capacity(10);

        // This call will actually block until an event occurs. The timeout
        // of `-1` means no timeout so we'll block until something happens.
        // Now the OS suspends our thread doing a context switch and work
        // on someting else - or just perserve power.

        let mut entries_removed: u32 = 0;
        let res = unsafe {
            ffi::GetQueuedCompletionStatusEx(
                queue,
                events.as_mut_ptr(),
                events.capacity() as u32,
                &mut entries_removed,
                ffi::INFINITE,
                false,
            )
        };

        if res == 0 {
            panic!(io::Error::last_os_error());
        };

        // This one unsafe we could avoid though but this technique is used
        // in libraries like `mio` and is safe as long as the OS does
        // what it's supposed to.
        unsafe { events.set_len(res as usize) };

        for event in events {
            let operation = unsafe { &*(event.lp_overlapped as *const Operation) };
            println!("RECIEVED: {:?}", operation);
            event_counter -= 1;
        }
    }

    // When we manually initialize resources we need to manually clean up
    // after our selves as well. Normally, in Rust, there will be a `Drop`
    // implementation which takes care of this for us.
    let res = unsafe { ffi::CloseHandle(queue) };
    if res == 0 {
        panic!(io::Error::last_os_error());
    };
    println!("FINISHED");
}

mod ffi {

    use std::os::windows::io::RawSocket;
    use std::ptr;

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
        pub lp_overlapped: *mut WSAOVERLAPPED,
        internal: usize,
        bytes_transferred: u32,
    }

    impl OVERLAPPED_ENTRY {

        pub(crate) fn zeroed() -> Self {
            OVERLAPPED_ENTRY {
                lp_completion_key: ptr::null_mut(),
                lp_overlapped: ptr::null_mut(),
                internal: 0,
                bytes_transferred: 0,
            }
        }
    }

    #[repr(C)]
    #[derive(Debug)]
    pub struct WSAOVERLAPPED {
        internal: ULONG_PTR,
        internal_high: ULONG_PTR,
        offset: DWORD,
        offset_high: DWORD,

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


    pub type HANDLE = isize;
    pub type BOOL = bool;
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

    #[link(name = "Kernel32")]
    extern "stdcall" {
        pub fn CreateIoCompletionPort(
            filehandle: HANDLE,
            existing_completionport: HANDLE,
            completion_key: ULONG_PTR,
            number_of_concurrent_threads: DWORD,
        ) -> HANDLE;

        pub fn WSARecv(
            s: RawSocket,
            lpBuffers: LPWSABUF,
            dwBufferCount: DWORD,
            lpNumberOfBytesRecvd: LPDWORD,
            lpFlags: LPDWORD,
            lpOverlapped: LPWSAOVERLAPPED,
            lpCompletionRoutine: LPWSAOVERLAPPED_COMPLETION_ROUTINE,
        ) -> i32;

        pub fn GetQueuedCompletionStatusEx(
            CompletionPort: HANDLE,
            lpCompletionPortEntries: *mut OVERLAPPED_ENTRY,
            ulCount: ULONG,
            ulNumEntriesRemoved: PULONG,
            dwMilliseconds: DWORD,
            fAlertable: BOOL,
        ) -> i32;

        pub fn CloseHandle(hObject: HANDLE) -> i32;

        pub fn WSAGetLastError() -> i32;
    }
}


```


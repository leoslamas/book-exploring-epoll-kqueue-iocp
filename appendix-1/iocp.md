---
description: >-
  IOCP is how Windows provides the infrastructure we need to create a scalable
  event queue. It differs from the two others since IOCP uses a completion based
  model.
---

# IOCP Reference

## Quick reference of important IOCP syscalls

### CreateCompletionPort

This syscall gives us a handle to a completion port. To us, this will represent the queue which we register an interest in getting notified on when an event has happened.

{% embed url="https://docs.microsoft.com/en-us/windows/win32/fileio/createiocompletionport" %}

### WSARecv

This syscall registers an interest in `Read`events on a socket. Windows calls this an `Receive`event.

{% embed url="https://docs.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-wsarecv" %}

### GetQueuedCompletionStatusEx

This syscall comes in a different flavor as well called `GetQueuedCompletionStatus`which only returns one event at a time. The `Ex`version can return multiple events that has been completed, and we'll use that since we there is no point for us to only handle one and one event at a time.

{% embed url="https://docs.microsoft.com/nb-no/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus" %}

### WSAGetLastError

We actually need this too since a perfectly normal situation is returned as an error on Windows. We have to check if the error is `WSA_IO_PENDING`which we most likely will get when calling `WSARecv`. This specific error simply means that the action we're waiting for is not ready immediately, and can be waited on by calling `GetQueuedCompletionStatusEx`.

{% embed url="https://docs.microsoft.com/nb-no/windows/win32/api/winsock/nf-winsock-wsagetlasterror" %}

### CloseHandle

When requesting resources from the OS directly we're also responsible for letting the OS know we're done with them. This function lets us do just that.

{% embed url="https://docs.microsoft.com/nb-no/windows/win32/api/handleapi/nf-handleapi-closehandle" %}

### PostQueuedCompletionStatus

It turns out that closing a thread which has yielded control to the OS is not so easy. We actually need to wake up the thread and let it know that we want it to stop waiting, release it's resources and close down gracefully. To do this, we issue a timeout event with the timeout set to 0 which means it times out immediately. We have to register interest to get woken up from our blocking `Poll::poll()`when the timeout event has completed, and we need this syscall will let us register such an interest to our completion port.

{% embed url="https://docs.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus" %}

So, this is the basic infrastructure we need on Windows. As we'll see when we look through the code, each of this call requires us to pass in some structures and flags to actually make them do what we want but it's good to keep this high-level overview in mind when we dig into the details.


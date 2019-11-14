# The recipie for an eventqueue

Lets look at the ingredient list for a simple cross platform event queue:

## The recipie consists of the following components

### `Poll`

The `Poll`structure is the heart of our event queue. This structure implements one important method: `poll()`. When called this method tells the OS _"I want you to park the thread I'm in, work on something else, and wake me up when one of the events I told you I'm interested in is ready"._

### `Registrator`

The `Registrator`has one job. It tells our operating system that we're interested in an event. An example of this is when data is ready for us to read on a network socket. The registrator is tied to `Poll`in the sense that we want the OS to wake us up from the call to `poll()`when the event is ready. Every event queue has an ID that is unique. Typically `Registrator`and `Poll`is on two different threads.

### `Event`

`Poll`can wait for any type of event it supports. Some examples of this is `Read`or `Write`events on a socket or a pipe. We call `Read`and `Write`. These types of events depends on what kind of resource we're working with. Timeouts is another type of event all systems support.

### `TcpStream` \(or any other resource\)

TcpStream is an I/O resource we want to register an interest in. This needs to be abstracted over if we want to support both `Epoll`,`Kqueue`and `IOCP`. We'll only cover the case of beeing interested in `Read`events on the `TcpStream`but this is only limited to what kind of resources the OS allows us to register interests on.



This is really all there is to it. The concept is not difficult, but making these compontents requires some raw materials which is dependant on the OS. Let's dive in and introduce everything we need to make this event queue work.


---
description: Well if any of this classify as "easy" at all...
---

# It's not that easy

heThe funny thing about the excercise in this book is that a lot of the work is setting up the infrastructure to get a running event loop in the first place. Actually adding new functionality is much easier, but to keep this example as short as I can we skip a lot of important things to actually implement a fully working event queue. I'll point some of the ones I consider the most important here.

### Everything in networking is asynchronous

In our example we only treat `Read`events on a sockets. In reality, both the opening of a socket, resolving the DNS and writing data to a socket are all blocking tasks. 

This actually means that we'll run into trouble using the standard library `std::net::TcpStream`as the real workhorse behind our own `TcpStream`since there is no way to open a socket without resolving the DNS using a blocking call. In reality we need to build our own `TcpStream`from scratch if we want it to be truly asynchronous.

### Communication could be interrupted

We're not accounting for the fact that communication could be interrupted. In such a case the right thing to do would be to suspend the task again and register interest for further events on that resource with our event queue.

### Buffers could be full while there is more data to read

This is particularely a problem we have with our `IOCP`implementation. We only submit one fixed buffer and does not even consider the possibility that we might recieve more data than would fit in it. Neither do we consider how we would optimally arrange these buffers to avoid allocating more memory than we need and make sure performance does not suffer.

The right thing to do here is to decide on a buffering strategy, and this can be a complicated topic itself. However, getting a working and correct solution is not that much work, but if you want to win be the fastest `I/O`library in the world, there is a lot to consider. 

We should check if we have read all the data or not and resubmit interest to our queue for more data if needed.

### Resolving DNS

As mentioned in the first paragraph of this chapter, almost everything is blocking at some level. DNS lookup is one of those things where we have to decide on a strategy. The optimal strategy for this might depend on our exact use case.

DNS entries are often cached by the OS. This means that if most of our DNS lookups are cached, the most efficient thing would probably be to actually resolve this in a blocking manner since there will be no real `I/O`. Deferring this to a thread pool, or adding such an interest to the event queue will most likely be slower than to just block if it's cached.

If most of our DNS lookups are not cached locally, this will be a bad strategy. It's very common to delegate the DNS lookup to a thread pool in these instances. First of all, some entries are cached and will return immidiately, and if we need to wait for it a threadpool will make sure we only block there and it's most likely for a realively short amount of time.

There is also the case of what APIs and methods are available for evented DNS lookup. Most experiences I've read seems to suggest that these APIs are pretty unergonomic to use and that it's difficult to justify that added complexity with the gains it would have \(if any at all\).

### Level triggered vs Edge triggered events

So this is actually hides some pretty important complexity we skipped for the `readiness`based models. The default mode for `epoll`is `level triggered`. This means that if we don't immidiately start reading from the socket, it will keep "nagging" us and trigger notifications that it's ready to be read.

Now this behaviour is most often unwanted. Our `executor`or `scheduler`might want to defer reading from that socket for a while for some reason if there is a lot of work to do. In this case our event queue will be woken up repeatedly to get notified of something we already know.

In `edge triggered`mode, we only get this notification once. We only get notifications on state transitions. An example of this is when the `socket`changes state from `NotReady`to `Ready`.

Another advantage of this is when we have multiple threads waiting on the same event queue. Let's pretent that we designed our `Poll`instance to actually be cloned and shared to multiple threads.

`edge triggered`mode guarantees that only one thread will be woken up and notified about the event. `level triggered`mode does not make this guarantee. This means you need additional logic \(and checks\) to see if an event is already beeing handled by another thread.

There are similar flags we can pass to `kqueue`even though it's not named the same there, but the same considerations is valid there as well.

{% hint style="info" %}
We circumvented this by using `EPOLLONESHOT.` Using this actually removes the resource from the event queue alltoghether, and since we made our `socket`blocking on recieving the first event we knew that we would only block if for some reason we recieved an `EAGAIN` error on the socket.

The proper thing to do here is to read until we get a "loss of readiness" \(if we get that at all\) and suspend the task and wait for another `Ready` event. When using `edge triggered`mode the socket will notify us when it changes state to `Ready`again and we can resume.
{% endhint %}

### Thread Safety and Race Conditions

Our implementation is pretty brittle in some aspects. Our main focus has been how Kqueue, Epoll and IOCP work, and not how to make a perfect event loop library.

In our `Registrator::register()`method we first check if the `Poll`instance is dead using an `atomic` flag. But if we were to create multiple `Registrators`and send them to different threads we're in trouble.

_What happens if another thread closes the loop before we hit line 22 in the code below?_

We'll nothing too dramatic, but our event will never return to us and if we're counting on that we might end up blocking for ever somewhere. 

**How can we solve this?**

There are probably several possible solutions to this. The first that came to my mind was:

1. Use another flag to indicate that a "registration is in progress". In practice a "registration lock".
2. We use an atomic swap\_and\_compare in a loop to try to set that flag, exiting the loop when successfull. By successfully setting that flag we're letting other threads know that we're about to register an event and that we hold the "registration lock"
3. Then we check if the poll is closed and if not we register our event
4. We then swap the flag back which "releases" the lock and let's other threads acquire the "lock".

This is of course assuming that the registration process itself is so fast that spinning on contention is preferred to other methods of synchronizing access like a `Mutex`or `CondVar`.

Maybe you have a better idea? Feel free to discuss and suggest it in the [Issue Tracker](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp/issues) for this book.

{% hint style="info" %}
In our example library we treat this as an limitation and do not consider registrations from different threads as something we support.
{% endhint %}

```rust
pub fn register(
    &self,
    soc: &mut TcpStream,
    token: usize,
    interests: Interests,
) -> io::Result<()> {
    // SPIN WHILE TRYING TO ACQUIRE A "REGISTRATION LOCK"
    
    // THEN check if poll is dead
    if self.is_poll_dead.load(Ordering::SeqCst) {
        return Err(io::Error::new(
            io::ErrorKind::Interrupted,
            "Poll instance is dead.",
        ));    
    }

    ffi::create_io_completion_port(
        soc.as_raw_socket(), 
        self.completion_port, token
    )?;

    if interests.is_readable() {
        // What happens if the poll has been closed since we checked it?
        ffi::wsa_recv(soc.as_raw_socket(), &mut soc.wsabuf)?;
    } else {
        unimplemented!();
    }
    
    // RELEASE THE "REGISTRATION LOCK"
    Ok(())
}
```




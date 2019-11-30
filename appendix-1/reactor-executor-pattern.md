# Reactor - Executor Pattern

This pattern is most often referred to as just [Reactor Pattern](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve) is a commonly used technique for "demultiplexing" asynchronous events in the order they arrive. Don't worry I'll explain this in english below.

In Rust we often refer to both a `Reactor`and an `Executor`when we talk about it's asynchronous model, the reason for this is that Rusts `Futures`fits nicely in between as the glue that allows these two pieces to work together.

One huge advantage of this is that this allows us in theory to pick both a `Reactor`and an `Executor`which best suits our problem at hand.

Before we talk more about how this relates to `Futures`lets first have a look at the minimum components this pattern needs:

1. Reactor
   1. An event queue
   2. A way to notify the executor of an event
2. Executor
   1. Scheduler
   2. Handlers
3. Tasks
   1. User code
   2. A way to register interest in an event with the Reactor
   3. A way to define a suspend point and give the Executor control over finishing our task

### The Reactor

Our cross platform Epoll/Kqueue/IOCP library can be viewed as the main building block of the reactor - the event queue part. The only missing piece is a way for us to communicate with the `Executor`that an event is ready and we actually need to recieve the events which are ready and wake up the task which will finish. The simplest way to do this is to use a `Channel`which is exactly what we'll do.

A very simple `Reactor`can look like this:

```rust
struct Reactor {
    handle: std::thread::JoinHandle<()>,
    registrator: Registrator,
}

impl Reactor {
    fn new(evt_sender: Sender<usize>) -> Reactor {
        let mut poll = Poll::new().unwrap();
        let registrator = poll.registrator();

        // Set up the epoll/IOCP event loop in a seperate thread
        let handle = thread::spawn(move || {
            let mut events = Events::with_capacity(1024);
            loop {
                println!("Waiting! {:?}", poll);
                match poll.poll(&mut events, Some(200)) {
                    Ok(..) => (),
                    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                        println!("INTERRUPTED: {}", e);
                        break;
                    }
                    Err(e) => panic!("Poll error: {:?}, {}", e.kind(), e),
                };
                for event in &events {
                    let event_token = event.id().value();
                    evt_sender.send(event_token).expect("send event_token err.");
                }
            }
        });

        Reactor { handle, registrator }
    }

    fn register_stream_read_interest(&self, stream: &mut TcpStream, token: usize) {
        self.registrator.register(stream, token, Interests::readable()).expect("registration err.");
    }

    fn stop_loop(&self) {
        self.registrator.close_loop().expect("close loop err.");
    }
}
```

{% hint style="info" %}

{% endhint %}

### Executor

The executor needs to schedule the execution of events which are ready and provide a way for us to set handlers for events. In our example we'll specifically set a handler function which will resume our task when the corresponding event is ready.

A simple executor can look like this:

```rust
struct Excutor {
    events: Vec<(usize, Box<dyn FnMut()>)>,
    evt_reciever: Receiver<usize>,
}

impl Excutor {
    fn new(evt_reciever: Receiver<usize>) -> Self {
        Excutor { events: vec![], evt_reciever }
    }
    fn spawn(&mut self, id: usize, f: impl FnMut() + 'static) {
        self.events.push((id, Box::new(f)));
    }
    fn resume(&mut self, event: usize) {
        println!("RESUMING EVENT: {}", event);
        let (_, f) = self.events
            .iter_mut()
            .find(|(e, _)| *e == event)
            .expect("Couldn't find event.");
        f();
    }
    fn block_on_all(&mut self) {
        while let Ok(recieved_token) = self.evt_reciever.recv() {
            assert_eq!(TEST_TOKEN, recieved_token, "Non matching tokens.");
            println!("EVENT: {} is ready", recieved_token);
            self.resume(recieved_token);
        }
    }
}
```

It's not very sophisticated but will do the work for us. As you see a handler needs not be anyting more fancy than a closure which we'll invoke when the time is right. The means of communication between our `Reactor`and `Executor`is just a regular `std::sync::mspc::Channel`and all we do in `block_on_all`is to wait for events which we can respond to.

There is of course many ways which we could choose to handle this, feel free to play around and try for yourself.

 


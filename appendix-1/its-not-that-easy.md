# It's not that easy

The funny thing about the excercise in this book is that a lot of the work is setting up the infrastructure to get a running event loop in the first place. Actually adding new functionality is much easier, but to keep this example as short as I can we skip a lot of important things to actually implement a fully working event queue.

### Everything in networking is asyncrhonous

In our example we only treat `Read`events on a sockets. In reality, both the opening of a socket, resolving the DNS and writing data to a socket are all blocking tasks. 

This actually means that we'll run into trouble using the standard library `std::net::TcpStream`as the real workhorse behind our own `TcpStream`since there is no way to open a socket without resolving the DNS using a blocking call. In reality we need to build our own `TcpStream`from scratch if we want it to be truly asynchronous.

### Communication could be interrupted

### Buffers could be full while there is more data to read

### Sporious wakeups

### Why resolving DNS is often delegated to a threadpool? 

### Level triggered vs Edge triggered events


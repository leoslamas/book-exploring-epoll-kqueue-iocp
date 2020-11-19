# Introduction

I'll introduce the different concepts in the order I think they should be introduced. `Epoll`is rather well known, but that doesn't mean it's the smartest one to start with. 

IOCP uses a completion based model, which means we'll get notified when an action has completed.`Epoll`and `IOCP`is readiness based. They let us know when an action is ready to be performed.

As you might gather from this, `Epoll`and `Kqueue`notifies us earlier in the cycle since "ready to start" happens before "is completed". Since we want all three to have a unified API,  we really only have one choice:

Provide an API where we're not dependent on whether we're ready to start or done. In the case of a `Read`action on a `TcpStreap`this is when we call `TcpStream::read()`. 


---
description: >-
  Before we start talking about the details on each platform. Let's start by
  looking at why you should care by talking about ways to handle external events
  and I/O operations.
---

# Part 1: An express explanation

### Ways to handle I/O events.

I/O has one important property: it takes time. And it's not we that's busy, it's some other computer, a disk or some other peripheral that we need to wait on.

Now, to do most I/O operations, we need to go through the operating sI/O operations are per definition dependent on resources outside our CPU, whether it's the disk drive, the network card or other peripherals. Now, especially in the case of network calls, we're not only dependant on our own hardware, but we're depeinding on resources that might reside far away from our own, causing a significant delay. 

#### Blocking I/O

When we ask the Operating System to perform a blocking operation it will suspend the thread that makes the call \(it will stop executing code and store the CPU state and go on to do other things\). When data arrives for us through the network it will wake up our thread again and let us resume. 

#### Non-blocking I/O

In contrast to a blocking I/O operation, the Operating System will not suspend the thread that made an I/O request, but instead give it a handle which the thread can use to ask the operating system if the event is ready or not \(to follow our network example, if data has arrived for us or not\).

When we ask the operating system using our handle we call that `polling`since we are actively asking for a status.

Non-blocking I/O operations gives us as programmers more freedom, but as usual, that comes with a responsibility. If we poll too often, like in a loop, we will occupy a lot of CPU time to just ask for an updated status which is very wasteful. 

### Why not use blocking I/O and many threads?

You're right, that's actually a really good idea and an absolutely acceptable solution for many tasks. However, when you have many small tasks you need to handle and a lot of I/O you'll get in to some issues using this approach. An example of one such system is a web server. The tasks are usually very small,  and each task spends an unproportunate amout of time waiting compared to actually doing work. Let's consider some downsides of using one thread per task in such a system:

#### Each thread has a huge stack

Even though this might be configured on some systems, the stack each thread needs is much bigger that we might need to wait for small tasks. This results in each waiting thread will occupy a lot of memory that simply goes unused and you'll hit a bottleneck on memory pretty fast.

#### Context switches and overhead

Even though the OS is pretty good at running many threads concurrently context switches has some overhead, but the real cost lies in creting new threads which involves a lot of bookkeeping and setup related to security. We could alleviate this slightly by using a threadpool. A threadpool it's still not optimal in the typical usecase when we have a huge amount of small tasks which are mostly waiting, or if the number of tasks \(the load\) varies a lot.

### Epoll/Kqueue/IOCP

These methods let's us hook into the OS in a way in which we can wait for many events, instead of beeing limited to waiting on one event per thread, we can wait for many events on one thread. This let's us avoid one of the biggest drawbacks using one thread pr event which is all the memory that is occupied and the overhead of continously spawning new threads. Now we only have one thread waiting for many tasks.

These methods have in common that they are a sort of `blocking`I/0. If we only register one event to the epoll/kqueue/iocp event queue and wait for it, it will be no different than using blocking I/O. The advantage comes we can have a queue that waits for hundreds of thousands events with very little wasted resources.

### Readiness based and completion based models

The biggest difference in the way epoll/kqueue and IOCP work is that they use two different models:

### Readiness based models

Both Epoll and Kqueue is what we call "readiness based". This means that they'll give a notification when an operation is ready to be performed. An example of this is when data _is ready to be read_ into a buffer.

### Completion based models

IOCP is and abbreviation of Input/Output Completion Ports. As the name implies this is a completion based model which means that we get a notification when an operation has happened. An example of this is when data _has been read_ into a buffer.

While these differences seem small, it has quite an impact on the difficulty of creating a cross platform event queue as we'll see in [part 2](../the-recipie-for-an-eventqueue/) of this book. 

If you follow along on the next three chapters, you'll see how these work in practice.


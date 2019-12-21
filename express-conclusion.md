# Express conclusion

By just running through our express examples you get an overview over what `Kqueue`, `Epoll`and `IOCP`looks like in real life. There is also one other thing to note:

* Epoll took 142 lines of code
* Kqueue took 200 lines of code
* IOCP took 299 lines of code

I know it's not a fair comparison since I've added some useful information to i.e. `Kqueue`implementation on how to work with the `timespec`struct and the `IOCP`implementation is so verbose that most function call definitions end up wrapping to new lines.

However it is indicative of my experience. IOCP is more verbose than the two others. 

They all work well and have their advantages and disadvantages. The real difficulties start when trying to unite all three into one API to use as a cross platform library.

{% hint style="warning" %}
Note that while `Kqueue`is used on all BSD based operating systems, there seems to be some subtle differences in flag and constant values between the `macos`version and the one for `FreeBSD`. Just keep this in mind if you're not using a crate which provides this for you already like the [libc crate](https://github.com/rust-lang/libc) does.
{% endhint %}

Creating a cross platform event queue is exactly what the last part of this book will go through.


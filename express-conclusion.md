# Express conclusion

By just running through our express examples you get an overview over what `Kqueue`, `Epoll`and `IOCP`looks like in real life. 

I know it's not a fair comparison based on our express examples since they're so extensively commentated, but in my experience, IOCP is more verbose than the two others and requires a lot more setup to get working. This will be especially apparent when we go through the code in part 2 of our book.

However, event though they're all pretty well documented, IOCP has really the best documentation of them. Go and have a look at the [IOCP](appendix-1/iocp.md),  [Epoll](appendix-1/epoll.md) and [Kqueue](appendix-1/kqueue.md) reference pages in the appendix and have a look yourself.

They all work well and have their advantages and disadvantages. The real difficulties start when trying to unite all three into one API to use as a cross platform library.

{% hint style="warning" %}
Note that while `Kqueue`is used on all BSD based operating systems, there seems to be some subtle differences in flag and constant values between the `macos`version and the one for `FreeBSD`. Just keep this in mind if you're not using a crate which provides this for you already like the [libc crate](https://github.com/rust-lang/libc) does.
{% endhint %}

Creating a cross platform event queue is exactly what the last part of this book will go through.


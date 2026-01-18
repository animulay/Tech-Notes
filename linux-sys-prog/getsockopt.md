
Linux network sockets can be opaque.  We often treat them like simple pipes for data.

Inside, however, the Linux kernel maintains a complex state machine of buffers, timeouts, and protocol flags.

To peek into this hidden state, you need syscall `getsockopt` ([#55 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L67)).

Here's a problem you've felt before:

You click a link. The page hangs. Is the server down? Is your WiFi flaky? Is it just slow?

Your browser doesn't know either - but it can ASK the kernel what's happening with that connection using `getsockopt`.

"But wait - don't read() and connect() already return errors?"

Yes! But here's the thing: high-performance servers don't wait around for connect() to finish.

They mark sockets as non-blocking with fcntl(), then call connect(). Instead of waiting, it returns immediately with EINPROGRESS - "I'm working on it."

The server then juggles thousands of these in-progress connections using poll() or epoll(), doing useful work instead of blocking.

When poll() says a socket is ready, how do you know if the connection succeeded or failed?

You can't call connect() again. That would try to start a NEW connection.

This is where `getsockopt` shines.

`getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &len);`

You're asking: "Hey kernel, did anything go wrong while I was away?"

The kernel hands you the pending error code (or zero if the handshake succeeded), then clears it. Clean and simple.

Beyond error checking, `getsockopt` lets you inspect socket internals.

`SO_RCVBUF` tells you the receive buffer size. `SO_KEEPALIVE` shows if keepalives are enabled.

And TCP_INFO? That's a goldmine - RTT, retransmits, congestion window.

---

- [Uros Popovic: getsockopt thread](https://x.com/popovicu94/status/2005842220386021660)
- [getsockopt(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getsockopt.2.html)

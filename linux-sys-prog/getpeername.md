
You have an open Linux file descriptor. Data is pouring in. But \*who\* is actually sending it?🕵️‍♂️

If your process inherits a connection, you might be flying blind.

Enter `getpeername`, ([syscall #52 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L64)).

Think of it as "Caller ID" for your code.

When you call accept() on a server, you usually get the client's address immediately.

But what if you didn't open the connection? Or if you are inside a library function that only received the file descriptor?

getpeername() queries the kernel's network stack to populate a struct with the remote IP and port associated with that specific integer.

Here is how you verify who is on the other end of the line:
```c
struct sockaddr_in addr;
socklen_t len = sizeof(addr);

// Ask the kernel
if (getpeername(sockfd, (struct sockaddr*)&addr, &len) == 0) {
    printf("Talking to: %s\n",
           inet_ntoa(addr.sin_addr));
}
```
It turns an anonymous data stream into an identified conversation.

Crucially, this works for more than just TCP streams.

If you connect() a UDP socket (which sets a default destination and filters incoming datagrams to that peer), `getpeername` will return that address.

If the socket isn't actually connected to a specific peer? The kernel returns an `ENOTCONN` error.

Why is this syscall essential?

For example, logging: Audit exactly who triggered a request.

Alternatively, service managers: Handle sockets passed into your process from launch tools like inetd or systemd.

It provides the context that read() and write() simply ignore.

---

- [Uros Popovic: getpeername thread](https://x.com/popovicu94/status/2004773680438804657)
- [getpeername(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getpeername.2.html)

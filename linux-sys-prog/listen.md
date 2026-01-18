
A socket has one goal: to connect.

Linux syscall `listen` ([#50 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L62)) changes its destiny.

It marks the socket as "passive".

This is the exact moment a file descriptor transforms into a server. 

The critical part is the backlog argument.

```int listen(int sockfd, int backlog);```

Think of backlog as the waiting room at a busy restaurant.

When the chef is plating dishes, how many customers can wait in the lobby before you start turning people away at the door?

Every server you use daily calls listen().

When nginx starts, it calls listen() to accept web requests.<br>
When sshd starts, it calls listen() to let you log in remotely.<br>
When postgres starts, it calls listen() to accept database queries.<br>

No listen(), no server.

Here's listen() in action: a minimal C server:
```c
int fd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);

bind(fd, (struct sockaddr*)&addr, sizeof(addr));
listen(fd, 128);  // Ready for visitors
```
Three syscalls to open for business: `socket`, `bind`, `listen`.

**"My server drops connections under heavy load!"**

Check `/proc/sys/net/core/somaxconn`.

This is the kernel's hard cap on your backlog.<br> Default is `4096` on modern kernels, but older systems default to just `128`.

You can ask for listen(fd, 10000), but the kernel will silently ignore you.

A common confusion: `listen` vs `read`.

read() pulls bytes from an already-connected client.

listen() never touches data. It just tells the kernel: "I'm open for business."

After listen(), you call accept() to greet each new client, THEN read() their request.

---

- [Uros Popovic: listen thread](https://x.com/popovicu94/status/2004062687874170958)
- [listen(2) — Linux manual page](https://man7.org/linux/man-pages/man2/listen.2.html)

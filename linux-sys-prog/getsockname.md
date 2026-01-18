
Binding to "Port 0" on Linux is a superpower. ⚡️

It tells the kernel: "Give me a port that's free."

It eliminates port conflicts instantly. But it creates a new problem: your program doesn't actually know where it's listening.

This is the job of ([syscall #51](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L63)): `getsockname`.

Usually, servers bind to fixed ports (80, 443, 8080).

But in dynamic environments, like RPC systems, FTP passive modes, or CI/CD test runners, hardcoding ports guarantees **"Address already in use"** errors.

Binding to port 0 delegates the choice to the OS. The kernel assigns an available ephemeral port.

But your port variable in userspace? It still says 0.

To find out what happened, you query the file descriptor.

Here is how to bind to port 0 and discover the assigned port dynamically:

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(0); // Wildcard port

    bind(fd, (struct sockaddr*)&addr, sizeof(addr));

    // Ask the kernel what port we got
    socklen_t len = sizeof(addr);
    getsockname(fd, (struct sockaddr*)&addr, &len);

    printf("Assigned port: %d\n", ntohs(addr.sin_port));
}
```

`getsockname` isn't just for ports. It handles IP addresses too.

If you bind to `0.0.0.0` (`INADDR_ANY`), the socket listens on all interfaces. Calling `getsockname` on the listening socket still returns `0.0.0.0`.

But on a \*connected\* socket, after accept() or connect(), `getsockname` retrieves the specific local IP used for that connection.

(Note: To find who is on the \*other\* end, you use its sibling, ([#52](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L64)) `getpeername`).

---

- [Uros Popovic: getsockname thread](https://x.com/popovicu94/status/2004390146746294569)
- [getsockname(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getsockname.2.html)

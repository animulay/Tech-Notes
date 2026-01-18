
A fresh socket in Linux has a major problem:

It is effectively invisible.

When you call socket(), the kernel creates the endpoint, but assigns no address.

To fix this, servers use syscall `bind`: ([#49 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L61)).

Let's see how it gives your socket an identity.

How is `bind` different from `connect`?

They set opposite ends of the connection:
```
- bind() sets YOUR address (local)
- connect() sets THEIR address (remote)
```
Clients usually skip `bind` entirely. When they connect(), the kernel auto-assigns a local ephemeral port.

Servers must `bind` to a known port so clients can find them.

Here is how a web server claims port 8080.

We create the socket, set up the struct, and ask the kernel to reserve the address.
```c
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    int fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr = {0};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);

    // Bind to the address
    bind(fd, (struct sockaddr*)&addr, sizeof(addr));

    return 0;
}
```
A powerful trick: Binding to Port 0.

If you are writing a service that doesn't need a fixed address (like an RPC worker), set the port to 0.

The Linux kernel interprets this as **"pick any free port for me."** It finds an unused ephemeral port and assigns it.

This largely avoids **"Address already in use"** errors, since the kernel searches for an available port.

---

- [Uros Popovic: bind thread](https://x.com/popovicu94/status/2003695077655478706)
- [bind(2) — Linux manual page](https://man7.org/linux/man-pages/man2/bind.2.html)

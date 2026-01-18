
Linux pipes are legendary, but they have a limitation: they are one-way.

If you need both directions between processes, you use syscall `socketpair` ([#53 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L65)).

It creates two connected sockets instantly. No IPs, no ports, no handshake.

Standard network sockets require a complex dance: `create`, `bind`, `listen`, `connect`.

socketpair() skips all that. It creates two anonymous sockets (usually in the `AF_UNIX` domain) that are already connected.

Pair it with fork(): parent keeps one end, child keeps the other. Instant bidirectional IPC.

`socketpair` has a capability pipes don't: you can pass \*\*File Descriptors\*\* through it.

This means a privileged process can open a restricted file (like a system log) and "hand" the open connection to a sandboxed child process that otherwise couldn't access it.

Here is the C code to spin one up.

No addresses needed. Just kernel-managed internal memory.

```c
#include <sys/socket.h>

int main() {
    int sv[2];
    char buf[3];

    // Create connected pair
    // AF_UNIX = Local internal socket
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) == -1) {
        return 1;
    }

    // sv[0] talks to sv[1]
    // Fully bidirectional
    write(sv[0], "Hey", 3);
    read(sv[1], buf, 3);
}
```
This mechanism (specifically using `sendmsg` with `SCM_RIGHTS`) is the secret sauce of a lot of important software on Linux systems.

It allows a "Broker" process to handle permissions and a "Worker" process to handle data, without the Worker needing root access.

---

- [Uros Popovic: socketpair thread](https://x.com/popovicu94/status/2005130320216293730)
- [socketpair(2) — Linux manual page](https://man7.org/linux/man-pages/man2/socketpair.2.html)


Ever stopped a server on Linux and tried to restart it immediately, only to see "Address already in use"?

The kernel is blocking you. But you can change that.

Check out the syscall `setsockopt` ([#54 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L66)).

Let's see what it's about.

When a TCP connection closes, the port doesn't free up instantly.

The kernel puts it in a `TIME_WAIT` state to catch any late-arriving packets from the old connection.

This ensures reliability, but it is incredibly annoying when you are iterating on code and need to restart your service right now.

`setsockopt` lets you work around this with `SO_REUSEADDR`.

This tells the kernel: "I know there's a lingering connection in `TIME_WAIT`, let me bind anyway."

The old connection still exists and handles its late packets correctly. You're allowing coexistence, not bypassing safety.

```c
int opt = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```
This syscall controls many aspects of network performance.

Default TCP waits to bundle small packets to save bandwidth.

Building a more real-time app? You want raw speed, not efficiency.

Use `setsockopt` with `TCP_NODELAY` to disable that buffering and send data instantly.

---

- [Uros Popovic: setsockopt thread](https://x.com/popovicu94/status/2005457171274981626)
- [getsockopt(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getsockopt.2.html)

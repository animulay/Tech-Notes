
Every program must die. 💀

When your code is done, or crashes spectacularly, on Linux it calls `exit` syscall ([#60 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L72)) to terminate immediately.

But what actually happens in that final moment? Let's trace a program's last breath.

`exit` takes one argument: the exit status.

But here's a detail: only the low 8 bits matter.

The kernel returns `(status & 0xFF)` to the parent. So `exit(256)` looks exactly like `exit(0)` to whoever's waiting.

Your program's last word is just one byte.

When `exit` fires, the kernel:
```
- Closes all open file descriptors
- Reparents any children to init (or the nearest subreaper)
- Sends SIGCHLD to the parent process
```
Here's `exit` at its rawest:

No atexit handlers. No flushing stdout. Instant termination.

The C library's exit (high level function, section 3 in man pages) is polite, it cleans up. The raw syscall is brutal.

```c
#include <unistd.h>
#include <sys/syscall.h>

int main() {
  syscall(SYS_exit, 42);
}
```

Plot twist: since glibc 2.3, even `_exit` (note the leading underscore) secretly calls `exit_group` to kill all threads.

The raw syscall `exit` only terminates the calling thread. The C lib wrapper terminates the whole process.

---

- [Uros Popovic: exit thread](https://x.com/popovicu94/status/2009827147595346269)
- [_exit(2) — Linux manual page](https://man7.org/linux/man-pages/man2/_exit.2.html)

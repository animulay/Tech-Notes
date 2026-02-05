
## creat - Linux syscall #85

`creat` ([#85 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L97)) is a living Linux fossil. It dates back to early UNIX. Ken Thompson famously said his one regret was not spelling it "create."

Calling `creat(path, mode)` is equivalent to: `open(path, O_CREAT | O_WRONLY | O_TRUNC, mode)`

**The catch**: it returns a write-only file descriptor. This creates a classic trap. You create a file, write data, then try to read it back, and the read fails with `EBADF`. You must `close` and `reopen`.

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd = creat("test.txt", 0644);
    write(fd, "hello\n", 6);
    
    char buf[100];
    if (read(fd, buf, sizeof(buf)) == -1)
        perror("read failed");  // EBADF: Bad file descriptor
    
    close(fd);
    return 0;
}
```

Should you ever use `creat`? No. Use `open()` with `O_RDWR | O_CREAT | O_TRUNC` instead. `creat` exists purely for backward compatibility with ancient code.

---

- [Uros Popovic: creat article](https://x.com/popovicu94/status/2018853496532189554)
- [open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)

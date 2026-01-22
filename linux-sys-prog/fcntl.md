
## fcntl - Linux syscall #72

When you open() a file, create a socket(), or set up a pipe on Linux, the kernel assigns it a set of behaviors. These rules are not permanent for the life of that file descriptor.

You can surgically modify a live file descriptor without closing it. This capability is powered by syscall `fcntl`: ([#72 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L84)).

While simple read/write syscalls handle data, `fcntl` handles the \*metadata\* of the file descriptor itself. It is a versatile tool for I/O, capable of duplicating descriptors, managing file locks, and changing I/O modes.

The most critical use case is toggling "Blocking" vs "Non-blocking" I/O. By default, Linux I/O is blocking. If you try to read from a network socket and no data is there, your thread sleeps. It freezes. This is acceptable for a CLI tool, but catastrophic for a high-performance web server handling 10,000 concurrent connections. You can't afford to sleep. You need to check for data, and if it's not there, do something else immediately.

This is where `fcntl` shines. By flipping the `O_NONBLOCK` flag on a running descriptor, you tell the kernel: "If I try to read and the buffer is empty, don't put me to sleep. Just return an error code (`EAGAIN`) instantly." Note: Not all flags can be changed after a file descriptor is created.

Here is a complete, runnable C program that demonstrates this. It takes Standard Input (which usually waits for you to type) and forces it into non-blocking mode.

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int main() {
    // 1. Get the current flags of Standard Input (0)
    // We need to read them first so we don't accidentally wipe
    // other important settings.
    int flags = fcntl(STDIN_FILENO, F_GETFL);
    if (flags == -1) {
        perror("fcntl get");
        return 1;
    }

    // 2. Add the O_NONBLOCK flag to the existing set
    if (fcntl(STDIN_FILENO, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl set");
        return 1;
    }

    printf("STDIN is now non-blocking. Trying to read...\n");

    char buffer[128];
    // This read() would normally pause the program.
    // Now, it returns immediately with -1.
    ssize_t bytes = read(STDIN_FILENO, buffer, sizeof(buffer));

    if (bytes == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("Success! read() returned immediately with EAGAIN.\n");
            printf("The kernel refused to put this process to sleep.\n");
        } else {
            perror("read error");
        }
    } else {
        printf("Read %zd bytes (did you pipe data in?)\n", bytes);
    }

    return 0;
}
```
---

- [Uros Popovic: fcntl](https://x.com/popovicu94/status/2014146560763039949)
- [fcntl(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fcntl.2.html)

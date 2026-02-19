
## fdatasync - Linux syscall #75

Writing files to disk on Linux consists of two operations: writing the content (the bytes), and writing the metadata (timestamps, permissions...).

When your app saves a transaction, it needs to ensure the data hits the physical hardware. The standard tool for this is `fsync`. But fsync comes with a hidden tax. It forces **both** writes to be flushed to disk, every single time.

Even if you just overwrote an existing record, fsync forces the "Last Modified" timestamp change to be persisted to disk. On high-throughput systems, flushing a timestamp update 1000 times a second is a massive waste of I/O resources.

`fdatasync` syscall, [#75 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L87) helps here.

This system call is the performance-obsessed sibling of `fsync`. It flushes your data to the disk but explicitly skips flushing non-essential metadata updates. If your write didn't change the file size (like overwriting a pre-allocated page), `fdatasync` won't bother flushing the modified timestamp.

Here is a complete C program to try it yourself. It writes data and syncs it without forcing a timestamp flush:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    const char *filename = "wal.log";

    // Open for writing, create if missing
    int fd = open(filename, O_WRONLY | O_CREAT, 0644);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    const char *data = "Transaction: COMMIT\n";

    // 1. Write to kernel buffer
    write(fd, data, strlen(data));

    // 2. Force to disk, skipping unnecessary metadata
    if (fdatasync(fd) == -1) {
        perror("fdatasync");
        return 1;
    }

    printf("Data persisted to disk.\n");

    close(fd);
    return 0;
}
```
---

## Reference

- [Uros Popovic: fdatasync article](https://x.com/popovicu94/status/2015237340231541114)
- [fsync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fsync.2.html)

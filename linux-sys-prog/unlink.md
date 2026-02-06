
## unlink - Linux syscall #87

When you "delete" a file on Linux, you rarely destroy the data immediately. You are actually calling `unlink` ([syscall #87 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L99)). This system call removes a specific name from the filesystem hierarchy, but not necessarily the inode (the actual data container). The filesystem follows a strict rule: a file's data is only truly deleted when BOTH conditions are met:

1. The link count reaches zero (no more hard links pointing to the inode)

2. No processes have the file open

This nuance allows for one of the most elegant patterns in Unix programming: The Invisible Temporary File. If you open a file and immediately unlink it, the name vanishes. No other process can see it. `ls` won't list it. But your process? It still holds a valid file descriptor. You can read and write gigabytes of data to this "ghost" file.

The moment your process exits (or crashes!), the open file descriptor is closed. Since the link count is already zero, the kernel sees this and reclaims the disk space. It is the perfect self-cleaning mechanism. No "garbage collection" scripts required.

Here is a complete, runnable C program demonstrating this mechanic:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    const char *name = "ghost.tmp";

    // 1. Create and open the file
    int fd = open(name, O_CREAT | O_RDWR, 0600);
    if (fd == -1) return 1;

    // 2. Immediately delete the name
    // The file is now invisible to 'ls'
    unlink(name);

    // 3. We can still write to it!
    // The data exists, but only we hold the handle.
    write(fd, "Invisible data!", 15);

    // 4. Verify it is gone from the directory
    if (access(name, F_OK) == -1) {
        printf("Proof: File is gone from directory listing.\n");
    }

    // 5. Read the data back
    char buf[32] = {0};
    lseek(fd, 0, SEEK_SET);
    read(fd, buf, 15);
    printf("Read back: %s\n", buf);

    close(fd);
    // Data is finally freed from disk here
    return 0;
}
```

---

## Reference

- [Uros Popovic: unlink article](https://x.com/popovicu94/status/2019589705508221033)
- [unlink(2) — Linux manual page](https://man7.org/linux/man-pages/man2/unlink.2.html)


## fchown - Linux syscall #93

A filename on Linux is just a volatile label. If your program changes file ownership using a path (like the `chown` syscall), it is inherently vulnerable. Why?<br><br>
Because between the time you decide to check a file and the time you actually modify it, the file could be swapped out. This is a classic **TOCTOU** (Time-of-Check Time-of-Use) race condition. An attacker can slide in a symlink to the password file in that split second, and your program might accidentally grant them ownership of a critical system file.

This is why syscall `fchown` ([#93 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L105)) is used.

Unlike `chown`, `fchown` operates on a File Descriptor, not a path. Think of a File Descriptor as a physical anchor. Once you `open()` a file, you have a direct grip on the underlying inode (the kernel's metadata structure that represents the file). The filename could be renamed, moved, or deleted from the directory structure by another process, but your FD remains bound to the specific object you opened. `fchown` ensures you change ownership of exactly the file you intend to, preventing "**imposter**" files from receiving privileges.

Here is a complete, runnable C program demonstrating this secure pattern. Notice how we operate entirely on fd after the initial creation:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/stat.h>

int main() {
    // 1. Create a file with strict permissions (root only initially)
    // We get a File Descriptor (fd) immediately.
    int fd = open("secure.log", O_CREAT | O_WRONLY, 0600);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 2. Securely change ownership to user ID 1000.
    // We use the FD, avoiding path-based race conditions.
    // Even if 'secure.log' is renamed right now, we modify
    // the correct inode.
    if (fchown(fd, 1000, -1) == -1) {
        perror("fchown");
        // Don't forget to close if error handling in real apps
        close(fd);
        return 1;
    }

    printf("Securely changed ownership via syscall 93!\n");

    close(fd);
    return 0;
}
```

---

## Reference

- [Uros Popovic: fchown article](https://x.com/popovicu94/status/2022196418379972902)
- [chown(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chown.2.html)

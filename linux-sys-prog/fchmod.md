
## fchmod - Linux syscall #91

Do you trust your filenames? You shouldn't. 

In Linux, a filename is just a volatile label. It can change in the microseconds between two system calls. This instability leads to a classic security vulnerability known as **TOCTOU** (Time of Check to Time of Use).

Here is the scenario:

1. You create a file: my_file

2. You lock it down: set permissions to 0600

It looks innocent. But between step 1 and 2, an attacker can delete my_file and replace it with a symbolic link pointing to the password file. When your program executes step 2, it unwittingly changes the permissions of the system password file, not your temporary file.

The solution is `fchmod` ([syscall #91 on x86-64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L103)).

Unlike `chmod`, which resolves a path every time, `fchmod` operates on a file descriptor. A file descriptor is a direct, pinned reference to the underlying kernel object (the inode). Once you open a file, you have a stable handle to that specific inode. Even if the filename is renamed, deleted, or swapped for a symlink, your file descriptor remains locked onto the original target.

Here is how to write this pattern securely in C. We create the file and use the resulting descriptor to modify permissions.

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    // 1. Open/Create the file.
    // We get a file descriptor (fd) immediately.
    int fd = open("secure.data", O_RDWR | O_CREAT, 0666);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 2. Write some sensitive data
    write(fd, "Top Secret", 10);

    // 3. SECURE: Change permissions using the fd.
    // Even if an attacker moved "secure.data" or swapped it
    // for a symlink right now, we are pinned to the correct inode.
    if (fchmod(fd, 0600) == -1) {
        perror("fchmod");
        return 1;
    }

    printf("Permissions locked down safely via fchmod.\n");

    close(fd);
    return 0;
}
```

---

## Reference

- [Uros Popovic: fchown article](https://x.com/popovicu94/status/2021466631994790348)
- [chmod(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chmod.2.html)

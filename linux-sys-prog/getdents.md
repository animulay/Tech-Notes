
## getdents - Linux syscall #78

How does `ls` actually list your files? In Unix, everything is a file, and directories are files in a way. A directory can be seen as a special file that maps names to inodes.

But if you try to read() a directory, the kernel blocks you with `EISDIR`. Why? Because directory formats are filesystem-specific. The kernel won't let you see the raw bytes. You need a special key: the `getdents` syscall ([#78 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L90)).

It parses the directory and returns a binary stream of entries. But here is the catch: glibc provides NO wrapper for `getdents`. The function literally does not exist in the standard library. To use it, you have to do it manually:

1. Call it via syscall() directly

2. Manually define the struct (standard headers don't have it)

3. Iterate through a buffer of variable-length records

It's the raw, messy interface. Here is how to implement ls from scratch using the raw system call:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <fcntl.h>
#include <stdio.h>

/* The struct is not in standard headers */
struct linux_dirent {
    unsigned long  d_ino;
    off_t          d_off;
    unsigned short d_reclen;
    char           d_name[];
};

int main() {
    int fd = open(".", O_RDONLY | O_DIRECTORY);
    if (fd == -1) return 1;

    char buf[1024];
    while (1) {
        /* Call getdents directly */
        long nread = syscall(SYS_getdents, fd, buf, 1024);

        if (nread == -1) return 1;
        if (nread == 0) break;

        /* Iterate over variable-length records */
        for (size_t bpos = 0; bpos < nread;) {
            struct linux_dirent *d = (struct linux_dirent *) (buf + bpos);

            /* The file type is in the last byte! */
            char d_type = *(buf + bpos + d->d_reclen - 1);

            printf("%-20s (type: %d)\n", d->d_name, d_type);

            bpos += d->d_reclen;
        }
    }
    close(fd);
    return 0;
}
```

---

## Reference

- [Uros Popovic: getdents article](https://x.com/popovicu94/status/2016333707322523780)
- [getdents(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getdents.2.html)

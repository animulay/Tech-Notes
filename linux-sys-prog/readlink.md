
## readlink - Linux syscall #89

Programs on Linux generally have no idea where they exist on the disk. Once loaded into memory, a process is just a collection of instructions and data. The connection to the original binary file is often abstract.

How does a program find its own executable path, perhaps to load a config file located next to it? It asks the kernel via syscall `readlink`: [#89 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L101).

You might know `readlink` as the mechanism `ls -l` uses to show where a symbolic link points. It reads the target path stored inside the symbolic link. On Linux, its most famous use case involves a bit of magic: `/proc/self/exe`. It is a kernel-maintained symbolic link that always points to the on-disk binary of the currently running process. By reading this link, a program discovers its own identity.

Unlike most string-producing functions in C, however, `readlink` does not null-terminate the output buffer. It simply copies the raw bytes of the path into your buffer and tells you how many bytes it wrote. If you try to pass that buffer to `printf` without manually adding a \0 at the end, you will print garbage memory or crash your application.

Here is a complete, runnable example demonstrating how to safely identify the running binary. Note the explicit manual null-termination.

```c
#include <unistd.h>
#include <stdio.h>
#include <limits.h>

int main() {
    char buf[PATH_MAX];

    // Syscall 89: readlink
    // We reserve 1 byte for the null terminator
    ssize_t len = readlink("/proc/self/exe", buf, sizeof(buf) - 1);

    if (len != -1) {
        // CRITICAL: Manually null-terminate the string
        buf[len] = '\0';
        printf("I am running from: %s\n", buf);
    } else {
        perror("readlink");
    }

    return 0;
}
```

---

## Reference

- [Uros Popovic: readlink article](https://x.com/popovicu94/status/2020265320834015673)
- [readlink(2) — Linux manual page](https://man7.org/linux/man-pages/man2/readlink.2.html)

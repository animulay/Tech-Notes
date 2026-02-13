
## chmod - Linux syscall #90

We rely on .sh or .py to tell us what a file does. But the Linux kernel doesn't care about the extensions. It checks the Mode Bits to decide if a file is allowed to run. But that alone isn't enough; the kernel also inspects the file's contents (like a #! shebang or ELF magic) to figure out how to run it.

This is the domain of syscall `chmod`: [#90 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L102).

It controls the bits that define a file's identity. It differentiates "data" from "program".<br><br>Toggle the execute bit? A text file becomes a command.<br>Toggle the SUID bit? That command runs as root, even if you aren't root.<br><br>This is how sudo gets its power: it's installed with the **SUID** bit set, so `execve` elevates it on launch.

Here is how you manipulate these bits directly in C using the syscall. This program creates a simple script and flips the "Execute" switch programmatically:

```c
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    const char *path = "demo.sh";

    // 1. Create a file with Read/Write only (0600)
    // The kernel sees this as data, not a program
    int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0600);
    if (fd < 0) { perror("open"); return 1; }

    write(fd, "#!/bin/sh\necho 'Hello from Syscall 90!'\n", 40);
    close(fd);

    printf("Created %s. Try running it: Permission denied.\n", path);

    // 2. Use chmod (Syscall 90) to set Read + Write + Execute (0700)
    // S_IRWXU = Read, Write, Execute for Owner
    if (chmod(path, S_IRWXU) == 0) {
        printf("Success: Mode bits updated via syscall 90.\n");
        printf("Run it now with: ./demo.sh\n");
    } else {
        perror("chmod");
    }

    return 0;
}
```

It's a perfect example of how metadata, not content, defines behavior in Linux.

---

## Reference

- [Uros Popovic: chmod article](https://x.com/popovicu94/2020736946918903972)
- [chmod(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chmod.2.html)

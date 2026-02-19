
## lchown - Linux syscall #94

Linux symlinks are transparent traps. Most of the time, the kernel follows them blindly. If you call the `chown()` syscall on a symlink, you're changing the file it points to, not the link. This creates a security nightmare.

If a malicious user plants a link to the password file in a directory, and an admin runs a recursive chown to "fix permissions," the admin just handed over ownership of the system password file. To solve this, Linux provides syscall `lchown`, [#94  on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L106).

It operates on the link itself, without dereferencing the pointer to the destination. It effectively distinguishes "the pointer" from "the thing pointed to."

Here is a runnable C program that demonstrates how to target the link itself:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 1. Create a dummy target file
    int fd = open("target.txt", O_CREAT | O_WRONLY, 0644);
    close(fd);

    // 2. Create a symlink pointing to it
    // "mylink" -> "target.txt"
    symlink("target.txt", "mylink");

    // 3. Change ownership of the LINK, not the target.
    // Standard chown() would affect target.txt here.
    // lchown() only affects mylink.
    // Note: Changing ownership usually requires root (CAP_CHOWN).
    // We use -1 for owner to leave it unchanged, and try
    // to set the group to 1000.
    if (lchown("mylink", -1, 1000) == 0) {
        printf("Success: Changed group of the symlink itself.\n");
    } else {
        perror("lchown failed (try running with sudo)");
    }

    return 0;
}
```
---

## Reference

- [Uros Popovic: lchown article](https://x.com/popovicu94/status/2022547191643124039)
- [chown(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chown.2.html)

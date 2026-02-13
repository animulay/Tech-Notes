
## chown - Linux syscall #92

You probably use `chown` every day to fix permission errors. But syscall `chown` ([#92 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L104)) does something quietly brilliant behind the scenes.

Say you have an executable with the Set-UID bit enabled. That bit means "when someone runs this file, run it as the file's owner, not as them." Now imagine you `chown` that file to root. If nothing else happened, you'd suddenly have a binary that gives root privileges to anyone who runs it.

The kernel catches this. Every time ownership changes, it automatically strips the Set-UID and Set-GID bits. The dangerous bits are gone before anyone can exploit them.

A few things to know when calling `chown` from C:

• You need **CAP_CHOWN** (typically `root`) to change a file's owner.

• But any file owner can change its group to a group they belong to.

• Pass -1 for owner or group to leave that field unchanged.

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <errno.h>

int main() {
    const char *path = "app.bin";

    // Create a dummy file
    int fd = open(path, O_CREAT | O_WRONLY, 0755);
    close(fd);

    // Trigger Syscall 92: chown
    // args: path, new_uid, new_gid
    // -1 means "don't change this part"
    // Note: Changing owner usually requires root
    if (chown(path, 1000, -1) == -1) {
        if (errno == EPERM) {
            printf("Need root privileges to change ownership!\n");
        } else {
            perror("chown error");
        }
    } else {
        printf("Ownership updated successfully.\n");
    }

    return 0;
}
```

---

## Reference

- [Uros Popovic: chown article](https://x.com/popovicu94/status/2021806714723742106)
- [chown(2) — Linux manual page](https://man7.org/linux/man-pages/man2/chown.2.html)

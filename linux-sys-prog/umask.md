
## umask - Linux syscall #95

When you write code on Linux to create a file, you can ask for something like: 0666 (rw-rw-rw-). But on disk, you almost never get that. Who silently altered your request? Your process's own `umask` did. Let's explore syscall `umask`: [#95 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L107).

Every process carries a file mode creation mask, inherited from its parent. When you call `open()` or `mkdir()`, the kernel applies this mask to strip permission bits away. The logic is simple bitwise arithmetic: Final_Mode = Requested_Mode & ~Umask. With the standard umask of 022, the write bits for "group" and "others" are forcibly turned off, ensuring new files aren't world-writable by accident.

(**Note**: if the parent directory has a default ACL, the umask is bypassed entirely and the ACL is used instead.)

But there is a fascinating API design flaw lurking here. For decades, it was impossible to check the current umask without changing it. The syscall prototype is: `mode_t umask(mode_t mask);` It sets the new mask and returns the old one. There is no `getumask()`. To simply read the configuration, libraries had to perform a dangerous dance:

1. old = umask(0); (Disable the mask to read the old value)

2. umask(old); (Quickly restore it)

In theory, if another thread created a file between step 1 and 2, that file would be created with no masking at all and the full requested permissions would be granted. A simple status check introduced a race condition and a potential security hole.

It wasn't until Linux 4.7 that the kernel finally exposed the mask safely via the Umask field in `/proc/self/status`. 

Here is a runnable C program demonstrating the "dangerous dance" required to check your mask, and how it filters your file creation requests:

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    // The "Dangerous Dance" to check the mask
    // We must change it to read it.

    // 1. Set to 0 to get the old value
    mode_t current_mask = umask(0);
    printf("Current umask: %03o\n", current_mask);

    // 2. Restore it immediately
    // In a multi-threaded app, a file created right NOW
    // by another thread would have no masking applied!
    umask(current_mask);

    // Create a file asking for full permissions (0666)
    // The kernel will apply the mask (e.g., 022) to strip bits
    int fd = open("test_file.txt", O_CREAT | O_WRONLY, 0666);

    if (fd != -1) {
        printf("Created 'test_file.txt' asking for 0666.\n");
        printf("Check `ls -l test_file.txt` to see the result.\n");
        close(fd);
    }

    return 0;
}
```

---

## Reference

- [Uros Popovic: umask article](https://x.com/popovicu94/status/2022882475354263567)
- [umask(2) — Linux manual page](https://man7.org/linux/man-pages/man2/umask.2.html)

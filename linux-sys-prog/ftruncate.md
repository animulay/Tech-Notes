
## ftruncate - Linux syscall #77

You can create a 1TB file on a 1GB drive. This feature is called "sparse files," and it can be achieved using the `ftruncate` Linux syscall (number [#77 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L89)) on filesystems that support it.

This system call is very similar to #76, `truncate`. Please check the [post](.//linux-sys-prog/truncate.md) for details. We'll keep this one brief to avoid massive duplication.

When you use `ftruncate` to extend a file beyond its current length, the filesystem does not write gigabytes of zeros to the physical disk. Instead, it simply marks that range as a "hole" in the filesystem metadata. The file reports a massive size, but it consumes almost zero physical blocks on the storage device. If a program later tries to read from that hole, the filesystem intercepts the read request and generates a buffer of null bytes on the fly.

**Note**: Not all filesystems support sparse files. Native Linux filesystems like `ext4`, `XFS`, and `btrfs` do, but `VFAT` does not.

Here is a complete C program that uses `ftruncate` to create a 10GB sparse file in a fraction of a second.

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main() {
    // Open a new file for writing
    int fd = open("sparse.bin", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    // Extend the file to 10GB using ftruncate (#77)
    // Since we haven't written data, this creates a "hole"
    // The filesystem allocates metadata, not physical blocks
    if (ftruncate(fd, 10L * 1024 * 1024 * 1024) < 0) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    printf("Success! Created a 10GB sparse file.\n");
    printf("Verify with:\n");
    printf("  ls -lh sparse.bin  (shows 10GB)\n");
    printf("  du -h sparse.bin   (shows almost 0)\n");

    close(fd);
    return 0;
}
```

---

## Reference

- [Uros Popovic: ftruncate article](https://x.com/popovicu94/status/2015978944558989576)
- [truncate(2) — Linux manual page](https://man7.org/linux/man-pages/man2/truncate.2.html)

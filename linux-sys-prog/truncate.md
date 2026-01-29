
## truncate - Linux syscall #76

A 100GB file that occupies 0 bytes on your disk? Linux filesystems have a capability called "sparse files," and one of the simplest ways to create them is the `truncate` syscall ([number #76 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L88)).

The `truncate` syscall allows you to modify the size of a file explicitly. If you shrink a file, the data is discarded. But if you extend a file beyond its current end, the kernel does something clever. It creates a "hole." Instead of writing gigabytes of actual zeros to the disk, which would be slow and wasteful, the filesystem simply updates the inode metadata. It says, "This file now ends at byte 100,000,000,000."

The space between the old end and the new end is unallocated. When you read from this hole, the operating system detects that no physical block exists and instantly returns a buffer of `null` bytes.

**Note**: sparse file support depends on the filesystem. Native Linux filesystems like `ext4` and `XFS` support this, but some filesystems like `VFAT` do not permit extending files this way at all.

Here is a complete C program to demonstrate this. We create a file and instantly resize it to 10GB without writing data.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char *filename = "sparse.bin";

    // 1. Create an empty file
    int fd = open(filename, O_CREAT | O_WRONLY, 0644);
    if (fd == -1) {
        perror("open failed");
        return 1;
    }
    close(fd);

    // 2. Syscall #76: Resize to 10GB instantly
    // The filesystem creates a "hole" rather than writing zeros
    if (truncate(filename, 10L * 1024 * 1024 * 1024) != 0) {
        perror("truncate failed");
        return 1;
    }

    printf("Created 10GB file: %s\n", filename);
    printf("Verify with:\n");
    printf("  ls -lh %s (shows logical size)\n", filename);
    printf("  du -h %s  (shows physical usage)\n", filename);

    return 0;
}
```

Beyond sparse files, `truncate` can be used in clearing logs. If you delete a log file while a service is writing to it, the disk space isn't freed because the file descriptor is still open. However, if you use `truncate(path, 0)`, it instantly resets the file size to zero, freeing the space while keeping the file handle valid. One caveat: the writing process's file offset isn't reset, so it may continue writing at its previous position, creating a sparse file with a hole at the start. Most well-behaved logging services handle this gracefully.

---

## Reference

- [Uros Popovic: truncate article](https://x.com/popovicu94/status/2015581453061673283)
- [truncate(2) — Linux manual page](https://man7.org/linux/man-pages/man2/truncate.2.html)

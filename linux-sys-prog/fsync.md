
## fsync - Linux syscall #74

When you write to a file on Linux, the kernel may create an illusion.

1. You call write()
   
2. Ther kernel returns "success"

3. Your data is effectively still just sitting in RAM

This is the Page Cache at work. The kernel promises to flush it to the disk "later" (write-back). But if the power fails before "later" happens, your data vanishes.

`fsync` ([syscall #74 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L86)) helps here.

This system call forces the kernel to stop procrastinating. It flushes the modified data AND metadata to the storage device and blocks until the hardware confirms the write is complete.

However, there is a specific detail that may trip you: if you create a NEW file and fsync it, the content is safe, but the **link** to it might not be. The directory entry holding the filename is a separate piece of data. To guarantee a newly created file survives a crash, you must fsync the file descriptor AND the directory's file descriptor.

Here is a complete, runnable C program demonstrating the pattern for durability:

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    // 1. Create and write the file
    int fd = open("critical.data", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return 1;

    write(fd, "Power-loss safe data", 20);

    // 2. Force file data & metadata to disk (Syscall 74)
    fsync(fd);
    close(fd);

    // 3. CRITICAL STEP: Sync the directory
    // Without this, the file 'name' might not exist after a crash
    int dir_fd = open(".", O_RDONLY);
    if (dir_fd >= 0) {
        fsync(dir_fd);
        close(dir_fd);
    }

    printf("Data is physically safe on the platter/flash.\n");
    return 0;
}
```

**Bonus**: if you don't need all metadata synced (e.g., you don't care about atime/mtime), you can use `fdatasync` instead. It skips non-essential metadata, reducing disk activity while still guaranteeing your data is durable.

This specific dance is why writing a truly crash-safe "Save" button or a database engine is a complex engineering challenge. You are constantly fighting the kernel's desire to cache everything for performance.

---

## Reference

- [Uros Popovic: fsync article](https://x.com/popovicu94/status/2014899153130975236)
- [fsync(2) — Linux manual page](https://man7.org/linux/man-pages/man2/fsync.2.html)


## flock - Linux syscall #73

You might assume that locking a file prevents other processes from writing to it. However, Linux file locking is mostly a gentleman's agreement.

`flock` ([#73 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L85)) creates an "advisory" lock. It works exactly like a "Do Not Disturb" sign on a hotel room door. It only stops other processes that explicitly look for the sign before entering. If a rogue process, like a text editor or a simple `echo` command, ignores the lock and writes anyway, the kernel won't stop it. The OS assumes you know what you are doing.

(Note: Semantics can vary with something like the SMB protocol)

It makes `flock` perfect for cooperating processes. The classic example is ensuring a script doesn't run twice simultaneously (a "singleton"). You want the second instance to fail, but you don't want to block system tools like `grep` or `cat` from reading the lock file to check its status.

Here is a complete, runnable C program that implements this "singleton" pattern. It tries to acquire an exclusive lock on a file. If you compile this and run it in two different terminals at the same time, the second instance will see the lock, print an error, and exit, while the first one keeps running.

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/file.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

int main() {
    // Open a file to serve as the lock
    int fd = open("app.lock", O_WRONLY | O_CREAT, 0666);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    printf("Attempting to acquire exclusive lock...\n");

    // LOCK_EX = Exclusive Lock
    // LOCK_NB = Non-blocking (fail immediately if locked)
    if (flock(fd, LOCK_EX | LOCK_NB) == -1) {
        if (errno == EWOULDBLOCK) {
            fprintf(stderr, "Error: Another instance is already running!\n");
            return 1;
        }
        perror("flock");
        return 1;
    }

    printf("Lock acquired! I am the only instance running.\n");
    printf("Press Enter to release lock and exit...");
    getchar();

    // Explicit unlock (optional, close() releases it too)
    flock(fd, LOCK_UN);
    close(fd);

    return 0;
}
```
One gotcha: fork() and dup() share the same lock: a child process can accidentally release the parent's lock by closing its copy of the fd.

---
## Reference

- [Uros Popovic: flock article](https://x.com/popovicu94/status/2014531004153922008)
- [flock(2) — Linux manual page](https://man7.org/linux/man-pages/man2/flock.2.html)
- [Singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern)

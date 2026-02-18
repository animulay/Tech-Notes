
## getrlimit - Linux syscall #97

Every Linux process lives inside a cage, but it usually just doesn't hit the bars. Syscall `getrlimit` ([#97 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L109)) explores those bars.

We can write code without worries about RAM, CPU cycles, and file handles. But to keep a single buggy loop from crashing the entire server, the kernel enforces strict quotas on every process. This syscall reveals those invisible boundaries. The design features a clever distinction between "Soft" and "Hard" limits:

1. Soft Limit: The actual current limit enforced by the kernel. If you exceed it, the consequence depends on the resource: you might get an error (`EMFILE` for too many open files, `ENOMEM` for address space), or a signal (`SIGXCPU` for CPU time).

2. Hard Limit: The ceiling for the Soft limit. It controls how high an unprivileged process can raise its own Soft limit.

There is a brilliant security detail here. **An unprivileged process can raise its Soft limit up to the Hard limit, and can voluntarily lower its own Hard limit, locking the door from the inside, but it can never raise the Hard limit back up**. Only a privileged process (one with the `CAP_SYS_RESOURCE` capability) can make arbitrary changes to either limit.

Here is a complete C program to inspect your own file descriptor limits:

```c
#include <stdio.h>
#include <sys/resource.h>

int main() {
    struct rlimit limit;

    // 97 is getrlimit. Let's check RLIMIT_NOFILE (7)
    // allowing us to see max open files.
    if (getrlimit(RLIMIT_NOFILE, &limit) == 0) {
        printf("Open Files Quota:\n");

        // Soft limit: The current enforcement
        printf("  Soft: %lu\n", (unsigned long)limit.rlim_cur);

        // Hard limit: The absolute ceiling
        printf("  Hard: %lu\n", (unsigned long)limit.rlim_max);
    }

    return 0;
}
```
---

## Reference

- [Uros Popovic: getrlimit article](https://x.com/popovicu94/status/2023975406676947376)
- [getrlimit(2) — Linux manual page](https://man7.org/linux/man-pages/man2/getrlimit.2.html)

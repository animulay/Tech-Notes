
## gettimeofday - Linux syscall #96

System call `gettimeofday` ([#96 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L108)) is a call that tries its absolute hardest not to be a system call. Normally, interacting with the kernel triggers a context switch.

1. Your app pauses.

2. The CPU switches privilege levels (User -> Kernel).

3. The kernel does the work.

4. The CPU switches back.

This is expensive. If you are logging every request, stamping every database transaction, or measuring metrics, this overhead adds up fast. But time is unique. It is global, read-only data. To solve this, Linux uses a mechanism called vDSO (virtual Dynamic Shared Object). Instead of forcing you to call into the kernel, the kernel brings the answer to you. It maps a special shared library directly into every process's address space. This library contains code and read-only data maintained by the kernel.

The data includes a base timestamp and parameters from the kernel's last timer tick. The code knows how to read the CPU's hardware cycle counter and combine it with those parameters to compute the exact current time, all without ever leaving userspace. When you call `gettimeofday`, the C library often redirects your call into this vDSO code. It runs entirely in your process's address space as a userspace function call that reads a hardware counter and does a little math.

Here is a simple C program to see the call in action.

```c
#include <stdio.h>
#include <sys/time.h>
#include <unistd.h>

int main() {
    struct timeval tv;

    // This call is intercepted by the vDSO.
    // No ring transition occurs!
    if (gettimeofday(&tv, NULL) == 0) {
        printf("Seconds since Epoch: %ld\n", (long)tv.tv_sec);
        printf("Microseconds: %ld\n", (long)tv.tv_usec);
    }

    return 0;
}
```
A quick note on modern standards: `gettimeofday` is more than just "technically obsolete." POSIX.1-2008 marked it obsolete, and POSIX.1-2024 removed it from the standard entirely. You should prefer `clock_gettime` for new code.

Note: The vDSO optimization is architecture-dependent. It is present on x86_64 and several other architectures, but is not guaranteed on every platform Linux supports.

---

## Reference

- [Uros Popovic: gettimeofday article](https://x.com/popovicu94/status/2023640695379292279)
- [gettimeofday(2) — Linux manual page](https://man7.org/linux/man-pages/man2/gettimeofday.2.html)

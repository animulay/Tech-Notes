
`fork` on Linux protects the parent. `vfork` ([syscall #58, x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L70)) hijacks it.

vfork disables Copy-On-Write protections entirely.

It suspends the parent and lets the child run in the parent's memory space.

Why would Linux allow that?

When you call vfork():

1. The kernel suspends the parent process completely.
2. The child process starts running using the parent's existing memory pages.
3. The parent stays frozen until the child calls execve() or _exit().

It's a temporary takeover.

⚠️ vfork() was removed from POSIX in 2008, but Linux preserves the historical contract: after vfork(), the child should only call _exit() or execve(). Anything else is undefined behavior.

Why _exit() and not exit()?

_exit() is a thin wrapper around the syscall. It terminates immediately.

exit() is a C library function that flushes stdio buffers and runs `atexit` handlers, in the parent's memory. The parent may later flush the same buffers again, causing duplicated or corrupted output.

There's another danger: the child must never return from the function that called vfork()!

Since the child shares the parent's stack, returning would pop the stack frame. When the parent wakes up, its return address and local variables are gone.

The parent crashes.

The child modifies val, and the parent sees it.

Actually, it's tricky! Likely undefined. [Man page:](https://man7.org/linux/man-pages/man2/vfork.2.html)

"The use of vfork() was tricky: for example, not modifying data in the parent process depended on knowing which variables were held in a register."

```c
#include <unistd.h>
#include <stdio.h>

int main() {
  int val = 10;

  if (vfork() == 0) {
    // Child runs in parent's memory
    val = 999;  // Undefined behavior! Don't do this
    _exit(0);
  }

  // Parent may see the change (undefined behavior!)
  printf("val: %d\n", val);
  return 0;
}
```

---

- [Uros Popovic: vfork thread](https://x.com/popovicu94/status/2009087220440027419)
- [vfork(2) — Linux manual page](https://man7.org/linux/man-pages/man2/vfork.2.html)

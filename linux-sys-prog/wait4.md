
## wait4 - Linux syscall #61

When a process dies on Linux, it still holds a slot in the kernel's process table. Complete, but visible, waiting for its parent to acknowledge it.

To clean this up, parents use a powerful syscall: `wait4` ([#61 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L73)).

Standard `wait` calls reap the zombie and return its exit status.

`wait4` does all that, but also returns a detailed "receipt."

Through the `struct rusage` argument, the kernel hands the parent an itemized invoice of exactly what resources the child consumed during its life.

Here is how to catch a dying process and read its stats.

We fork a child, let it run, and then use `wait4` to retrieve the execution time directly from the kernel.

```c
#include <sys/wait.h>
#include <sys/resource.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  if (fork() == 0) {
    // Child: burn some cycles
    for(volatile int i=0; i<10000000; i++);
    return 0l
  }

  struct rusage ru;
  // Wait for any child (-1) and fill ru
  wait4(-1, NULL, 0, &ru);

  printf("User CPU: %ld.%06ld s\n",
          ru.ru_utime.tv_sec,
          ru.ru_utime.tvusec);
}
```
That `rusage` struct is incredibly detailed. It tracks:

1. CPU time (User vs System)<br>
2. Memory page faults<br>
3. Context switches<br>
4. Block IO operations<br>

This is how your shell gets CPU stats after running 'time cmd'. The shell times the wall clock separately, but the kernel's bill gives it the real CPU breakdown at the moment of death.

```c
           struct rusage {
               struct timeval ru_utime; /* user CPU time used */
               struct timeval ru_stime; /* system CPU time used */
               long   ru_maxrss;        /* maximum resident set size */
               long   ru_ixrss;         /* integral shared memory size */
               long   ru_idrss;         /* integral unshared data size */
               long   ru_isrss;         /* integral unshared stack size */
               long   ru_minflt;        /* page reclaims (soft page faults) */
               long   ru_majflt;        /* page faults (hard page faults) */
               long   ru_nswap;         /* swaps */
               long   ru_inblock;       /* block input operations */
               long   ru_oublock;       /* block output operations */
               long   ru_msgsnd;        /* IPC messages sent */
               long   ru_msgrcv;        /* IPC messages received */
               long   ru_nsignals;      /* signals received */
               long   ru_nvcsw;         /* voluntary context switches */
               long   ru_nivcsw;        /* involuntary context switches */
           };
```

`wait4` is a BSD-ism dating back to 4.3BSD and it's not POSIX.

The man page even says "in new programs, the use of `waitpid(2)` or `waitid(2)` is preferable."

But Linux keeps it around, and it remains the cleanest way to get lifecycle management and resource accounting in a single syscall.

---

## Reference

- [Uros Popovic: wait4 thread](https://x.com/popovicu94/status/2010190293723951485)
- [wait4(2) — Linux manual page](https://man7.org/linux/man-pages/man2/wait3.2.html)

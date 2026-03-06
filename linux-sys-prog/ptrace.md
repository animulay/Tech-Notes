
## ptrace - Linux syscall #101

Linux processes are designed to be isolated fortresses. Memory is virtualized, and one process usually cannot touch another. But there is a deliberate back door key that unlocks them all: syscall `ptrace` ([#101 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L113)).

This system call allows one process to "be a controller" over another. The "tracer" can pause the "tracee," read or write its memory, and modify CPU registers on the fly. This is the engine behind tools like `gdb` and `strace`.

Here is a complete, runnable C program that builds a tiny version of `strace`. It spawns `ls` as a child process and uses `ptrace` to intercept and print the ID of every system call `ls` makes.

```c
#include <sys/ptrace.h>
#include <sys/wait.h>
#include <sys/user.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/syscall.h>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // Child process: The "Tracee"
        // PTRACE_TRACEME tells the kernel: "Let my parent trace me"
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);

        // Execute a program. The kernel will stop the child
        // at the execve boundary so the parent can begin tracing.
        execl("/bin/ls", "ls", NULL);
    } else {
        // Parent process: The "Tracer"
        int status;
        int in_syscall = 0;

        // Wait for the child to stop at the exec
        waitpid(pid, &status, 0);

        while (WIFSTOPPED(status)) {
            if (in_syscall == 0) {
                struct user_regs_struct regs;

                // Inspect the CPU registers of the child
                ptrace(PTRACE_GETREGS, pid, NULL, &regs);

                // On x86_64, orig_rax holds the system call number
                printf("Syscall triggered: %llu\n", regs.orig_rax);
            }

            // Toggle: PTRACE_SYSCALL stops at both entry AND exit
            // of each syscall, so we alternate to avoid duplicates
            in_syscall = !in_syscall;

            // PTRACE_SYSCALL resumes execution but stops at the
            // next syscall entry or exit
            ptrace(PTRACE_SYSCALL, pid, NULL, NULL);

            // Wait for the next stop
            waitpid(pid, &status, 0);
        }
    }
    return 0;
}
```

---

- [Uros Popovic: ptrace article](https://x.com/popovicu94/status/2026166602711097849)
- [ptrace(2) — Linux manual page](https://man7.org/linux/man-pages/man2/ptrace.2.html)

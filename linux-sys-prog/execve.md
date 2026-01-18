
Running a command on Linux is a brain transplant. 🧠

When you type `ls`, your shell first creates a clone of itself. Then, the clone calls syscall `execve` ([#59 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L71)).

The clone's memory is wiped but the Process ID remains.

The identity stays. The mind is replaced.

Linux decouples "creating the process" and "running a program":

1. fork() duplicates the current process.
2. execve() replaces the running code with a new binary.

This separation allows the shell to configure pipes and redirection between the two steps.

Because `execve` overwrites the process's memory, it has an interesting property: it never returns on success.

The code that called it ceases to exist.

```c
#include <unistd.h>

int main() {
  char *args[] = {"/usr/bin/ls", NULL};
  char *env[] = {NULL};

  execve("/usr/bin/ls", args, env);

  // This line is unreachable!
  // The process is now 'ls'.
  return 1;
}
```

This syscall is also why scripts work seamlessly.

When the kernel opens a file for `execve` and sees the '#!' bytes (the shebang), it reads the interpreter path (like `/usr/bin/python3`) and executes that instead.

The kernel transparently swaps the binary before userspace notices.

The "brain transplant" model is vital for Linux piping.

File descriptors generally survive `execve`, so the shell can connect stdout to a pipe \*before\* loading the new program.

When a program starts, it just writes to file descriptor 1, unaware it's talking to grep instead of a screen.

---

- [Uros Popovic: execve thread](https://x.com/popovicu94/status/2009463454852370692)
- [execve(2) — Linux manual page](https://man7.org/linux/man-pages/man2/execve.2.html)

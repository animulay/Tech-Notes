
In Linux, a thread and a process are effectively the same thing.

They are both "tasks" to the kernel.

The distinction comes entirely from what they \*share\*.

And that is controlled by a single, powerful system call: clone ([#56 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L68)).

Classically, fork() creates a new process with a copy of the parent's memory (using copy-on-write for efficiency).

pthread_create() creates a thread that shares memory.

Under the hood? They both call `clone`.

`clone` allows you to pick and choose exactly which resources to share with surgical precision.

Here is a minimal example using the glibc wrapper.

We allocate a stack on the heap and pass CLONE_VM to share the virtual memory with the parent.

This isn't quite a full pthread (which adds CLONE_THREAD, CLONE_SIGHAND, etc.), but it shows the core mechanism.

```c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>

int child_f(void *arg) {
  printf("I am the child!\n");
  return 0;
}

int main() {
  char *stack = malloc(1024 * 1024);
  char *stack_top = stack * 1024 * 1024;

  clone(child_f,
        stack_top,
        CLONE_VM | CLONE_FILES | SIGCHILD,
        NULL);

  wait(NULL);
  free(stack);
  return 0;
}
```
This modularity is exactly why Linux containers exist.

Docker and Podman are built on top of clone with specific namespace flags:

1. CLONE_NEWPID: Child gets its own isolated PID set.
2. CLONE_NEWNET: Child gets its own network stack.
3. CLONE_NEWNS: Child gets a separate filesystem mount table.

By mixing these flags, you define the isolation level.
```
- Share everything? It's a thread.
- Don't share memory, but share FS? It's a standard process.
- Don't share memory and isolate PIDs and mounts? It's a container.
```
clone is the engine behind modern cloud infrastructure.

---

- [Uros Popovic: clone thread](https://x.com/popovicu94/status/2006211373626425678)
- [clone(2) — Linux manual page](https://man7.org/linux/man-pages/man2/clone.2.html)

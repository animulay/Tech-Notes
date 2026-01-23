
## semctl - Linux syscall #66

`semget` creates semaphores. `semop` operates on them.

But how do you check their value? Change it directly? Or destroy them when you're done?

Meet `semctl` ([syscall #66](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L78)).

It's the control panel for System V semaphores on Linux.

`semctl` can do a lot:
```
- GETVAL: read a semaphore's current value
- SETVAL: set it directly (useful for initialization)
- IPC_STAT: get metadata (permissions, timestamps)
- IPC_RMID: destroy the semaphore set entirely
```
It's the Swiss Army knife you need after `semget` and `semop`.

Here's the catch: System V semaphores are kernel-persistent objects.

When your process exits, the kernel frees its memory, closes its files, and cleans up its sockets.

But the semaphore set? It stays. It belongs to the system, not your process.

This is by design, but it's also a trap.

If you decrement a semaphore (lock it) and your process crashes, what happens?

With `SEM_UNDO`: the kernel reverses the operation automatically. Crisis averted.

Without `SEM_UNDO`: the semaphore stays decremented. Other processes wait forever for a signal that will never come.

The resource AND the lock become ghosts. 👻

Here's a complete example. Proper creation, use, and cleanup.

```c
#include <stdio.h>
#include <sys/sem.h>

int main() {
  int id = semget(IPC_PRIVATE, 1, 0600);

  semctl(id, 0, SETVAL, 1); // Initialize to 1

  printf("Value: %d\n", semctl(id, 0, GETVAL));

  // Critical: destroy when doen
  semctl(id, 0, IPC_RMID);
  printf("Semaphore removed.\n");

  return 0;
}
```
Forget to call `IPC_RMID`? Now you've leaked a kernel resource.

Your sysadmin will have to hunt it down:
```
$ ipcs -s          # Find orphaned semaphores
$ ipcrm -s <id>    # Manually delete them
```
Clean up your semaphores, or use `SEM_UNDO` and let the kernel help.

---
## Reference

- [Uros Popovic: semctl post](https://x.com/popovicu94/status/2011994571585249578)
- [semctl(2) — Linux manual page](https://man7.org/linux/man-pages/man2/semctl.2.html)

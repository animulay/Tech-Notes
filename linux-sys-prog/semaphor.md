System V semaphores are one of the oldest synchronization primitives in Unix.

They've been around since the 1980s, and Linux still supports them today.

Meet syscall `semget` ([#64, x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L76)).

It creates or accesses a semaphore set, but there's a wrinkle.

`semget` creates the semaphore set, but doesn't initialize the values.

POSIX says the initial values are "indeterminate."

Linux actually zeroes them, but you can't rely on this for portable code.

Either way, 0 probably isn't what you want (1 = "unlocked" for a mutex).

You need a second call: `semctl` with `SETVAL`.

This two-step process creates a classic race condition.

Think of it like installing a traffic light:

Step 1: Mount the light on the pole (`semget`)<br>
Step 2: Wire it up and turn it on (`semctl`)

If a car arrives between steps, it sees a dark signal. Does it stop? Go? Flip a coin?

Your processes face the same dilemma.

Here's the code that causes the race.

If another process attaches during the gap, it sees the wrong state.

```c
#include <sys/sem.h>

int main() {
  int id = semget(key, 1, IPC_CREAT | 0666);

  // DANGER ZONE
  // Semaphore exists but value is
  // uninitialized (0 on Linux, but
  // not what we want for a lock)

  semctl(id, 0, SETVAL, 1);  // Set to "unlocked"
}
```

The kernel gives us a clever workaround.

The `sem_otime` field (last operation time) starts at 0 and only updates after the first semop() call.

Processes can poll `sem_otime` via `semctl(IPC_STAT)`.

If it's still 0, the creator hasn't finished setup yet: so wait and retry.

It's a hack, but it works.

---

## Reference

- [Uros Popovic: System V semaphores thread](https://x.com/popovicu94/status/2011280120162738384)
- [semget(2) — Linux manual page](https://man7.org/linux/man-pages/man2/semget.2.html)

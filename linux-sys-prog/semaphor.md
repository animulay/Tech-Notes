System V semaphores are one of the oldest synchronization primitives in Unix.<br>

They've been around since the 1980s, and Linux still supports them today.<br>

Meet syscall _semget_ (#64, x86_64).<br>

It creates or accesses a semaphore set, but there's a wrinkle.<br>

_semget_ creates the semaphore set, but doesn't initialize the values.<br>

POSIX says the initial values are "indeterminate."<br>

Linux actually zeroes them, but you can't rely on this for portable code.<br>

Either way, 0 probably isn't what you want (1 = "unlocked" for a mutex).<br>

You need a second call: _semctl_ with SETVAL.<br>

This two-step process creates a classic race condition.<br>

Think of it like installing a traffic light:<br>

Step 1: Mount the light on the pole (semget)<br>
Step 2: Wire it up and turn it on (semctl)<br>

If a car arrives between steps, it sees a dark signal. Does it stop? Go? Flip a coin?<br>

Your processes face the same dilemma.<br>

Here's the code that causes the race.<br>

If another process attaches during the gap, it sees the wrong state.<br>

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

The kernel gives us a clever workaround.<br>

The _sem_otime_ field (last operation time) starts at 0 and only updates after the first semop() call.<br>

Processes can poll _sem_otime_ via semctl(IPC_STAT).<br>

If it's still 0, the creator hasn't finished setup yet: so wait and retry.<br>

It's a hack, but it works.<br>

---

## Reference

- [Uros Popovic: System V semaphores thread](https://x.com/popovicu94/status/2011280120162738384)
- [semget(2) — Linux manual page](https://man7.org/linux/man-pages/man2/semget.2.html)

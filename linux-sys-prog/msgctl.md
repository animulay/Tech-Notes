
## msgctl - Linux syscall ([#71](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L83))

Pipes vanish when processes die but the System V message queues don't. They persist in the kernel until explicitly removed. This is why you often see "orphaned" IPC objects cluttering servers after a crash. The kernel keeps them alive until explicitly told otherwise.

`msgctl` is the control center for these objects.

While `msgget` creates the queue and `msgsnd`/`msgrcv` use it, `msgctl` is what manages its lifecycle. It allows you to modify permissions, increase size limits, or check `msg_qnum` to see if your consumer process is falling behind. But its most distinct feature is destruction (`IPC_RMID`). Deleting a queue isn't a passive cleanup operation; it is an active event. When you call `msgctl` with `IPC_RMID`, the kernel hunts down every process sleeping on that queue: all waiting readers and writers. It wakes them all up immediately with an error (`EIDRM`). It serves as a master switch to unblock and terminate a distributed system.

Here is a complete, runnable C program that creates a private queue, inspects its status, and then actively destroys it so no orphans are left behind.

```c
#include <stdio.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <errno.h>

int main() {
    // 1. Create a private queue
    // IPC_PRIVATE creates a new queue. 0600 gives read/write to owner.
    int qid = msgget(IPC_PRIVATE, 0600);
    if (qid == -1) {
        perror("msgget failed");
        return 1;
    }
    printf("Queue created with ID: %d\n", qid);

    // 2. Check queue status (IPC_STAT)
    struct msqid_ds buf;
    if (msgctl(qid, IPC_STAT, &buf) == 0) {
        printf("Current messages: %lu\n", buf.msg_qnum);
        printf("Max bytes allowed: %lu\n", buf.msg_qbytes);
    } else {
        perror("msgctl stat failed");
    }

    // 3. Destroy the queue (IPC_RMID)
    // This is crucial. If we don't do this, the queue stays in RAM
    // even after this program exits!
    if (msgctl(qid, IPC_RMID, NULL) == 0) {
        printf("Queue destroyed. No orphans left.\n");
    } else {
        perror("msgctl removal failed");
    }

    return 0;
}
```
---

- [Uros Popovic: msgctl article](https://x.com/popovicu94/status/2013775065578643642)
- [msgctl(2) — Linux manual page](https://man7.org/linux/man-pages/man2/msgctl.2.html)

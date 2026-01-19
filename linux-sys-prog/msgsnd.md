
## msgsnd - Linux syscall #69

Linux pipes are the backbone of the command line, but they suffer from a rigid constraint: They are strictly `FIFO`. Once you write data into a pipe, it is trapped behind everything written before it. There is no "fast lane" for urgent data.

Syscall `msgsnd` ([#69 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L81)) addresses this.

It introduces the concept of Typed Messages to Inter-Process Communication. Unlike a pipe, which treats data as a raw stream of bytes, `msgsnd` transmits distinct packets. Crucially, every packet implies a structure containing a positive numeric label (a long integer type).

This simple integer changes the physics of the queue. When the receiver (using the counterpart syscall `msgrcv`) wakes up, it doesn't have to read sequentially. It can ask the Linux kernel: "Give me the first message of Type 1." The kernel will traverse the internal linked list, skip over any "Type 2" messages, and return the first "Type 1" message it finds. This effectively creates a kernel-managed queue with type-based message selection between processes.

## Example

Imagine a print server daemon.

If you use a standard pipe, a 500-page report sent by an intern blocks the 1-page urgent invoice sent by the manager.

With `msgsnd`:

- Intern's Report = Type 2
- Manager's Invoice = Type 1

The printer daemon requests Type 1 messages first. The Invoice jumps the queue, regardless of arrival time.

## Code

Here is a complete, runnable C program that creates a queue and sends two messages with different types.

Note the `struct msg_buffer`: The Linux kernel mandates that the first field is a long representing the message type.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/msg.h>

// The kernel requires a struct starting with a long
struct msg_buffer {
    long msg_type;
    char msg_text[100];
};

int main() {
    // 1. Create a file to generate a unique key
    system("touch msgqfile");
    key_t key = ftok("msgqfile", 65);

    // 2. Create a message queue
    // 0666 = Read/Write permissions
    int msgid = msgget(key, 0666 | IPC_CREAT);
    if (msgid == -1) {
        perror("msgget failed");
        return 1;
    }

    struct msg_buffer message;

    // 3. Send a LOW PRIORITY message (Type 2)
    message.msg_type = 2;
    strcpy(message.msg_text, "Heavy report (print later)");

    // Syscall 69: msgsnd
    if (msgsnd(msgid, &message, sizeof(message.msg_text), 0) == -1) {
        perror("msgsnd failed");
    }
    printf("Sent: %s [Type %ld]\n", message.msg_text, message.msg_type);

    // 4. Send a HIGH PRIORITY message (Type 1)
    message.msg_type = 1;
    strcpy(message.msg_text, "URGENT MEMO");

    // Syscall 69: msgsnd
    if (msgsnd(msgid, &message, sizeof(message.msg_text), 0) == -1) {
        perror("msgsnd failed");
    }
    printf("Sent: %s [Type %ld]\n", message.msg_text, message.msg_type);

    printf("\nQueue ready. A reader can now request Type 1 first.\n");

    return 0;
}
```



---

- [Uros Popovic: msgsnd article](https://x.com/popovicu94/status/2013013179354533961)
- [msgop(2) — Linux manual page](https://man7.org/linux/man-pages/man2/msgsnd.2.html)

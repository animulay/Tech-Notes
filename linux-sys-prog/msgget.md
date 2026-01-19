
Linux processes need to talk, but pipes can feel limited, and sockets add networking overhead.

Sometimes you just need a simple, persistent "inbox" where any process can drop a note. That's where syscall `msgget` ([#68 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L80)) comes in.

Think of a restaurant kitchen. A waiter places an order ticket on a rail; the chef grabs it. They don't need to interrupt each other or be synchronized. They just need a shared "drop-off point."

`msgget` creates that rail: a System V Message Queue.

You provide a key, and the kernel gives you a Queue ID. Any process with that ID (and the right permissions) can push or pull messages.

But there is a famous trap in this API. To ensure you create a brand new queue, you pass the key `IPC_PRIVATE`. You might assume this creates a secure, private channel restricted to your process tree. It does not. It simply guarantees a \*new\* queue is created with a unique key. Any other process on the system that finds the ID can still access it. The official Linux man page actually apologizes for this! It calls the name "unfortunate" and suggests it should have been named `IPC_NEW`. It is a lasting reminder that in software engineering, naming really is the hardest problem.

Here is a complete, runnable C program that creates a queue using this "private" key, sends a message to itself, and reads it back.

```c
#include <sys/ipc.h>
#include <sys/msg.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

// Structure for the message
struct msg_buffer {
    long msg_type;
    char msg_text[100];
} message;

int main() {
    // 1. Create a queue
    // IPC_PRIVATE = guarantee a new queue.
    // 0666 = Read/Write permission for everyone.
    int msgid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

    if (msgid == -1) {
        perror("msgget failed");
        return 1;
    }

    printf("Queue created with ID: %d\n", msgid);

    // 2. Send a message to the queue
    message.msg_type = 1; // Must be > 0
    strcpy(message.msg_text, "Hello from the kernel queue!");

    // msgsnd(id, msg_struct, size_of_text, flags)
    msgsnd(msgid, &message, sizeof(message.msg_text), 0);
    printf("Message sent.\n");

    // 3. Read the message back
    // msgrcv(id, msg_struct, size, type, flags)
    msgrcv(msgid, &message, sizeof(message.msg_text), 1, 0);
    printf("Received: %s\n", message.msg_text);

    // 4. Cleanup: Remove the queue
    msgctl(msgid, IPC_RMID, NULL);

    return 0;
}
```

---

- [Uros Popovic: msgget](https://x.com/popovicu94/status/2012766894399303842)
- [msgget(2) — Linux manual page](https://man7.org/linux/man-pages/man2/msgget.2.html)


The `kill` Linux syscall ([#62 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L74)) is a general messaging system, not just a weapon to destroy frozen apps.

We associate it with force-quits, but it handles everything from pausing execution to reloading configurations.

Let's look at this misunderstood syscall.

Why is it called "kill"?

When you run kill without specifying a signal, it sends `SIGTERM` by default, a request to terminate.

Early Unix needed a way to stop runaway processes, so the most common use was termination.

The name stuck, even as the syscall evolved into a general-purpose signaling mechanism.

Changing it would break decades of code.

When you call `kill`, you pass a Signal ID.

The kernel delivers it, and the target reacts:
```
- SIGSTOP: Freezes the process (Pause)
- SIGCONT: Unfreezes it (Resume)
- SIGHUP: Often tells daemons to reload config
- SIGINT: Equivalent to pressing Ctrl+C
```

It is the fundamental way processes talk to each other.

Here is a powerful trick: Signal 0.

If you run kill with signal 0, the kernel sends absolutely nothing.

However, it still checks permissions and existence.

It is maybe the fastest way to check "is this process still running?" from code. No overhead of parsing `/proc` or running `ps`.

One final superpower: Broadcasting.

You don't always need a specific PID.

If you pass pid = -1, the kernel sends your signal to EVERY process you have permission to touch (except init and the caller itself).

This is how shutdown scripts instantly terminate an entire user session without iterating through IDs.

---

- [Uros Popovic: kill thread](https://x.com/popovicu94/status/2010510197756948722)
- [kill(2) — Linux manual page](https://man7.org/linux/man-pages/man2/kill.2.html)

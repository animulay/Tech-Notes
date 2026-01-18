Your Linux system's identity is just a C struct.

We commonly run `uname -a` to check our kernel version. Under the hood, this triggers a specific syscall: _uname_ ( [#63 on x86_64](https://github.com/torvalds/linux/blob/master/arch/x86/entry/syscalls/syscall_64.tbl#L75)).

It's the fundamental way software answers the question: "Who am I?"

When called, the kernel fills a `struct utsname` with details.

Most fields are fixed for a given kernel binary:
```
- sysname (always "Linux")
- release (e.g., 6.15.0)
- version (build timestamp, config)
- machine (determined by architecture)
```
These won't change at runtime.

The nodename (hostname) is different.

The kernel doesn't inherently know its own name. It treats the hostname as a simple, mutable string in memory.

It relies entirely on userspace (like systemd or init scripts) to set this name during boot. `uname` simply reports back what it was told earlier.

Here is how to extract this identity directly in C.

It is much cleaner than parsing string output from command line tools.

```c
#include <stdio.h>
#include <sys/utsname.h>

int main() {
  struct utsname u;
  if (uname(&u) == 0) {
    printf("I am %s running kernel %s\n",
            u.nodename, u.release);
  }
  return 0;
}
```

---

- [Uros Popovic: uname thread](https://x.com/popovicu94/status/2010910202875498842)
- [uname(2) — Linux manual page](https://man7.org/linux/man-pages/man2/uname.2.html)

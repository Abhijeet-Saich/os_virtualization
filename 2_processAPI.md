# OSTEP — Process API: Interview Revision Notes

## 1. Process

A **program** is a passive executable; a **process** is a running instance of a program.

A process has:
- PID
- Address space / memory
- CPU registers and program counter
- Open file descriptors
- Execution state

**Intuition:** Program = recipe; Process = recipe being executed.

---

## 2. `fork()`

```c
pid_t rc = fork();
```

`fork()` creates a new process that is a near-copy of the calling process.

Both processes continue **from the instruction after `fork()`**, not from `main()`.

Return value:
- Parent → child's PID
- Child → `0`
- Failure → `-1`

The parent and child have separate address spaces.

---

## 3. Process Tree / Counting

When there are multiple `fork()` calls, **draw the process tree first**.

```c
fork();
fork();
```

Both processes created by the first `fork()` reach the second `fork()`, so:

```text
        P
       /       P   C
     / \ /     P  C P  C
```

Total = **4 processes**.

General rule: `n` unconditional forks can produce up to `2^n` processes, assuming every process reaches every fork.

---

## 4. `fork()` and Output

Example:

```c
printf("A\n");
fork();
printf("B\n");
```

Output counts:

```text
A → 1 time
B → 2 times
```

Why? Only one process exists when `A` executes; two processes reach `B`.

After a `fork()`, output order is generally **nondeterministic** because the scheduler decides which process runs first.

**Habit:** Count how many processes reach each line.

---

## 5. `wait()`

```c
wait(NULL);
```

`wait()` makes the calling process wait for a child process to terminate.

It provides **synchronization and ordering**.

A waiting process is blocked; it does not continuously consume CPU while waiting.

Think:

```text
fork() → create
wait() → synchronize
```

---

## 6. `exec()`

`exec()` does **not** create a new process.

It replaces the current process's program with another program.

Important:
- PID stays the same.
- Current program code is replaced.
- On successful `exec()`, it **never returns** to the old program.
- Code after `exec()` runs only if `exec()` fails.

Example:

```c
execl("/bin/echo", "echo", "HELLO", NULL);
```

Mental model:

```text
Before exec:
PID 123 → program A

After exec:
PID 123 → program B
```

Same process, different program.

---

## 7. `fork()` vs `exec()` vs `wait()`

Remember this three-word model:

```text
fork()  = create
exec()  = replace
wait()  = synchronize
```

This distinction is one of the most important interview concepts in the Process API.

---

## 8. The UNIX Shell

A typical shell executing a foreground command conceptually does:

```text
shell
  |
 fork()
  |
  +---- child ---- configure FDs/pipes/env ---- exec(command)
  |
  +---- parent ---- wait()
```

For a background command, the shell can avoid waiting and continue accepting commands.

The separation between `fork()` and `exec()` lets the child configure its environment before becoming the requested program.

---

## 9. File Descriptors

The standard file descriptors are:

```text
0 → stdin
1 → stdout
2 → stderr
```

A program generally does not need to know whether stdout is connected to:
- a terminal
- a file
- a pipe

It simply writes to file descriptor `1`.

---

## 10. Redirection

For:

```bash
ls > output.txt
```

The shell/child arranges for stdout (FD `1`) to refer to the file before executing `ls`.

Conceptually:

```c
close(1);
open("output.txt", ...);
exec(...);
```

Open file descriptors survive `exec()` unless configured otherwise (for example, close-on-exec).

This is why `ls` can remain unaware that its output is being redirected.

---

## 11. Pipes

For:

```bash
ls | wc
```

Conceptually:

```text
ls stdout
    |
    v
  pipe
    |
    v
wc stdin
```

The shell creates the pipe and arranges the file descriptors before `exec()`.

The programs simply use their normal stdout/stdin.

---

## 12. Signals

Signals are asynchronous notifications used for process control.

Example:

```c
kill(pid, signal);
```

Important:
- `kill()` sends a signal; it does not necessarily mean "kill the process."
- `SIGTERM` requests termination and can be handled.
- `SIGKILL` forces termination and cannot be caught or ignored.
- Permissions/users constrain which processes can send signals.
- `root` / superuser has elevated privileges.

---

## Interview Checklist

Before answering a `fork()` question, ask:

1. **How many processes exist now?**
2. **Which processes reach the next `fork()`?**
3. **How many processes reach each `printf()`?**
4. **Which process calls `wait()`?**
5. **Does `exec()` succeed?**
6. **If `exec()` succeeds, stop tracing the old program.**
7. **What do file descriptors 0, 1, and 2 refer to?**
8. **For multiple forks, draw the process tree first.**
9. **Separate process count from output order.**
10. **In production code, check system-call return values.**

---

## 30-Second Summary

```text
Process = running program
fork    = create a process
exec    = replace the program
wait    = synchronize with a child
FD 0    = stdin
FD 1    = stdout
FD 2    = stderr
pipe    = connect processes
signal  = notify/control a process
```

### Most Important Habit

For `fork()` problems:

```text
1. Draw the process tree.
2. Determine which processes reach each line.
3. Count the output.
4. Apply wait() for ordering/synchronization.
5. Apply exec() as a program replacement.
6. Only then reason about possible output order.
```

# Operating Systems - Chapter 4: The Process

## 1. Why do we need Processes?

A program is just a passive executable stored on disk. It contains instructions and static data but cannot execute by itself.

A process is a running instance of a program created and managed by the Operating System.

The OS virtualizes the CPU by rapidly switching between processes (time sharing), giving the illusion that every program has its own CPU.

---

## 2. Mechanism vs Policy

### Mechanism
Defines **how** something is implemented.

Examples:
- Context Switching
- Interrupt Handling
- Register Saving

### Policy
Defines **which** decision should be taken.

Examples:
- Which process should run next?
- Which scheduling algorithm to use?

A good OS separates mechanisms from policies to keep the system modular.

---

## 3. What is a Process?

A process is the complete execution state of a running program.

It consists of:

### Address Space
- Code Segment
- Static Data
- Heap
- Stack

### CPU State
- Program Counter (PC)
- Stack Pointer (SP)
- CPU Registers

### I/O State
- Open File Descriptors
- Current Working Directory
- Other OS-managed resources

---

## 4. Process Address Space

```
+----------------------+
| Code                 |
+----------------------+
| Static Data          |
+----------------------+
| Heap                 |  ↑ grows upward
|                      |
| Free Space           |
|                      |
| Stack                |  ↓ grows downward
+----------------------+
```

### Code
Contains executable machine instructions.

### Static Data
Contains global and static variables.

### Heap
Used for dynamically allocated memory.

Characteristics:
- Lifetime controlled by the program (or GC in managed languages)
- Shared among functions
- Objects generally live here

### Stack
Stores execution context.

Contains:
- Local variables
- Function parameters
- Return addresses
- Stack frames

Each thread has its own stack.

---

## 5. Stack vs Heap

Primitive local variables generally live on the stack.

Objects are allocated on the heap.

Variables referencing objects are stored on the stack, while the actual object resides on the heap.

```
Stack                  Heap

person ----------->  { name: "John" }
```

JavaScript copies references, not objects.

---

## 6. Machine State

Machine State is everything required to resume execution.

Includes:

- Program Counter
- Stack Pointer
- General Purpose Registers
- Address Space
- Open Files

---

## 7. Important Registers

### Program Counter (PC)

Stores the address of the next instruction to execute.

Without the PC, execution cannot continue.

---

### Stack Pointer (SP)

Points to the top of the current stack.

Used during:
- Function calls
- Returns
- Local variable allocation

---

### CPU Registers

Small, extremely fast storage inside the CPU.

Used for:
- Arithmetic
- Temporary values
- Addresses
- Intermediate computations

Registers are process-specific state.

---

## 8. File Descriptors

Every process maintains a table of open files.

Standard descriptors:

```
0 → stdin
1 → stdout
2 → stderr
```

Example:

```
console.log()
        ↓
write(fd = 1)
```

---

## 9. Process Creation

When executing:

```bash
node app.js
```

The OS performs:

1. Loads executable from disk to memory.
2. Creates process address space.
3. Loads code and static data.
4. Allocates stack.
5. Allocates heap.
6. Initializes argc and argv.
7. Opens standard file descriptors.
8. Initializes CPU registers.
9. Sets Program Counter to program entry point.
10. Places process into the Ready Queue.

The scheduler eventually dispatches it to the CPU.

---

## 10. Program Loading

Program initially resides on:

```
SSD
```

OS loads required sections into:

```
RAM
```

Modern operating systems use **lazy loading**, loading pages only when required.

---

## 11. Process States

### Running

Currently executing on a CPU.

---

### Ready

Ready to execute but waiting for CPU time.

---

### Blocked (Waiting)

Cannot continue until an external event completes.

Examples:
- Disk I/O
- Network packet
- User input

Transition:

```
Running
    ↓
Blocked
    ↓ (event completes)
Ready
    ↓
Running
```

Blocked processes do **not** consume CPU.

---

### Zombie

A process that has finished execution but whose exit status has not yet been collected by its parent via `wait()`.

---

## 12. Scheduling

The scheduler decides which Ready process runs next.

Scheduler performs:

```
Ready Queue

↓

Choose Process

↓

Dispatch to CPU
```

The scheduler is a **policy**.

---

## 13. Context Switching

A context switch occurs when the OS stops one process and resumes another.

Steps:

1. Save current CPU context.
2. Load next process context.
3. Resume execution.

Saved context includes:

- Program Counter
- Stack Pointer
- CPU Registers

Without saving context, execution cannot resume correctly.

---

## 14. CPU Virtualization

Even with few physical CPU cores, the OS creates the illusion of many CPUs using:

- Time Sharing
- Context Switching
- Scheduling

This is called **CPU Virtualization**.

---

## 15. Process API

Every OS exposes operations for processes.

Typical operations:

- Create
- Destroy
- Wait
- Suspend
- Resume
- Query Status

Examples (Linux):

- fork()
- exec()
- wait()
- exit()
- kill()

---

## 16. Process Control Block (PCB)

The OS maintains one PCB per process.

Typical information:

- PID
- Process State
- Register Context
- Memory Information
- Open Files
- Parent Process
- Kernel Stack
- Scheduling Information

During a context switch, CPU state is copied between the CPU and the PCB.

---

## 17. Key Interview Points

- Program is passive; Process is active.
- Process = Address Space + CPU State + I/O State.
- Heap stores dynamically allocated objects.
- Stack stores execution context.
- Objects live on heap; references may live on stack.
- PC stores next instruction.
- SP points to current stack top.
- Registers are process execution state.
- File descriptors represent open files.
- Scheduler is policy.
- Context switch is mechanism.
- Ready != Running.
- Blocked processes are waiting for external events.
- Zombie process has exited but has not yet been reaped.
- PCB stores everything required to manage and resume a process.
- CPU virtualization is achieved through scheduling and context switching.

---

# Roadmap Completed

✅ Process Abstraction

✅ Address Space

✅ Stack & Heap

✅ Machine State

✅ Registers

✅ Program Counter

✅ Stack Pointer

✅ File Descriptors

✅ Process Creation

✅ Program Loading

✅ Process States

✅ Scheduling Basics

✅ Context Switching

✅ CPU Virtualization

✅ Process API

✅ Process Control Block (PCB)

---

# Next Chapter

**Limited Direct Execution (LDE)**

Topics:

- Interrupts
- User Mode vs Kernel Mode
- Timer Interrupts
- Interrupt Controller
- Interrupt Vector Table
- Trap Handler
- Context Switching Internals
- System Calls

This chapter explains **how the Operating System regains control of the CPU** while user programs are executing.
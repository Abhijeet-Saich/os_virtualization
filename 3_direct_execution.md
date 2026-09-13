# Limited Direct Execution — Revision Notes

## Core Tension
- OS virtualizes CPU = makes 1 physical CPU look like many, via **time sharing** (run process a bit, switch, repeat)
- Two competing goals:
  - **Performance** — switching overhead must be near-invisible
  - **Control** — OS stays ultimate authority; no process can hog CPU or touch memory/devices it shouldn't
- Everything below = mechanisms resolving this tension

---

## 1. Direct Execution (naive baseline)
- Protocol: create PCB entry → alloc memory → load binary → set up stack (`argc`/`argv`) → zero registers → jump to `main()`
- Fast (native execution, zero overhead) but **two fatal gaps:**
  - ❌ **No restriction** — process could do raw I/O, corrupt disk, read other processes' memory
  - ❌ **No preemption path** — if process never returns (infinite loop), OS never regains CPU
- **Gotcha:** OS isn't a background supervisor — it's just code sitting in memory. If a process is running, OS code is *not* executing. Only one CPU, one instruction stream.
- These 2 gaps → Problem #1 and Problem #2 below

---

## 2. Problem #1: Restricted Operations
**Fix = hardware privilege modes**
- **User mode** — restricted; privileged instr → hardware exception → OS kills process
- **Kernel mode** — unrestricted; only OS runs here

**Syscall flow (e.g. `read()`):**
1. Library wrapper puts args + **syscall number** in agreed registers
2. Executes **trap instruction** (`int 0x80` / `syscall` on x86)
3. Hardware auto-does: save caller's PC/registers → onto process's own **kernel stack**; flip mode bit → kernel; jump to fixed address
4. Fixed address = looked up in **trap table** (IDT) — set up by OS at boot (privileged instr: `lidt`)
   - ⚠️ **Gotcha:** process gives a *number*, never a raw address. Prevents jumping into middle of kernel code past permission checks (this is the ROP-exploit primitive)
5. Kernel validates args (e.g. is buffer pointer actually inside *this* process's memory, not kernel's?) then does privileged work
6. **Return-from-trap** (`iret`) — reverse of step 3: restore regs/PC, drop to user mode, resume right after trap

**Key term split:**
- **Trap** = synchronous, self-inflicted (syscall or fault like div-by-zero) — process causes it
- **Interrupt** = asynchronous, external (device-triggered) — see next section

---

## 3. Problem #2: Regaining Control (Preemption)

**Cooperative scheduling**
- OS only regains CPU when process voluntarily traps in (syscall / `yield()` / fault)
- ❌ Fatal flaw: infinite loop + no syscalls = OS locked out forever → only fix historically = **reboot**

**Non-cooperative fix = timer interrupt**
- Interrupt = external, hardware-triggered, independent of what instruction is executing
- OS at boot: programs timer chip to fire every **~1–10 ms** (100–1000 Hz typical), registers handler in trap table
  - Arming/disarming timer = privileged (else process could disable its own preemption)
- On fire: hardware does same save+vector as trap — **no process cooperation needed**
- Inside handler: **scheduler** picks next process
  - ⚠️ **Gotcha:** scheduler is NOT a separate entity — it's just a function *inside* the kernel, called during interrupt handling
  - Scheduling *policy* (which process, why) ≠ this chapter's topic — that's *mechanism* only

**Numbers (lmbench, 1996, 200MHz):**
- Syscall ≈ 4 µs
- Context switch ≈ 6 µs
- Modern CPUs: ~10x faster, sub-microsecond

---

## 4. Context Switching — Mechanics

**Context** = registers + program counter + stack pointer (full resumable state)

**Two separate saves — don't merge these:**
| # | Who saves | What | Where |
|---|---|---|---|
| 1 | Hardware (automatic, on interrupt) | outgoing process's user-mode registers | onto **its own kernel stack** |
| 2 | OS software (`switch()`/`swtch()`) | outgoing process's kernel-mode registers | into its **process structure (PCB)** — persistent, not a stack |

**`switch()` steps:**
1. Save A's registers → A's PCB
2. Restore B's registers ← B's PCB
3. **Pivot: reassign stack pointer (`%esp`) to B's kernel stack** ← this is the actual "switch" moment
4. `ret` — pops return addr off *whatever stack is now active* (B's) → lands in B's saved location, not A's

- ⚠️ **Gotcha:** it's the *same instruction stream* before/after step 3 — no branch. Only the stack pointer changing makes it "become" B's context.

**xv6 `swtch()` — two symmetric halves:**
- Save half: `movl`/`popl` writes A's registers + popped IP into A's context struct
- Restore half: `movl` reloads B's registers from B's struct, ending with `movl 4(%eax),%esp` (the pivot) + `pushl 0(%eax)` (stage B's IP for closing `ret`)

**Full end-to-end timeline:**
A (user mode) → timer fires → HW saves A → A's kernel stack, mode→kernel, jump to timer handler → scheduler picks B → `switch()`: save A→PCB, restore B←PCB, flip `%esp` to B's stack → `iret`: restore B's user regs, mode→user → B resumes exactly where it left off

---

## 5. Concurrency Hazard (preview only — not solved in this chapter)
- **Risk:** 2nd interrupt fires mid-handling of 1st (e.g. mid-`switch()`, state half-migrated) → inconsistent kernel state = **race condition**
- **Baseline fix:** disable interrupts during interrupt handling
  - ⚠️ Trade-off: disable too long → risk **dropping** other interrupts
- **Multicore gotcha:** disabling interrupts only protects *your* core — other cores still run and can touch shared kernel data → need **locking** (spinlocks/mutexes) too
- Full treatment = separate concurrency unit

---

## Glossary (fast lookup)

| Term | Meaning |
|---|---|
| Virtualization | Shared physical resource made to look dedicated, via time-multiplexing |
| User / kernel mode | HW-enforced privilege levels; mode bit gates privileged instructions |
| Trap | Synchronous, self-inflicted exception → mode switch into kernel |
| Trap table (IDT) | Boot-time table of fixed kernel entry points, indexed by number not address |
| Return-from-trap (`iret`) | Restore state, drop back to user mode |
| Interrupt | Asynchronous, externally-sourced exception (timer/disk/etc.) |
| Timer interrupt | Periodic (~1–10ms) HW interrupt enabling non-cooperative scheduling |
| Scheduler | Kernel-internal logic picking next process — not a separate entity |
| Context | Registers + PC + stack pointer |
| Context switch | SW save/restore of two processes' contexts + kernel-stack pivot |
| PCB (process control block) | Persistent per-process struct holding saved context between runs |
| Kernel stack (per-process) | Where HW auto-pushes registers on trap/interrupt entry |
| Race condition | Inconsistent state from unsynchronized concurrent access |

---

## Open Question → Next Chapter
This chapter = **mechanism** only (how a switch physically happens).
Next chapter = **policy** (which process gets picked — round robin, MLFQ, CFS, etc.)
# Chapter 9 — Scheduling: Proportional Share

Source: OSTEP, Chapter 9

## 1. The Core Idea

Earlier schedulers (FIFO, SJF, MLFQ) optimize for **turnaround time** or **response time** — i.e., "finish fast" or "feel responsive."

**Proportional-share scheduling** asks a different question entirely: instead of speed, it guarantees each process a fixed **percentage** of the CPU.

> Think of the CPU as a pie. The question isn't "who eats first?" — it's "how big is each person's slice?"

Three approaches to this, covered below: **Lottery**, **Stride**, and **CFS** (the real Linux scheduler).

---

## 2. Lottery Scheduling

### 2.1 Tickets = Proportional Share

Every process gets a number of **tickets**. Its share of the CPU is:

```
share = (process's tickets) / (total tickets in system)
```

A has 75 tickets, B has 25 (100 total) → A should get 75% of the CPU, B gets 25%.

⚠️ Tickets do **not** map directly to CPU time (e.g., "10 tickets = 10ms"). They only mean something as a **ratio against the total**. Add a third process with 100 more tickets, and A's unchanged 75 tickets now represent a smaller share.

### 2.2 The Lottery Mechanism

Every time slice, hold a "lottery":

1. Know the total ticket count (e.g., 100).
2. Draw a random number in `[0, totaltickets)` — the "winning ticket."
3. Each process owns a contiguous range of numbers (A: 0–74, B: 75–99).
4. Whoever's range contains the winner, runs.

```js
function pickWinner(processes) {
  // processes = [{ name: "A", tickets: 100 }, { name: "B", tickets: 50 }, { name: "C", tickets: 250 }]
  const totalTickets = processes.reduce((sum, p) => sum + p.tickets, 0);
  const winner = Math.floor(Math.random() * totalTickets);

  let counter = 0;
  for (const process of processes) {
    counter += process.tickets;
    if (counter > winner) {
      return process; // found the winner
    }
  }
}
```

The running `counter` is really just computing each process's range on the fly, rather than storing ranges explicitly.

**Implementation tip:** sort the list from highest → lowest tickets. Doesn't change correctness, but reduces average iterations (you're more likely to hit the winner early).

### 2.3 Why Randomness?

1. **Avoids worst-case corner cases** — no pathological input can reliably trip it up (unlike e.g. LRU's cyclic-access worst case).
2. **Lightweight** — no need to track per-process history, just ticket counts.
3. **Fast** — a random draw + lookup is cheap.

### 2.4 Fairness in Practice

Fairness metric: `F = (first job's finish time) / (second job's finish time)`. F = 1 is perfectly fair.

**Finding:** for **short** jobs, fairness can be quite poor (randomness hasn't had time to average out). As job length increases, fairness converges toward 1 — same logic as "law of large numbers."

### 2.5 Ticket Mechanisms

| Mechanism | What it does | Why it's useful |
|---|---|---|
| **Currency** | A user allocates tickets to their own jobs in their own private "currency"; system converts to global tickets | Users don't need to track the global ticket total — they just manage their own jobs' relative shares |
| **Transfer** | A process temporarily hands its tickets to another | Client/server: client is idle while waiting, so it lends its tickets to the server to speed up its own request |
| **Inflation** | A process temporarily raises/lowers its own ticket count | Only safe among **trusted/cooperating** processes — lets one self-announce "I need more CPU" without messaging others. Unsafe among competing/untrusted processes (one could cheat) |

### 2.6 Open Problem

How many tickets should a given job get in the first place? Unsolved in general. "Let users decide" just pushes the question down a level.

---

## 3. Stride Scheduling

Lottery is only *on average* correct. **Stride scheduling** is deterministic — exact proportions every cycle, no randomness.

### 3.1 Stride & Pass

- **Stride** = inversely proportional to ticket count. `stride = SOME_LARGE_NUMBER / tickets`.
  - More tickets → smaller stride.
- **Pass** = running total; starts at 0. Every time a process runs: `pass += stride`.
- **Rule:** always run whichever process has the **lowest pass** value.

Example — A=100, B=50, C=250 tickets, using 10,000 as the large number:

```
stride(A) = 10000/100 = 100
stride(B) = 10000/50  = 200
stride(C) = 10000/250 = 40
```

C has the most tickets but the *smallest* stride — its pass value grows slowest, so it keeps winning the "lowest pass" comparison, and runs most often. This matches its larger share.

### 3.2 Trace

| Pass(A) str=100 | Pass(B) str=200 | Pass(C) str=40 | Who runs? |
|---|---|---|---|
| 0 | 0 | 0 | A (tie-break) |
| 100 | 0 | 0 | B |
| 100 | 200 | 0 | C |
| 100 | 200 | 40 | C |
| 100 | 200 | 80 | C |
| 100 | 200 | 120 | A |
| 200 | 200 | 120 | C |
| 200 | 200 | 160 | C |
| 200 | 200 | 200 | *(tie — cycle repeats)* |

Over this full cycle (8 turns): A ran 2×, B ran 1×, C ran 5× → exactly 25% / 12.5% / 62.5%, matching their ticket ratio (100:50:250) **exactly**.

### 3.3 Stride's Weakness — No Global-State-Free Extension

What pass value should a **new** process start at?

- `0` → it looks maximally "behind," monopolizes the CPU until it catches up. Unfair to existing processes.
- "Current minimum" → avoids that, but requires inspecting **every other process's pass value** (global state) just to add one job.

**Lottery has no such problem** — a new process is added with its ticket count, total tickets updated, done. No per-process history to reconcile.

> **Trade-off:** Stride = exact, but fragile to dynamic changes. Lottery = imprecise short-term, but simple/robust to extend.

---

## 4. Linux CFS (Completely Fair Scheduler)

The actual scheduler used in real Linux for years. Primary design goal: **efficiency** (scheduling itself burns CPU — one Google datacenter study found ~5% of total CPU time went to scheduling).

### 4.1 vruntime — CFS's Version of "Pass"

Each process tracks `vruntime` — in the simple case, grows at the same rate as real time while running.

**Rule:** always run whichever process has the **lowest vruntime**. (Same pattern as stride's "lowest pass wins.")

### 4.2 Dynamic Time Slices

- `sched_latency` (typically 48ms) — the target period over which every runnable process should get to run once.
- `time_slice = sched_latency / n` (n = number of running processes).
  - 4 processes → 48/4 = 12ms each.

**Problem:** if n is large (e.g., 100), time_slice shrinks to near-zero → excessive context-switch overhead.

**Fix — `min_granularity`** (typically 6ms): time slice is never allowed to drop below this floor, even if it means the system isn't perfectly fair within exactly one `sched_latency` window.

> This is the same **fairness vs. overhead** trade-off seen with round-robin's quantum size, just resurfacing inside CFS.

### 4.3 Priorities — `nice` Values & Weights

- `nice` ranges from **-20 to +19**, default **0**.
- **Lower nice = higher priority** (coun터-intuitive: "less nice" = wants more CPU).
- Each nice value maps to a fixed **weight** (bigger weight = bigger CPU share — CFS's equivalent of "more tickets"):

```
nice -20 → weight 88761
nice  -5 → weight  3121
nice   0 → weight  1024   (default, weight0)
nice   5 → weight   335
nice  15 → weight    36
```

**Time slice formula** (proportional by weight instead of equal split):

```
time_slice(k) = (weight(k) / Σ all weights) × sched_latency
```

Example: A (nice=-5, weight 3121) vs B (nice=0, weight 1024):
```
time_slice(A) ≈ 36ms   (≈3× B's slice)
time_slice(B) ≈ 12ms
```

### 4.4 The vruntime Weighting Formula

To make high-priority processes run more (given "lowest vruntime wins"), their vruntime must grow **slower**:

```
vruntime += (weight(0) / weight(i)) × runtime
```

- `weight(0)` = 1024, the fixed default-priority weight.
- `weight(i)` = this process's own weight.

Example: A (weight 3121) runs 10ms real time:
```
vruntime(A) += (1024/3121) × 10 ≈ 3.3   ← grows slowly
vruntime(B) += (1024/1024) × 10 = 10    ← grows at full rate (default priority)
```

A's vruntime (3.3) < B's vruntime (10) → A gets picked next. **The weight ratio is what creates the favoritism** — no separate priority-check logic needed; it falls naturally out of "always pick lowest vruntime."

### 4.5 Efficient Storage — Red-Black Tree

**Problem:** every scheduling decision needs the minimum vruntime, and every process re-insertion (after running) needs correct sorted placement. A plain sorted list makes re-insertion `O(n)` — too slow at scale (thousands of processes).

**Background — BST (binary search tree):** left subtree < node < right subtree. Lets you skip half the remaining tree at each step when searching.

**Balanced tree:** a BST degenerates into a straight line (`O(n)` search, no better than a list) if values arrive in sorted order and nothing corrects the shape. A **rotation** restructures a BST into a different, shallower shape while preserving the BST ordering rule — e.g., turning a 3-node line into a balanced 2-level tree. A **red-black tree** automatically performs rotations on insert/delete to guarantee it never degenerates, keeping depth at `O(log n)`.

CFS keeps all **runnable** processes in a red-black tree ordered by vruntime:
- Find next-to-run (min vruntime) → cheap.
- Re-insert after running → `O(log n)` instead of `O(n)`.

Sleeping/blocked (e.g., I/O-waiting) processes are **not** kept in this tree.

### 4.6 Handling Sleeping / I/O-Bound Processes

**Problem:** if process B sleeps 10 seconds while A keeps running, A's vruntime climbs the whole time but B's stays frozen. If B resumes with that old, low vruntime, "lowest vruntime wins" means B would **monopolize the CPU** for ~10 seconds straight to "catch up," starving A.

**CFS's fix:** on wake-up, reset the process's vruntime to the **current minimum** in the tree — not its old frozen value.

**Residual cost:** this reset happens on *every* wake-up. A process that sleeps **briefly and frequently** (typical of I/O-heavy jobs — constant short waits on disk/network) gets its vruntime "reset to current minimum" over and over, never accumulating the kind of continuous fairness tracking a CPU-bound process gets. Net result: **frequently-sleeping jobs tend not to get their fair share of CPU**, even with this fix in place.

### 4.7 Beyond Scope (mentioned, not detailed)

CFS also includes: cache-performance heuristics, multi-CPU scheduling strategies, and scheduling across process *groups* rather than individuals.

---

## 5. Summary Comparison

| | Mechanism | Precision | Key weakness |
|---|---|---|---|
| **Lottery** | Random draw weighted by tickets | Accurate only *on average*, over many rounds | Poor fairness for short jobs |
| **Stride** | Deterministic pass += stride; lowest pass runs | Exact every cycle | Needs global state; awkward for new processes joining |
| **CFS** | vruntime (weighted) + red-black tree; lowest vruntime runs | Practical/scalable, the real-world Linux default for years | Short, frequent sleepers (I/O-bound jobs) under-served |

**General weaknesses of proportional-share scheduling as a category:**
1. Doesn't mesh well with I/O-bound jobs (see §4.6).
2. The "ticket/priority assignment problem" is never really solved — no formula tells you *how many tickets* or *what nice value* a given job deserves.

**Historical note:** as of **Linux 6.6**, CFS was replaced as the default scheduler by **EEVDF** (Earliest Eligible Virtual Deadline First) — another fair-share approach, not covered in depth in this chapter.
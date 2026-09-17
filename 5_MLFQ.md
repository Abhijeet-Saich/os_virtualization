# MLFQ — Multi-Level Feedback Queue (Revision Notes)

## The Problem It Solves
The OS wants two things at once:
- **Low turnaround time** → best achieved by running short jobs first (like SJF/STCF)
- **Low response time** → best achieved by round robin (everyone gets quick, frequent turns)

These conflict. And the OS can't just "run short jobs first" because **it doesn't know how long a job will run in advance** — it can only observe behavior as the job actually executes.

**MLFQ's core idea:** learn a job's nature from its *past behavior*, and use that history to predict how to treat it going forward.

---

## The Five Rules (Final Version)

**Rule 1:** If Priority(A) > Priority(B), A runs, B waits.

**Rule 2:** If Priority(A) = Priority(B), A and B run in round robin (using that queue's time slice/quantum).

**Rule 3:** When a job enters the system, it is placed at the **highest** priority queue.
> New jobs get the benefit of the doubt — treated as potentially short/interactive until proven otherwise.

**Rule 4:** Once a job uses up its **allotment** (total CPU time allowed) at a given priority level — **regardless of how many times it gave up the CPU in between** — its priority is reduced by one queue.
> This replaces the original naive Rules 4a/4b, which reset the allotment every time a job yielded early (e.g. for I/O). That let jobs "game" the scheduler by yielding just before their allotment ran out, staying at high priority forever. The fix: track **cumulative** usage at a level instead of resetting it on every yield.

**Rule 5:** After some time period **S**, move **all** jobs in the system to the topmost queue.
> This is the **priority boost** — a full reset for everyone, not just the job that's been waiting longest (boosting only the starved job would let it unfairly cut ahead of jobs already behaving well at the top). It solves:
> - **Starvation** (long jobs stuck at the bottom get periodic guaranteed access again)
> - **Behavior change** (a job that used to be CPU-bound but has since become interactive gets a fresh chance to prove itself)
>
> Choosing S is a classic **"voo-doo constant"** (Ousterhout's term) — no clean formula:
> - S too large → starvation creeps back in between boosts
> - S too small → interactive jobs get hurt by too-frequent resets

---

## Key Terms
- **Allotment** — the amount of CPU time a job may spend at a given priority level before being demoted.
- **Starvation** — a job never getting to run because higher-priority jobs keep arriving.
- **Gaming the scheduler** — exploiting scheduler rules to unfairly gain more CPU time than intended.
- **Voo-doo constant** — a tunable system parameter with no principled "correct" value, only tradeoffs (Ousterhout's Law: avoid these when possible, though it's often hard).

---

## Worked Examples (for intuition)
1. **Long CPU-bound job alone** (10ms time slice = allotment, 3 queues): enters at top (Q2) → uses full allotment → drops to Q1 → uses full allotment again → drops to Q0 (bottom), stays there.
2. **Long job (A) + short job (B) arrives:** A is already at the bottom. B arrives at T=100, enters at top (Rule 3), immediately preempts A (Rule 1). B only needs 20ms total, so it finishes within two time slices, never reaching the bottom — MLFQ "approximates SJF" without ever knowing B's length in advance.
3. **Long job (A) + I/O-bound interactive job (B):** B needs only 1ms of CPU before doing I/O. It yields well before its allotment expires, so by (old) Rule 4b it stays at the top queue indefinitely — near-instant service every time it needs the CPU.
4. **Gaming exploit (old rules):** a job issues I/O at ~99% of its allotment repeatedly → resets priority every time → nearly monopolizes the CPU. Fixed by new Rule 4's cumulative accounting — the job now slowly sinks down the queues regardless of its I/O behavior.
5. **Priority boost in action:** without boost, a long job competing with recurring short interactive jobs starves completely. With a boost every S (e.g. 100ms), the long job is guaranteed periodic access to the top queue.

---

## Real-World Notes (lighter — context, not core mechanism)
- **Solaris (TS class):** ~60 queues, time slices from 20ms (top) to a few hundred ms (bottom), boost roughly every ~1 second. Configurable via tables.
- **FreeBSD (4.3):** uses a formula that decays priority based on CPU usage over time, rather than the table/queue approach described here.
- **`nice`:** a command-line utility letting users/admins give "advice" to the scheduler about a job's priority.
- Some schedulers reserve top priority levels exclusively for OS work, off-limits to normal user jobs.
- Varying time-slice length by queue is common: short slices (≤10ms) at high-priority/interactive queues, long slices (100s of ms) at low-priority/CPU-bound queues.

---

## Chapter Summary (verbatim gist)
MLFQ has multiple levels of queues and uses *feedback* (observed behavior) to set a job's priority. History is its guide — pay attention to how jobs behave over time and treat them accordingly. It doesn't need a priori knowledge of a job's nature; instead it observes execution and prioritizes accordingly, giving near-SJF performance for short/interactive jobs while remaining fair and making progress on long CPU-bound jobs. Real systems using MLFQ variants: BSD UNIX derivatives, Solaris, Windows NT and later Windows.

---
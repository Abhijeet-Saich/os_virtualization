# Scheduling — Revision Notes (OSTEP Ch. 7)

## Core Idea
The CPU can only run one thing at a time (per core). The **scheduler** decides
whose turn is next. A **context switch** = pausing one job, saving its state,
and running another. Fast enough to look simultaneous, but it isn't.

## Workload Assumptions (starting point, relaxed one by one through the chapter)
1. Every job runs for the same amount of time
2. All jobs arrive at the same time
3. Once started, a job runs to completion (no interruption)
4. Jobs only use the CPU (no I/O)
5. The scheduler knows each job's run-time in advance

## Metrics

**Turnaround time** = finish time − arrival time
→ "How long start-to-finish?"

**Response time** = time of *first* run − arrival time
→ "How long until I see *anything* happen?"

These two metrics often pull in opposite directions — that tension is the
main theme of the chapter.

---

## Algorithms

### 1. FIFO (First In, First Out)
- Run jobs in arrival order, straight through to completion.
- **Simple**, but suffers the **convoy effect**: a short job stuck behind a
  long one waits far longer than necessary.
- Example: A=100s, B=10s, C=10s, FIFO order → turnarounds 100, 110, 120
  → avg = **110**

### 2. SJF (Shortest Job First)
- Run the shortest job first, then next shortest, etc.
- **Non-preemptive**: once started, a job can't be interrupted.
- Fixes convoy effect *if all jobs arrive together*.
- Same example, SJF order (B, C, A) → turnarounds 10, 20, 120 → avg = **50**
- **Still breaks** if a long job is already running when short jobs arrive
  late — can't preempt, so short jobs wait anyway.
- SJF = FIFO when all jobs are equal length (nothing to prioritize).

### 3. STCF (Shortest Time-to-Completion First) — aka Preemptive SJF
- Same idea as SJF, but **preemptive**: on every new arrival, compare
  *remaining* time of current job vs. new job; run whichever is shorter.
- Fixes SJF's late-arrival weakness.
- **Optimal for average turnaround time** — but bad for response time,
  since jobs arriving together still wait for others to run to completion.

### 4. Round Robin (RR)
- Run each job for a fixed **time slice** (quantum), then switch to next
  job, cycling through repeatedly.
- **Great for response time** — everyone gets a turn quickly.
- **Bad for turnaround time** — jobs get stretched out since they keep
  getting paused for others.
- Time slice too short → context-switch overhead dominates.
  Time slice too long → RR starts behaving like FIFO.

---

## The Core Trade-off

| | Turnaround Time | Response Time |
|---|---|---|
| **SJF / STCF** | ✅ Great | ❌ Bad |
| **Round Robin** | ❌ Bad | ✅ Great |

Fairness (RR) and raw completion speed (STCF) are fundamentally at odds.
You can't fully optimize both with these algorithms.

---

## Incorporating I/O (relaxing assumption 4)
- Real jobs alternate CPU bursts and I/O waits.
- Trick: treat each CPU burst between I/O requests as its **own mini-job**.
- While Job A waits on I/O (disk/network), scheduler runs Job B instead.
- This **overlap** keeps the CPU busy instead of idling during I/O waits.

## The "No Oracle" Problem (relaxing assumption 5)
- Real schedulers don't know job lengths in advance — STCF/SJF are
  unrealistic as described.
- Unsolved by this chapter — the fix (predicting job length from recent
  past behavior) is the **next chapter: Multi-Level Feedback Queue (MLFQ)**.

---

## Homework Log

**Q1 — Three jobs, length 200 each, FIFO vs SJF:**
- Turnaround: 200, 400, 600 (avg 400)
- Response: 0, 200, 400 (avg 200)
- SJF identical to FIFO — equal-length jobs give SJF nothing to sort by.
  (Answers homework Q4: SJF = FIFO turnaround whenever jobs are equal length.)

**Q2 — Jobs of length 100, 200, 300, FIFO vs SJF:** *(in progress)*

**Q3 — same jobs + RR, quantum=1:** *(not yet done)*

**Q4 — When does SJF = FIFO turnaround?** → Equal-length jobs (see Q1).

**Q5 — When does SJF = RR response time?** *(not yet done)*

**Q6 — Response time trend as SJF job lengths increase?** *(not yet done)*

**Q7 — RR worst-case response time equation for N jobs?** *(not yet done)*
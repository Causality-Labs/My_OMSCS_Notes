---
id: operating-systems-scheduling-summary
title: Scheduling (Summary)
course: operating-systems
lecture: scheduling
---


## Scheduling Overview

- CPU scheduler mediates CPU access for both user-level and kernel threads.
- Ready tasks are queued; when a CPU goes idle, the scheduler dispatches the next task.
- Tasks reach the ready queue through multiple events (e.g., new arrivals, I/O completions).
 ![](https://assets.omscs.io/notes/0CD9CFF9-08C7-4678-B3AE-F3C8E25B5042.png)
- Schedulers often allocate fixed timeslices; expiry forces a reschedule and context switch into the chosen task.
- Core goal: always pick the “best” next task from the ready queue.
- Effective policies hinge on matching scheduling algorithms with suitable runqueue data structures.

## Run to Completion Scheduling

- **Run to completion assumption:** once a task starts on a single CPU, it runs uninterrupted until it finishes; we know task runtimes and disallow preemption.
- **Evaluation metrics:** throughput, average completion time, average wait time, and CPU utilization gauge policy quality.
- **FCFS basics:** schedule tasks strictly by arrival order via a FIFO queue; simple data structure (enqueue back, dequeue front).
- **Example (T1=1s, T2=10s, T3=1s):** throughput 0.25 tasks/sec; average completion time 8 sec; average wait time 4 sec under FCFS.
- **Improving with SJF:** order jobs by shortest execution time to lower average wait/completion metrics when durations are known.
- **Runqueue implications:** SJF demands an ordered runqueue (e.g., priority queue or balanced tree) to find the shortest job quickly; FIFO is insufficient.

### Formulas:
**Throughput Formula:**
- jobs_completed / time_to_complete_all_job

**Avg. Completion Time Formula:**
- sum_of_times_to_complete_each_job / jobs_completed

**Avg. Wait Time Formula:**
- (t1_wait_time + t2_wait_time + t3_wait_time) / jobs_completed

## Preemptive Scheduling: SJF + Preempt

- Remove the no-preemption assumption: tasks may arrive at different times and interrupt currently running work.
- ![](https://assets.omscs.io/notes/302527F5-EA2F-4DD0-9E07-F38CA209E204.png)
- Example: T2 starts first; when shorter T1 and T3 arrive, SJF policy preempts T2 to run shorter jobs.
- ![](https://assets.omscs.io/notes/0EC22063-46A4-47EF-B109-571A03494F41.png)
- Each arrival triggers the scheduler to reassess the runqueue and potentially preempt the CPU.
- SJF still assumes known runtimes; in practice, heuristics such as last-run time or windowed averages estimate remaining duration.

## Preemptive Scheduling: Priority

- Priorities replace runtime estimates: critical kernel work usually outranks user tasks.
- Policy: always run the highest-priority runnable task; preempt lower-priority jobs when a higher one arrives.
- ![](https://assets.omscs.io/notes/55C02A40-B081-416F-826E-19749053B0C8.png)
- Example: T2 starts alone; when T3 (highest priority) and later T1 appear, the scheduler preempts T2, runs T3 to completion, resumes T2, then finally runs T1.
- ![](https://assets.omscs.io/notes/9FF251EC-C80C-4184-91CE-11DC71A15A45.png)
- Implementation often keeps one runqueue per priority level so the scheduler drains higher queues first.
- ![](https://assets.omscs.io/notes/B9E26ED4-074D-4A16-A0C4-8EBB6AB9966D.png)
- Risk: starvation of low-priority tasks if high-priority work keeps arriving.
- Mitigation: priority aging raises a task’s effective priority the longer it waits, guaranteeing eventual service.

## Priority Inversion
- Scenario: SJF with priorities P3 < P2 < P1; T3 starts alone, grabs a mutex.
- ![](https://assets.omscs.io/notes/975B4BFD-B8BD-46FC-BE7A-8E48B91F47E5.png)
- T2 (higher priority) preempts T3; later T1 (highest priority) arrives, preempts T2, but blocks waiting on T3’s mutex.
- T2 resumes and finishes; only then can T3 run, release the lock, and finally T1 executes.
- ![](https://assets.omscs.io/notes/E11CA11B-7FD7-435C-95CC-0A2E2CACFACE.png)
- Result: priority inversion—highest-priority T1 is delayed by lowest-priority T3; scheduling order becomes T2 → T3 → T1 instead of T1 → T2 → T3.
- Fix: priority inheritance—when T1 blocks on T3’s mutex, temporarily raise T3’s priority to P1 so it finishes quickly, then drop back to P3 after unlocking.
- Tracking mutex ownership in the lock data structure is essential to implement this boost.

## Round Robin Scheduling
- Round robin offers a fair alternative to FCFS/SJF when tasks share a priority.
- ![](https://assets.omscs.io/notes/BD3715F2-BD12-4661-ACDB-2E74F02B6B38.png)
- With T1, T2, T3 arriving together, the scheduler cycles: run T1, then T2, then T3, each getting CPU once before repeat.
- ![](https://assets.omscs.io/notes/62D89DA0-691E-4F48-9343-E437EB64C1B8.png)
- Priorities can layer on top of round robin by maintaining a queue per priority level.
- ![](https://assets.omscs.io/notes/EDCC9CA7-6286-49DB-A03C-F40A1B12DDE8.png)
- Timeslicing extends the idea: force preemption after a fixed quantum (e.g., 1 time unit) so runnable tasks interleave even if they don’t yield voluntarily.

## Timesharing and Timeslices
- **Timeslice basics:** maximum uninterrupted CPU window per task; tasks may yield sooner for I/O, sync, or higher-priority preemption.
- Enables interleaving/timesharing; essential for CPU-bound workloads, less critical when tasks often block on I/O.
- ![](https://assets.omscs.io/notes/4D3CD488-5851-47C3-8840-4E79CBD288FA.png)
- Round robin with 1-unit slices: T1 runs 1s and finishes; T2 runs 1s then is preempted by T3; T3 runs 1s and finishes; T2 resumes for remaining 9s.
- Metrics match SJF (throughput, average wait/completion) without needing runtime predictions.
- Benefits: short jobs finish quickly, scheduler reacts to changes, long I/O ops begin sooner.
- Costs: overhead from periodic interrupts, scheduler execution, context switches—even when only one runnable task remains.
- Overheads slightly reduce throughput and increase wait/completion times; keep slices much longer than context-switch time to limit impact.

## How long should a time slice be

-  Timeslices provide benefits to the system, but also come with certain overheads. The balance of the benefits and the overheads will inform the length of the timeslice. The answer is different for I/O bound tasks vs. CPU bound tasks.

## CPU Bound Timeslice Length

- Scenario: two CPU-bound tasks, each 10s; context switch cost 0.1s; compare 1s vs 5s quanta.
- ![](https://assets.omscs.io/notes/7E4262FA-1297-474A-A624-4BBD1B57DA5B.png)
- Short timeslice (1s) triggers frequent context switches, reducing throughput and lengthening completion time.
- Benefit of short slices: tasks begin sooner, improving average wait time—but users care more about completion for CPU-bound jobs.
- Best choice for CPU-heavy workloads is long slices (ideally infinite/no preemption) to minimize overhead and finish jobs fastest.
- ![](https://assets.omscs.io/notes/033B24F5-0B4D-445D-899E-9CC8A7119851.png)

### Calculations
- Timeslice = 1 second
    - throughput = 2 / (10 + 10 + 19*0.1) = 0.091 tasks/second
    - avg. wait time = (0 + (1+0.1)) / 2 = 0.55 seconds
    - avg. comp. time = 21.35 seconds
- Timeslice = 5 seconds
    - throughput = 2 / (10 + 10 + 3*0.1) = 0.098 tasks/second
    - avg. wait time = (0 + (5+0.1)) / 2 = 3.05 seconds
    - avg. comp. time = 17.75 seconds
- Timeslice = âˆž
    - throughput = 2 / (10 + 10) = 0.1 tasks/second
    - avg. wait time = (0 + (10)) / 2 = 5 seconds
    - avg. comp. time = (10 + 20)/2 = 15 seconds

## I/O Bound Timeslice Length

- Scenario 1: two I/O-bound tasks, 10s total work each, context switch 0.1s, I/O every 1s with 0.5s latency; compare 1s vs 5s quanta.
- ![](https://assets.omscs.io/notes/C7A8B07E-5DBC-445B-8C9B-639425A92F93.png)
- Both tasks yield after 1s for I/O no matter the timeslice, so metrics (throughput, wait, completion) are identical for 1s and 5s slices.
- Scenario 2: T1 is CPU-bound, T2 is I/O-bound.
- ![](https://assets.omscs.io/notes/F384B9B2-2367-42EA-B43F-DFAC45DF4626.png)
- With a 1s slice, behavior mirrors the mixed workload case; preemption keeps T2 responsive.
- With a 5s slice, T1 runs 5s straight, delaying T2’s I/O; throughput and wait time worsen for T2, though T1 finishes sooner.
- Conclusion: I/O-bound work benefits from short timeslices to keep devices busy and latency low.
- **Summary:** long quanta suit CPU-bound tasks (minimize context-switch overhead), short quanta suit I/O-bound/interactive tasks (fast I/O initiation, better utilization).

### Full Calculations

- for Timeslice = 1sec
    - avg. comp. time = (21.9 + 20.8) / 2 = 21.35

- Timeslice = 5 second*
    - throughput = 2 / 24.3 = 0.082 tasks/second
    - avg. wait time = 5.1 / 2 = 2.55 seconds
    - avg. comp. time = (11.2 + 24.3) / 2 = 17.75 seconds

## Summarizing Timeslice Length

CPU bound tasks prefer longer timeslices as this limits the number of context switching overheads that the scheduling will introduce. This ensures that CPU utilization and throughput will be as high as possible.

On the other hand, I/O bound tasks prefer short timeslices. This allows I/O bound tasks to issue I/O requests as soon as possible, and as a result this keeps CPU and device utilization high and well improves user-perceived performance (remember wait times are low).

### Formula 

- cpu_utilization = [cpu_running_time / (cpu_running_time + context_switching_overheads)] * 100
- The cpu_running_time and context_switching_overheads **should be calculated over a consistent, recurring interval**

## Runqueue Data Structures

- Runqueues are logical queues but can be implemented as multiple queues or trees; structure must make “next task” selection efficient.
- Different timeslices for CPU vs I/O-bound work can be handled with queue tagging or separate runqueues.
- ![](https://assets.omscs.io/notes/E73C69B5-5984-42BF-A68B-3E495829C2A3.png)
- Multi-queue designs assign small quanta to I/O-heavy tasks and long quanta to CPU-heavy ones, combining responsiveness with low overhead.
- Task classification leverages history: past behavior predicts future CPU/I/O intensity, but new or shifting tasks may need reclassification.
- Tasks move between queues: start in the top (short quantum); if they consume the full slice, demote to longer-slice queues; if they yield quickly, keep or promote them upward.
- I/O-friendly behavior from lower queues triggers boosts back toward shorter quanta.
- This adaptive structure is the **multi-level feedback queue (MLFQ)**.
- ![](https://assets.omscs.io/notes/1A833C17-905A-46B8-B90B-F5E0803B3B6E.png)
- MLFQ combines priority levels with distinct policies, using feedback to place each task on the most appropriate queue over time.

## Linux O(1) Scheduler

- **Design:** Linux O(1) is a preemptive, priority-based scheduler with constant-time enqueue/selection across 140 priority levels (0–99 real-time, 100–139 timesharing).
- **Priorities:** User tasks default to priority 120; nice values (−20…19) shift priority within the timesharing band.
- **Policy cues:** Each priority level gets its own timeslice length, echoing MLFQ ideas; interactive/sleepy tasks earn longer quanta, CPU-bound ones shorter quanta.
- **Feedback signal:** Sleep time during the slice adjusts priority (more sleep ⇒ boost by −5; less sleep ⇒ demote by +5).
- **Runqueue structure:** Two per-priority arrays—active and expired; bitmaps locate the highest non-empty priority in O(1).
- **Lifecycle:** Tasks that yield early rejoin active; those exhausting their slice move to expired. When active drains, the arrays swap.
- **Rationale:** Short slices for low-priority tasks prevent them from blocking higher priorities for long, despite their CPU intensity.
- **History:** Introduced in Linux 2.5; later replaced by Completely Fair Scheduler (CFS) in 2.6.23 to better meet modern responsiveness demands.
- ![[Pasted image 20251204102245.png]]
- 

## Linux CFS Scheduler

- O(1) scheduler drawback: tasks in the expired queue wait until all active tasks finish their quanta, causing jitter for interactive workloads and no fairness guarantee.
- Fairness goal: each task should receive CPU time proportionate to its weight/priority within a window.
- Replacement: Completely Fair Scheduler (CFS) for non-realtime Linux workloads.
- CFS runqueue: self-balancing red-black tree ordered by virtual runtime (vruntime).
- ![](https://assets.omscs.io/notes/EB1FD238-3A0E-4D51-AC7A-A859C6539011.png)
- Scheduling rule: always pick the leftmost (smallest vruntime) node; running task’s vruntime increases, and if it surpasses the new minimum, it is reinserted.
- Priority impact: lower-priority tasks accumulate vruntime faster (time “moves faster”), so they’re preempted sooner; higher priorities advance more slowly.
- Complexity: selecting next task remains O(1); insertions cost O(log n).
- Potential future pressure: if task counts explode, even logarithmic inserts might prompt searches for fair schedulers with constant-time operations.


## Scheduling on Multi-CPU Systems

- Multiprocessor basics: shared-memory systems have multiple CPUs with private L1/L2 caches and a possibly shared LLC; multicore CPUs expose multiple cores sharing an LLC.
- ![](https://assets.omscs.io/notes/4CF9EA4E-5B8F-4B27-B0EC-397F76E05656.png)
- OS treats every CPU/core as a schedulable entity.
- Cache affinity: schedule threads on the CPU/core they ran on previously to preserve hot caches and boost performance.
- Hierarchical approach: global load balancer spreads work; each CPU maintains its own runqueue and schedules locally.
- Balancing tools: monitor runqueue lengths and steal work from busy CPUs to idle ones.
- NUMA awareness: systems may have multiple memory nodes with differing access latencies depending on CPU proximity.
- ![](https://assets.omscs.io/notes/B506B72A-ECBC-4DF4-8DAB-62F9FD2BC10B.png)
- NUMA-aware scheduling keeps threads near their memory node to minimize remote access penalties.

## Hyperthreading

- Traditional CPUs hold one register set, so switching threads requires saving/restoring state; hardware designers mitigate this by provisioning multiple register sets per core.
- ![](https://assets.omscs.io/notes/DA77AC78-FEB2-4DB0-B12C-F84B43FF7D24.png)
- **Hyperthreading/SMT/CMT** exposes several hardware execution contexts per core; only one issues instructions at a time, but switching contexts is nearly instantaneous.
- Terminology varies—hardware multithreading, hyperthreading, CMT, SMT—but the concept is the same.
- Most systems offer two hardware threads per core (some high-end CPUs provide more) and let admins toggle hyperthreading at boot due to trade-offs.
- OS schedulers treat each hardware thread as a schedulable CPU.
- Scheduling choice: co-schedule threads that benefit from fast swaps; if a thread stalls on memory longer than the two quick context switches, switch to hide latency and keep the core busy.

## Scheduling for Hyperthreading Platforms

- Assumptions: SMT core with two hardware threads; CPU-bound threads can issue one instruction per cycle (max IPC); memory accesses stall for four cycles; context switches are effectively instantaneous.
- ![](https://assets.omscs.io/notes/2B1476A6-EE31-4AA9-A733-718FDDFE3727.png)
- Two CPU-bound threads: both compete for the single pipeline, alternating every cycle; each sees roughly half throughput while memory units sit idle.
- ![](https://assets.omscs.io/notes/B90046E6-16D2-4B95-89A9-DCF7264862CF.png)
- Two memory-bound threads: both spend cycles waiting on memory, leaving execution units underutilized—again poor overall throughput.
- ![](https://assets.omscs.io/notes/F48DA477-45B0-45E0-966D-BDA4E885C859.png)
- Mixed pair (one CPU-bound, one memory-bound): overlap compute with memory stalls by switching when the memory-bound thread requests data, keeping both pipeline and memory resources busy.
- ![](https://assets.omscs.io/notes/702A8D07-EEE9-4FB3-98B0-C32117214897.png)
- Result: co-scheduling complementary workloads minimizes idle cycles and contention; some interference remains but far less than homogeneous pairings.

## CPU Bound or Memory Bound

- Distinguishing CPU- vs memory-bound threads requires history, but simple “sleep time” heuristics fail: memory stalls happen in hardware pipelines, not OS queues, and tracking at software speed is too slow.
- Solution: use hardware performance counters that record execution metrics in real time (cache misses, IPC, power use, etc.).
- Tools like **oprofile** and **perf** expose these counters to the OS.
- Schedulers read counters to infer resource needs—for example, high LLC miss counts imply memory-bound behavior.
- Combining multiple counters and platform-specific models helps pick complementary thread mixes that minimize interference and maximize utilization.

## Scheduling with Hardware Counters

- CPI distinguishes workload type: low CPI (~1) ⇒ CPU-bound, high CPI ⇒ memory-bound.
- No hardware CPI counter was available, so Fedorova modeled behavior in a simulator instead of estimating 1/IPC in software.
- Simulation setup: 4 cores × 4-way SMT = 16 hardware contexts.
- Synthetic workload: 16 threads split evenly across CPI values 1, 6, 11, 16 (from CPU-heavy to memory-heavy).
- Goal: measure overall IPC (max 4) while assigning different CPI mixes to the cores.
- ![](https://assets.omscs.io/notes/19415E6A-E6EF-4A29-823A-EE8C95FFB327.png)

## CPI Experiment Results

- Four experiment results show IPC impact of different CPI mixes.
- ![](https://assets.omscs.io/notes/F39D7297-567A-4D1C-8B45-56285FCC173A.png)
- Balanced mixes of low/high CPI threads (experiments a, b) keep the pipeline busy and deliver high IPC.
- Homogeneous CPI groupings (experiments c, d) suffer—CPU-bound threads contend for the pipeline, while memory-bound threads leave it idle—dropping IPC sharply.
- Conclusion: scheduling heterogeneous CPI mixes boosts overall utilization.
- Real-world check: profiling real applications shows CPI values cluster tightly (≈2.5–4.5), unlike the synthetic extremes.
- ![](https://assets.omscs.io/notes/CC98A928-2A37-420E-A949-F801FA2DEB7E.png)
- Because practical CPI ranges overlap heavily, CPI alone may be insufficient for precise scheduling decisions.
- ![](https://assets.omscs.io/notes/A260F044-D32B-4ADA-8E2C-AE04AE7F0A92.png)
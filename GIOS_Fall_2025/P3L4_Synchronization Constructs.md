

## More on Synchronization

- Mutexes and condition variables are error-prone: easy to unlock the wrong mutex or signal the wrong condition, and they only model simple binary states.
- Complex patterns (e.g., readers/writers) need extra bookkeeping variables because these primitives lack expressive power.
- Correct synchronization hinges on hardware support via atomic instructions.
- Upcoming topics explore richer synchronization constructs and how hardware implements them efficiently.

## Spinlocks

- Spinlocks, like mutexes, enforce mutual exclusion with lock/unlock semantics.
- Key difference: a contending thread busy-waits on the CPU instead of blocking, consuming cycles until the lock opens or it’s preempted.
- Their simplicity makes them useful building blocks for richer synchronization primitives.


## Semaphore 

- Semaphores encode “how many slots” are available by holding an integer counter.
- Arriving threads decrement a positive counter and continue; if it’s zero they block until someone signals.
- Exiting threads signal the semaphore, incrementing the count for the next waiter.
- A binary semaphore (initial value 1) acts like a mutex, admitting only one thread at a time.
- p (proberen) try or wait,  v( verbhogen) exit post


## POSIX Semaphore

The simple POSIX semaphore API defines one type `sem_t` as well as three operations that manipulate that type. These operations create the semaphore, wait on the semaphore, and unlock the semaphore.

![](https://assets.omscs.io/notes/EA5326AD-08F0-47F0-9ACC-0640A0BFC497.png)

**NB** The pshared flag indicates whether the semaphore is to be shared across processes.


## Reader/Writer Locks

- Some workloads separate non-mutating reads from mutating writes; multiple readers can coexist, but writers need exclusivity.
- Reader/writer locks handle this pattern: the caller declares intent (read or write) and the lock enforces concurrent reads vs exclusive writes automatically.

## Using Reader Writer Locks

- Linux exposes rwlock_t plus read/write lock/unlock calls, letting code express intent directly.
- Reader/writer locks exist across OSes but semantics vary: naming (shared/exclusive), recursive read unlock behavior, whether readers may upgrade to writers, and fairness toward queued writers.
- Snippet:![](https://assets.omscs.io/notes/B9FC4930-CACD-4388-BF04-A83D95487B84.png)


## Monitors

- Monitors wrap shared resources with entry procedures and optional condition variables, so the monitor handles locking and signaling automatically.
- Developers call the monitored procedure; the framework does the acquire/check on entry and releases/notifies on exit, reducing manual locking errors.

## More Synchronization Constructs

- Serializers enqueue operations by priority and hide explicit signaling/condition handling.
- Path expressions let you declare allowed interleavings (e.g., “many reads, single write”) instead of coding locks directly.
- Barriers and rendezvous points hold progress until all required threads arrive—semantically opposite to semaphores that release a set number of threads.
- Wait-free schemes (e.g., Linux’s RCU) aim for lockless scalability by letting reads proceed optimistically when concurrent writes are rare.
- All of these rely on hardware atomic instructions to update shared state safely.
## Spinlocks Revisited

We will now focus on "The Performance of Spin Lock Alternatives for Shared Memory Multiprocessors", a paper that discusses different implementations of spinlocks. Since spinlocks are one of the most basic synchronization primitives and are used to create more complex synchronization constructs, this paper will help us understand both spinlocks and their higher level counterparts.


## Need for Hardware Support

- Spinlocks require the test and set to execute atomically so only one thread claims the lock.
- This broken version separates the check and assignment, leaving a race window:

```c

spinlock_lock(lock) {

    while(lock == busy)

        // Spin

    lock = busy;

}

```

- Because that sequence spans multiple cycles, threads can interleave between test and assignment; only hardware atomic instructions can make it safe.

## Atomic Instructions

- Atomic instructions span multiple cycles but execute all-or-nothing and serialize concurrent callers, guaranteeing mutual exclusion.
- Safe spinlock acquire uses test_and_set, which atomically swaps and reports the old value:

```c

spinlock_lock(lock) {

    while(test_and_set(lock) == busy);

}

```

- First arrival sees 0, sets it to 1, and enters; later arrivals read 1 and keep spinning until it clears.
- Available atomic primitives and their efficiency differ by architecture, so code must pick supported, optimal instructions per platform.

## Shared Memory Multiprocessors

- Shared-memory multiprocessors (SMPs) tie multiple CPUs to a common memory, either via per-module interconnects (parallel requests) or a single shared bus (one request at a time).![](https://assets.omscs.io/notes/8A94ED90-9931-476A-BA86-570434DB54A3.png)
- Each CPU caches data to mask main-memory latency, which is amplified by contention among processors for shared memory.![](https://assets.omscs.io/notes/6FBD6B73-A5F8-45C8-AF72-BEBBB7EA9B47.png)
- Write policies include no-write (bypass cache, invalidate entries), write-through (update cache and memory simultaneously), and write-back (update cache now, push to memory later on eviction).

## Cache Coherence

- Multiprocessors may be non-cache-coherent (software must sync caches) or cache-coherent (hardware keeps shared data consistent).![](https://assets.omscs.io/notes/B8415B29-8C25-4E4B-9735-AC1A2591DD34.png)
- Hardware enforces coherence via write-invalidate (broadcast invalidations; lower bandwidth, main-memory reloads later) or write-update (push new values everywhere; immediate cache hits).
- Write-invalidate amortizes traffic when other CPUs don’t need the data soon; write-update favors workloads that consume the fresh value right away.
- Developers can’t choose the policy—it’s fixed by the platform’s architecture.

## Cache Coherence and Atomics

- Atomic ops bypass caches and hit main memory so all CPUs serialize updates at a single point, avoiding conflicting cached versions.
- This guarantees correctness but slows atomics: every op pays main-memory latency, contends on the controller, and triggers coherence traffic to update/invalidate cached copies even if unchanged.
- Consequently, atomics are costlier on SMPs than on single-CPU systems (bus/interconnect contention) and more expensive than regular cached operations in general.

## Spinlock Performance Metrics

- Ideal spinlocks offer low latency: a free lock should be claimed instantly via one atomic instruction.
- They also minimize wait time: once a lock is released, spinning threads should acquire it immediately.
- Designs must curb bus/interconnect contention from atomics and coherence traffic, otherwise every CPU—including the one trying to release/acquire the lock—gets delayed.

## Test and Set Spinlock

- Classic test-and-set spinlock (test_and_set API above) uses a single, widely supported atomic.![](https://assets.omscs.io/notes/5CB26CEB-E617-4CBC-8123-3375C2C6875D.png)
- Latency and delay are optimal: a free lock is claimed with one atomic, and spinning threads grab it immediately after release.
- Drawback is contention: every spin reissues the atomic against main memory, wasting cycles and stalling other CPUs regardless of cache-coherence policy.

## Test and Test and Set Spinlock

- Test-and-test-and-set spinlock (spin-on-read) separates the cached check from the atomic: read lock locally, spin while it’s busy, then fire the atomic only when the cached value shows free.
- ![](https://assets.omscs.io/notes/C46BDD52-2F95-4EB4-8FA3-D10C244CF546.png)
- Latency/wait time are slightly worse than pure test-and-set because of the extra predicate, but still acceptable.
- Contention impact depends on cache coherence: no-coherence gains nothing; write-update lowers traffic yet lets everyone pounce simultaneously; write-invalidate is worst since every atomic invalidates cached copies, forcing all CPUs back to main memory.

## Spinlock "Delay" Alternatives

- Delay-based spinlocks pause after spotting a free lock so not every thread fires the atomic simultaneously.
- ![](https://assets.omscs.io/notes/525BCCBF-AF42-4E79-A432-B1108B420862.png)
- Contention drops because many contenders recheck later and often see the lock busy again; latency stays similar (cache fetch + atomic), but wait time grows—idle delay is wasted when no contention exists.
- A variation delays after every memory reference, which greatly cuts traffic on non-cache-coherent systems where each poll hits main memory.![](https://assets.omscs.io/notes/7925E4F5-39B8-4964-94C8-495FF0644EE2.png)
- That second form further increases wait time since the thread pauses on every probe, not just when the lock appears free.

## Picking a Delay

- Static delays (e.g., derived from CPU ID) are simple and spread out atomics under heavy load, but waste time when contention is low.
- Dynamic delays pick a random pause from a range that grows with perceived contention, avoiding long waits when the lock isn’t busy.
- Failed test_and_set attempts serve as a contention signal—the more failures, the more crowded the lock.
- Beware schemes that delay after every reference: delay with steady contention grows with critical-section length, even if actual contention hasn’t increased.

## Queueing Lock

- Queuing locks stagger visibility of the free lock so contenders don’t stampede simultaneously.![](https://assets.omscs.io/notes/16939119-DCB7-4091-9ADF-06204B9178B4.png)
- Implementation: maintain an n-element flag array (has_lock / must_wait), plus pointers to the current owner and queue tail; each arriving thread atomically increments queuelast, takes a ticket, and spins on its own flag until it flips to has_lock.
- Unlocking sets the next flag (queue[ticket + 1] = has_lock), handing off ownership in order.
- Downsides: needs a read_and_increment atomic (less ubiquitous than test_and_set) and consumes O(n) space versus one word for simpler locks.

## Queuing Lock Implementation

- Queue-based lock runs a single read_and_increment on queuelast; spinning happens on separate flag entries, so invalidations on the ticket counter don’t disturb waiters.![](https://assets.omscs.io/notes/64C44444-FF38-4F73-80C2-60198EE906BD.png)
- Latency suffers—read_and_increment is heavier than a plain test_and_set.
- Delay is excellent: each holder hands the lock to the next ticket immediately.
- Contention is very low because the atomic isn’t in the spin loop; to reap that benefit you need cache coherence and must place each flag on its own cache line to avoid false sharing.

## Spinlock Performance Comparisons

- Benchmark: each process ran a million critical sections on a 20-CPU Sequent Symmetry (write-invalidate coherence), measuring overhead vs the ideal no-contention runtime.![](https://assets.omscs.io/notes/6D87B21E-1691-43A6-BB9D-9901D9B4E543.png)
- Heavy load: queueing lock scales best (flat overhead), while test-and-test-and-set suffers most due to O(N²) coherence traffic with write-invalidate.
- Among delay locks, static delays marginally beat dynamic ones, and delaying after every reference edges out delaying only on perceived release.
- Light load: test-and-test-and-set excels thanks to minimal latency; dynamic delays outshine static (no unnecessary waits); queueing lock fares worst because read_and_increment and extra bookkeeping add latency.
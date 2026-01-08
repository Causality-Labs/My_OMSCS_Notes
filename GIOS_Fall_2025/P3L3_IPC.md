

## Inter Process Communication

- IPC is the OS toolkit that lets processes coordinate, synchronize, and exchange data.
- Mechanisms fall into message-based (pipes, sockets, queues) or memory-based (shared pages, mapped files).
- RPC builds richer semantics by defining protocols, data formats, and call procedures atop basic IPC.
- Synchronization primitives remain essential to regulate cooperative access.

## Message Passing IPC

- Message-based IPC lets processes exchange data via OS-managed channels (ports).
- Each send/recv involves a system call plus a data copy into/out of the channel, so a round trip costs four crossings and four copies—simple but not lightweight.
- Kernel handles channel management and synchronization on behalf of the processes.
- ![](https://assets.omscs.io/notes/68065B2A-21F6-4052-BD65-47800EEEE360.png)



## Forms of Message Passing

- Message-based IPC comes in several flavors, each with distinct semantics.
- Pipes link exactly two endpoints and stream raw bytes; great for chaining process output/input.![](https://assets.omscs.io/notes/C3ABC9F5-3DF1-4795-A5ED-0EE0D815A999.png)
- Message queues enforce message boundaries, support priorities/scheduling, and have SysV/POSIX APIs.
- ![](https://assets.omscs.io/notes/CC5C29F1-46D7-49F3-A77A-E95EEDA3F086.png)
- Sockets offer send/recv buffers, can attach stacks like TCP/IP, and work across machines via network devices.
- ![](https://assets.omscs.io/notes/5B515698-B3C8-4AA8-9866-853C236F4A31.png)



## Shared Memory IPC

- Shared memory IPC maps common physical pages into each process’s virtual space; addresses can differ and the physical region needn’t be contiguous.
- After setup, the OS steps aside—system calls happen only when creating the shared region.
- Fewer data copies, but apps must copy data into the shared region or allocate there directly.
- Both processes must coordinate with explicit synchronization and implement their own protocols.
- Unix offers SysV and POSIX shared memory APIs; memory-mapped files provide another shared channel.
- ![](https://assets.omscs.io/notes/321E7AFD-A603-4134-A47C-3A11F6A2CFC9.png)

## Copy vs Map

- Both IPC styles move data between process address spaces; they trade setup vs copy cost.
- Message-based IPC spends CPU cycles on every port copy (with user/kernel crossings).
- Memory-based IPC pays a one-time mapping cost; no crossings during bulk transfers, so it wins for large payloads.
- Windows optimizes via Local Procedure Calls: small messages use port copies, large ones map memory into the target.

## SysV Shared Memory 

- Shared-memory segments need not be backed by contiguous physical pages, but they’re governed by OS-wide limits (Linux allows up to ~4000 segments).
- When created, the OS allocates the physical pages and assigns a unique key; other processes use that key to locate and attach the segment.
- Attaching maps the shared physical pages into a process’s virtual space so all participants see reads/writes immediately.
- Detaching simply invalidates that process’s mappings; the segment persists and can be attached again until explicitly destroyed.

## SysV Shared Memory API

- shmget(shmid, size, flag) creates/opens a segment of size bytes with specified flags; caller supplies a shared-memory key.
- Use ftok(pathname, proj_id) to deterministically generate that key so cooperating processes agree on the same segment.
- shmat(shmid, addr, flags) attaches the segment, mapping it at addr (or an OS-chosen address if NULL); caller casts the returned pointer as needed.
- shmdt(shmid) detaches the segment, dropping this process’s mappings.
- shmctl(shmid, cmd, buf) issues control commands; pass IPC_RMID to destroy the segment when finished.

## Posix Shared Memory API

- POSIX shared memory models regions as pseudo-files in tmpfs, reusing filesystem metadata while keeping pages purely in RAM.
- Regions are identified by file descriptors, eliminating the SysV-style key management.
- Typical workflow: shm_open to create/open, mmap/munmap to map/unmap into a process, and shm_unlink to remove the region.

## Shared Memory and Synch

- Shared memory lets multiple processes touch the same data, so every access must be synchronized to prevent races.
- Processes can reuse threading primitives (e.g., pthread mutexes/conds) or OS-provided IPC locks/semaphores.
- Whatever tools you pick, you need mutual exclusion for shared regions and signaling to indicate when data becomes ready.

## Pthreads Synch for IPC

- Use PTHREAD_PROCESS_SHARED when initializing pthread mutex/cond attributes so they work across processes—and place the structs themselves in shared memory.
- Typical flow: generate a key with ftok, create the segment via shmget, attach with shmat, cast the returned pointer to your shared struct holding both data and the mutex.
- Set up pthread_mutexattr, call pthread_mutexattr_setpshared(..., PTHREAD_PROCESS_SHARED), then initialize the shared mutex at that mapped address.
- ![](https://assets.omscs.io/notes/79BDED5A-FAA7-4ADE-9C0E-2E696A76E998.png)


## Sync for Other IPC

- If PTHREAD_PROCESS_SHARED isn’t available, processes can still coordinate shared memory via OS-level IPC primitives.
- Message queues can enforce turn-taking by exchanging “ready”/“ok” signals around shared data access.
- Binary semaphores provide a two-state lock: value 1 lets a process enter (and decrements to 0); value 0 blocks until released.

## IPC Command Line Tools

![](https://assets.omscs.io/notes/A87134C6-ABEF-4923-B993-8E3C48A4A8E6.png)


## Shared Mem Design Considerations

- Plan how many shared segments you need: one large region requires your own allocator; many smaller segments need a pool plus a way (e.g., queue of IDs) to hand them out.
- Choose segment sizes carefully: fixed-size data can use uniform segments, but OS limits may constrain you.
- For variable or large payloads, stream data in chunks and include a protocol to track progress across rounds.


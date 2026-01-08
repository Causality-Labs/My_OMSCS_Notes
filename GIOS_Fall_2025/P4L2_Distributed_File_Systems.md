
## Distributed File Systems

- Operating systems present a unified filesystem API that can abstract away device differences—even when storage lives on remote machines accessed over the network.
- When the filesystem service spans multiple machines, it’s called a distributed filesystem (DFS).


## DFS Models

- DFS servers may replicate every file (fault tolerant, highly available) or partition the namespace (scales by adding servers).
- Many systems combine both: partition datasets, then replicate each shard.
- In peer-based designs every node stores and serves data, blurring client/server roles and sharing load.


## Remote File Service: Extremes

- Upload/download model: client pulls entire file, edits locally, then pushes back—fast local edits but costly for small changes and removes server visibility.![](https://assets.omscs.io/notes/B02657E2-D79B-43AB-87F4-7B75275CA3CB.png)
- True remote access: file stays on server; every operation goes over the network—ensures server-controlled consistency but incurs per-op network cost and limits scalability.![](https://assets.omscs.io/notes/4358EB0F-C95F-4106-A8ED-74E92E275263.png)


## Remote File Service: A Compromise

- Let clients cache and prefetch file blocks locally to cut latency and relieve server load.
- Cached clients must alert the server on writes and periodically validate cached data to catch remote updates.
- This hybrid keeps the server aware enough to enforce consistency while still letting many operations run locally.
- Trade-off: server logic and state become more complex, and clients must honor new sharing semantics beyond a normal local filesystem.


## Stateless vs Stateful File Server

- **Stateless servers** keep zero per-client or per-file context, so every request must be self-contained. They fit extreme models (pure upload/download or pure remote access) but can’t support caching or consistency control. Upsides: no state-management overhead and easy recovery—just reboot and resume service.

- **Stateful servers** track clients, open files, access patterns, caches, and locks. This enables coordinated caching, strong consistency, relative reads, etc. Downsides: extra CPU/memory for state, need to checkpoint/recover state on failure, and clients must participate in consistency protocols.

**Stateless DFS Servers**

- **Behavior:** Track no session data—each request must carry full context (file id, offset, credentials).
- **When it fits:** Simple upload/download or pure remote-access models where clients don’t cache.
- **Pros:** Minimal CPU/memory overhead; failover is trivial (restart and continue).
- **Cons:** No cache consistency or locking; higher per-request payloads; limited performance flexibility.

**Stateful DFS Servers**

- **Behavior:** Maintain per-client/file state (open handles, locks, cache ownership, recent writes).
- **Capabilities:** Coordinated caching and invalidation, byte-range and relative reads, fine-grained locking, better scheduling.
- **Pros:** Stronger consistency guarantees; richer semantics (sharing, leases, delegation).
- **Cons:** Extra resource cost; state must be checkpointed/replayed on failure; clients abide by protocol (callbacks, cache flushes).


## Caching State in a DFS

- DFS clients cache file blocks locally, but shared blocks need coherence: when one client updates, others must be told.
- Coherence can be triggered on access, periodically, or at open time, and may be enforced by clients or the server.


## File Sharing Semantics in DFS

- Single-node UNIX: all processes see writes immediately via the shared buffer cache.![](https://assets.omscs.io/notes/78F61AEB-F700-4CEE-8D92-14B7D8E0FC26.png)
- DFS: network latency and caching delay remote visibility of writes, so systems trade strict consistency for performance.
- ![](https://assets.omscs.io/notes/4559FB2E-50BC-44BB-8A2C-39C2159B9945.png)
- Session semantics: client pushes updates on close and checks for fresher versions on open; simple but stale data persists while files stay open.
- Periodic updates: clients and/or servers flush or invalidate on a timer; APIs like flush/sync help bound staleness.
- Other approaches: immutable files (write new versions) or transactional APIs to batch updates atomically.

## File Vs Directory Service

- Regular files and directories see very different access patterns, so DFS implementations often apply different consistency semantics to each (e.g., session semantics for files, stricter UNIX semantics for directories).


## Replication and Partitioning

- Replication copies the entire filesystem onto each server, enabling load balancing, high availability, and fault tolerance, but write operations must be coordinated across replicas (synchronously or via async propagation with reconciliation, e.g., voting).
- Partitioning assigns subsets of files to different servers, scaling capacity linearly and keeping writes local, but a partition failure makes its data unavailable and skewed access patterns can create hotspots; many systems combine both (partition + replicate each shard).


## Networking File System (NFS) Design

- NFS lets clients access remote files through the local VFS interface; the VFS routes remote operations to the NFS client, which talks to the NFS server via RPC.![](https://assets.omscs.io/notes/8EB8E1FB-C4FF-427C-A887-CD2B754A4DE0.png)
- On the server, NFS requests appear as regular filesystem calls to the local VFS/FS stack.
- Opening a file yields a file handle containing server and file metadata; the client reuses this handle for subsequent operations until it becomes stale (e.g., file deleted or server down).


## NFS Versions

- NFSv3 is stateless (in theory) but often implements some stateful helpers; NFSv4 is fully stateful, enabling coherent caching and locking.
- For non-shared files NFS acts like session semantics: close flushes writes, open refreshes caches.
- Periodic cache checks (default 3 s for files, 30 s for directories) bound staleness but break pure session behavior when multiple writers exist.
- NFSv4 can delegate file management to a client temporarily, removing the need for those periodic checks.
- State also lets NFSv4 provide lease-based locks and share reservations (reader/writer locks with upgrade/downgrade support).


## Sprite Distributed File System

![](https://assets.omscs.io/notes/BF2F3D15-E0CE-4DFE-B0DF-EFF69297E33E.png)



## Sprite DFS Access Pattern Analysis

- Sprite’s workload study showed ~33% writes, so caching (beyond simple write-through) could benefit the remaining 66% of operations.
- Most files are short-lived: 75% stay open <0.5 s, 90% <10 s; many new files are deleted quickly (20–30% within 30 s, 50% within 5 min).
- File sharing was rare, so the designers concluded write-back-on-close (session semantics) wasn’t worth the overhead, though concurrency had to be supported.


## Sprite DFS from Analysis to Design

- Sprite caches file data client-side but uses write-back: every 30 s it flushes blocks that haven’t been modified in the last 30 s, banking on the observation that freshly written data often gets deleted quickly.
- When another client opens a file being written elsewhere, the server forces the writer to flush outstanding dirty blocks.
- All open requests go to the server, so directory contents aren’t cached locally.
- Concurrent writes disable caching entirely; updates serialize at the server, which is acceptable because such sharing is rare.


## File Access Operations in Sprite

- All opens flow through the server; clients cache blocks locally and writers timestamp dirty blocks to honor the 30 s write-back policy.
- Clients track per-file cache status, cached blocks, dirty timers, and version numbers; the server tracks current readers, writer, and file version.
- When a new (sequential) writer arrives, the server asks the previous writer for any dirty data before handing off ownership; once the old writer closes, the new writer can cache normally.
- If another writer wants concurrent access, the server collects dirty blocks, then disables caching (using a per-file “cacheable” flag) so both writers operate directly through the server.
- After a writer closes, the server notices and re-enables caching for future access.

## Overall Summary

- Distributed filesystems provide a single filesystem interface over remote storage by connecting clients and servers (often via RPC) under the OS’s VFS layer.
- Data placement uses replication (high availability, complex writes) and/or partitioning (scales capacity, risks hotspots); many systems replicate each partition.
- Client-side caching is vital for latency and load but demands coherence policies (session semantics, periodic refresh, server delegations/invalidations).
- Stateless servers (e.g., NFSv3) keep no per-client state—simple yet incompatible with advanced caching/locking; stateful servers (NFSv4, Sprite) track readers, writers, versions, and cacheability to manage leases and coordinate writes.
- Real systems (NFS, Sprite) tune write-back intervals, callbacks, and lock models to balance performance gains from local caching with correctness guarantees across the network.

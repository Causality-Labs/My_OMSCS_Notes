
## Reviewing DFS

- DFS stores state (files) on server nodes and exposes file operations to clients; servers own and manage that state.
- The lesson focused on client/server caching to boost performance and scalability.
- It didn’t cover how multiple file servers coordinate shared state or peer-to-peer setups where nodes act as both clients and servers.

## Peer Distributed Applications

- In some systems, every node both serves and requests state: each holds a portion locally, and the global state is the union of all pieces.
- Nodes act as peers—consuming remote state and offering their own—though there may still be special nodes for configuration/management (unlike fully cooperative peer-to-peer setups).

## Distributed Shared Memory (DSM)

- Distributed shared memory (DSM) lets apps see multiple nodes’ RAM as one shared-memory machine.
- Each node owns some physical memory but any node can read/write it; consistency protocols order these accesses so all nodes agree.
- DSM offers a cheaper path to large memory than single huge machines, trading some latency for scalability (often mitigated by design and fast interconnects like RDMA).


## Hardware vs Software DSM

- Hardware DSM uses a special interconnect/NIC so each OS thinks it has a larger physical address space; remote accesses are turned into network messages transparently, and the NIC can even handle atomics
- .![](https://assets.omscs.io/notes/A0136738-FC8B-4E44-B4A1-1862ADDA3938.png)
- This hardware is expensive and mainly for high-end systems.
- Software DSM (in the OS or language runtime) must detect remote accesses, message the right node, perform the requested ops on arrival, and participate in sharing/consistency protocols.

## DSM Design: Sharing Granularity

- Cache-line coherence is too costly over the network for DSM, so sharing is managed at coarser granularity (variables, pages, or objects).
- Variables are still too fine-grained; pages work well for OS-level DSM, and objects fit language/runtime-level DSM (without OS changes).
- Coarser granularity risks false sharing (unrelated data on the same page/object triggers unnecessary coherence traffic).

## DSM Design: Access Algorithm

- DSM complexity depends on access patterns: single-reader/single-writer mainly needs remote access; multiple readers/writers require returning up-to-date values and ordering writes so all nodes see a consistent state.


## DSM Design: Migration vs Replication

- DSM performance hinges on access latency, so maximize local hits.
- Migration: move the needed data to the requesting node—good for single reader/single writer, but costly if done for one-off accesses.
- Replication: keep copies on multiple nodes for multi-reader/writer workloads, but then enforce ordering and freshness; limit replica count to reduce consistency overhead.

## DSM Design: Consistency Management

- DSM must keep replicated pages coherent, borrowing ideas from SMP: write-invalidate (invalidate others’ copies on write) and write-update (push new value).
- Doing this on every write over the network is too expensive.
- Two broad strategies: push invalidations eagerly/proactively (pessimistic) when data is written, or have nodes poll/check lazily/reactively (optimistic) for updates.
- The trigger points depend on the chosen consistency model.

## DSM Architecture

- DSM pools pages from multiple nodes into one global address space; each address is identified by (node, page frame), with a “home node” for each page.
- To support multiple readers/writers with low latency, nodes cache pages locally while the home node coordinates coherence, tracking access, modifications, cacheability, and locks.
- If a non-home node accesses a page heavily, ownership can shift to an “owner node” that drives coherence, while the home node just tracks current ownership.
- Beyond caching, explicit replicas may be kept (e.g., triplication across racks/sites) for load, performance, and reliability; consistency is managed by the home/manager node.


## Summarizing DSM Architecture

![](https://assets.omscs.io/notes/17D02718-3116-442E-8010-B82EF5CE2861.png)



## Indexing Distributed States

- DSM identifies each page by (node ID, page frame); the manager/home node for that page is derived from its address.
- A global map (page ID → manager ID) is replicated on every node; per-page metadata is partitioned and kept by each page’s manager.
- Basic scheme fixes the manager from address bits; a mapping table adds flexibility—update the table to remap managers without changing page IDs.
- ![](https://assets.omscs.io/notes/9EA39B5E-9AF1-4519-A1C2-D3EB9924491E.png)


## Implementing DSM

- DSM must intercept every shared-memory access: detect local vs remote and detect writes to trigger coherence.
- Use the MMU: missing mappings or write-protected pages cause traps; DSM handles remote fetches or coherence on write attempts.
- Avoid overhead for private memory by enabling DSM only for shared regions.
- MMU metadata (dirty/accessed bits) can help implement consistency policies.
- Language-level/object-based DSM can use similar OS traps or pure software mechanisms.


## What is a Consistency Model?

- DSM design hinges on the chosen consistency model—what ordering/visibility guarantees are promised if software follows specified rules.
- Correctness requires reads to reflect writes in a way that matches issued operation order and recency expectations; APIs and usage must align with the model’s guarantees.
- Notation for describing consistency behaviors:![](https://assets.omscs.io/notes/F9F33EA4-108F-44FC-BB04-FC92DE1CB897.png)


## Strict Consistency

- Strict consistency is the ideal: every write is seen by all nodes immediately and in the exact real-time order issued.![](https://assets.omscs.io/notes/0B6FCE54-C65D-4D3B-8D80-D3FCF8CF0E4E.png)
- In practice it’s unattainable—even SMP systems need locks for ordering, and distributed systems face latency, loss, and reordering—so weaker consistency models are used instead.

## Sequential Consistency

- Sequential consistency relaxes immediacy but enforces a single global interleaving of all operations that could have occurred, seen identically by every process.
- Writes from different processes may be interleaved arbitrarily, but a process’s own operations always preserve program order.
- All observers must see the same ordering (no showing different values for the same location to different readers).
- Diagrams:![](https://assets.omscs.io/notes/2FAC54FC-883E-45B2-88D6-94843AC1F9F1.png)![](https://assets.omscs.io/notes/09F1D347-5C52-4A76-BF50-0229E5B3BE2A.png)![](https://assets.omscs.io/notes/6A81FC78-9212-4BF3-81E7-C2207BC06C32.png)

## Causal Consistency

- Sequential consistency is stronger than needed; causal consistency only orders writes with potential cause-effect relationships.![](https://assets.omscs.io/notes/0A182699-4585-47FD-B1C7-AC089170ACD0.png)
- If a write depends on a prior read (e.g., P1 reads m1 then writes m2), all processes must see m1’s update before m2’s update.![](https://assets.omscs.io/notes/AF2CD37E-0407-4646-AB4E-D6F3A045A887.png)
- Concurrent, unrelated writes have no enforced order; different processes may see them in different sequences.![](https://assets.omscs.io/notes/FD6FC037-E098-41D7-8446-A703DEB6B344.png)
- Per-process program order is still preserved across the system.

## Weak Consistency

- Weak consistency adds explicit synchronization ops (sync/acquire/release) alongside reads/writes.![](https://assets.omscs.io/notes/B7846427-7BD2-4398-B69D-7BE7761F0BC6.png)
- A sync makes prior local updates visible to others and pulls in others’ prior updates for the caller—but both writer and reader must sync for visibility.![](https://assets.omscs.io/notes/5FDC8B58-6835-482E-A762-80163BFD2ACC.png)
- Systems may scope syncs (per variable/group) or split them into acquire (see others’ updates) and release (publish my updates) to reduce coherence overhead, tracking extra state in the DSM layer.



## Consistency Summary

- **Strict consistency:** Idealized, impossible in practice. Every write is seen by all nodes instantly and in real-time order.
- **Sequential consistency:** All nodes see the same single interleaving of operations; program order is preserved per process. Writes may be delayed, but everyone agrees on the order.
- **Causal consistency:** Only orders writes with potential cause-effect links (a write that follows a read). Concurrent, unrelated writes can be seen in different orders; program order is still preserved.
- **Weak/relaxed consistency:** Visibility is guaranteed only at synchronization points. Processes use ops like sync/acquire/release to publish or observe updates; between syncs, reads/writes may be stale.
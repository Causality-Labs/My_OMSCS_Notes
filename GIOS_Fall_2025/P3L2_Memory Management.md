

## Memory Management: Goals

- OS mediates DRAM for processes by presenting each with its own virtual address space.
- Virtual addresses usually outnumber available physical frames, so the OS must both allocate and arbitrate memory use.
- Allocation: track free vs used frames and swap data to/from disk when physical memory runs short.
- Arbitration: translate and validate virtual addresses rapidly, relying on hardware plus software.
- Paging maps fixed-size virtual pages to physical page frames via page tables; dominant approach today.
- Segmentation offers variable-sized regions managed with segment registers, but is less common than paging.


## Memory Management: Hardware Support

- Hardware teams with the OS so memory decisions remain fast, accurate, and enforced.
- Each CPU’s MMU translates every virtual address the CPU issues into a physical address.
- MMU faults catch illegal addresses, permission violations, or pages missing from RAM that must be fetched from disk.
- Translation helpers include dedicated registers for active page tables or segment descriptors.
- A **translation lookaside buffer** (TLB) caches recent translations so most lookups skip a full table walk.
- Hardware performs the translation itself, so supported modes (paging, segmentation) depend on CPU design.
- ![](https://assets.omscs.io/notes/3A5AF4E6-D08B-4627-8382-6B7E6108065E.png)


## Page Tables

- Paging maps virtual addresses to physical frames via per-process page tables.
- ![](https://assets.omscs.io/notes/F4D30C66-2702-4D92-95A0-57F1A761B83C.png)
- Equal virtual page and physical frame sizes let translation use a virtual page number (VPN) plus offset; VPN → PFN lookup, then PFN + offset yields the physical address.
- ![](https://assets.omscs.io/notes/6C1CB2E4-D8C7-4F2C-BC0C-831E2DF40CCD.png)
- Physical memory is provisioned lazily—allocation on first touch—by mapping untouched virtual pages to free frames only when accessed.
- ![](https://assets.omscs.io/notes/6785C624-4B55-4F1F-AED0-B9C31954877A.png)
- Reclaimed or swapped-out pages clear their valid bit; accesses with valid=0 trigger a page fault so the OS can locate/load the page and remap it.
- Each process owns its page table; context switches swap the active table pointer, tracked in hardware (e.g., x86 CR3).
- ![](https://assets.omscs.io/notes/1234345B-1FE2-4424-A6FB-7E13693B22EE.png)

## Page table entry

- Each page-table entry (PTE) stores at least a physical frame number (PFN) plus a valid/present bit indicating whether the page resides in DRAM.
- Additional PTE flags guide policy and hardware checks: dirty bit records writes for cache/file sync; access/reference bit tracks any read/write touch; protection bits encode read/write/execute permissions.
- ![](https://assets.omscs.io/notes/761F6331-A7C2-44BF-B516-92A921D1483B.png)
- MMU uses these bits during translation; invalid accesses raise a page fault.
- Fault handling: CPU pushes an error code, traps into the kernel’s page-fault handler, which inspects the code and faulting address to decide whether to fetch the page or deny access.
- On x86, the error code derives from PTE flags and the fault address is reported via CR2.

## Page Table Size

- Page table size = number of virtual page numbers (VPNs) or no of PTE × size of each page-table entry (PTE).
- 32-bit example:
    - Address space: 2³² bytes (4 GB).
    - Page size: 4 KB (2¹²).
    - VPN count: 2³² / 2¹² = 2²⁰ entries.
    - PTE size: 4 bytes → table needs 2²⁰ × 4 B = 2²² B ≈ 4 MB per process.
- 64-bit example:
    - Address space: 2⁶⁴ bytes.
    - Same 4 KB pages → 2⁶⁴ / 2¹² = 2⁵² entries.
    - PTE size: 8 bytes → table would require 2⁵² × 8 B = 2⁵⁵ B ≈ 32 PB per process.
- Since page tables are per-process, naïvely allocating entries for every possible VPN is wasteful—most processes use only a fraction of the theoretical address space.

## Hierarchical Page Tables

- Modern systems favor hierarchical (multi-level) page tables over flat tables to save space.
- ![](https://assets.omscs.io/notes/Screen%20Shot%202018-10-19%20at%201.47.34%20PM.png)
- A page-directory (outer level) points to inner page tables, which hold PFNs and protection bits.
- Only regions actually mapped allocate inner tables; gaps in virtual space require no backing tables.
- New allocations (malloc) may trigger creation of additional inner tables and directory entries.
- Virtual addresses split into multiple indices plus an offset: directory index → page-table index → PFN, offset selects bytes within the frame.
- ![](https://assets.omscs.io/notes/9CB5D47B-9A63-46A9-9117-E147205AD34C.png)
- Example format: 12-bit outer index, 10-bit inner index, 10-bit offset; each table covers 2¹⁰ entries × 2¹⁰ bytes = 1 MB.
- ![](https://assets.omscs.io/notes/94D5824C-54DD-4118-A769-6AD8B2847A94.png)
- Large unmapped spans (≥1 MB) avoid allocating intermediate tables, unlike flat tables requiring full coverage.
- Additional levels (third, fourth) further localize coverage, crucial for sparse 64-bit address spaces.
- ![](https://assets.omscs.io/notes/2D989531-A664-498C-BEDE-D5A0BE50C8F1.png)
- Trade-off: more levels reduce table size but increase page-translation latency due to extra memory references.

## Speeding Up Translation TLB

- More page-table levels mean more memory references: single-level needs two (PTE + data), four-level needs four table lookups before the actual data access.
- Hardware mitigates this cost with a Translation Lookaside Buffer (TLB), a cache of recent VPN→PFN translations (plus protection bits).
- On each access, the MMU consults the TLB first; hits avoid page-table walks, misses trigger the full multi-level lookup.
- TLB entries also enforce permissions/validity, generating faults when required.
- Thanks to temporal/spatial locality, even small TLBs deliver high hit rates.
- Typical modern x86 core: 64-entry data TLB, 128-entry instruction TLB, and a shared 512-entry second-level TLB.

## Inverted Page Tables

- Traditional per-process page tables map each process’s virtual pages to frames; total “virtual memory” scales with process count.
- Inverted page tables flip the viewpoint: one system-wide table indexed by physical frames, each entry noting which process and virtual page occupy that frame.
- Logical addresses under this scheme include PID + virtual page + offset; lookup scans the inverted table for a matching PID/page, then uses the entry index (frame number) plus offset.
- ![](https://assets.omscs.io/notes/598B0234-CFB4-47D7-9356-73432E59A65C.png)
- Linear scans are expensive, though TLB caching hides some cost; to accelerate misses, inverted tables pair with hashed page tables that map address hashes to small candidate lists.
- ![](https://assets.omscs.io/notes/5B8687C4-AA22-40E8-920C-B5B021652EAE.png)
  

## Segmentation

- Segmentation divides a process’s address space into variable-sized regions (code, data, heap, stack).
- Logical addresses include a segment selector plus offset; the selector indexes a descriptor table to fetch the segment’s base/limit, which combine with the offset to produce a linear address.
- Pure segmentation could map segments directly onto contiguous physical memory via base + limit registers.
- Real systems often layer segmentation over paging: segmentation yields the linear address, paging maps it to a physical frame.
- Hardware dictates support: x86-32 uses both segmentation and paging (Linux allows up to 8K per-process and 8K global segments); x86-64 keeps segmentation mainly for legacy mode, relying on paging by default.
- ![](https://assets.omscs.io/notes/B38534C8-2529-4E6F-AC22-3C6989B13194.png)


## Page Size

- Page size equals 2^(offset bits): e.g., 10-bit offset → 1 KB; 12-bit → 4 KB.
- Linux on x86 commonly uses 4 KB pages (default), but also supports 2 MB “large pages” (21-bit offset) and 1 GB “huge pages” (30-bit offset).
- Larger pages shrink VPN space, yielding smaller page tables (4 KB baseline → 2 MB cuts entries by ×512; 1 GB cuts ×1024) and improving TLB hit rates.
- Trade-off: bigger frames risk internal fragmentation—unused space inside a page when allocations don’t fill it.
- Supported sizes vary by platform; e.g., Solaris 10 on SPARC handles 8 KB, 4 MB, and 2 GB pages.

## Memory Allocation

- Memory allocation decides which physical pages back each virtual region, with allocators operating in both kernel and user space.
- **Kernel allocators** supply pages for the kernel itself and static process regions (code, stack) while tracking overall free memory.
- **User allocators** (e.g., malloc/free) manage dynamically requested heap memory, drawing from pages granted by the kernel.
- After the kernel hands pages to a process via malloc, user-space allocator controls their lifetime and reuse.

## Memory Allocation Challenges

- Scenario: 16 page frames, requests for one 2-frame block and three 4-frame blocks; allocations must be contiguous.
- ![](https://assets.omscs.io/notes/FF84D314-D11E-4B0D-9883-5A4B95B81696.png)
- Naïve placement can satisfy initial requests but, after freeing the 2-frame block, leaves dispersed free frames that can’t service a new 4-frame request.
- ![](https://assets.omscs.io/notes/2E0A9A0A-A838-4A7F-B3F7-AB1A90BC8718.png)
- ![](https://assets.omscs.io/notes/D858CB66-A95B-4EFB-84D1-AD3448FEBEC5.png)
- This fragmentation of free space is **external fragmentation**—total free frames exist, yet none form the required contiguous extent.
- Better allocation layout coalesces free frames (e.g., grouping 4-frame blocks together), ensuring future large requests succeed.
- ![](https://assets.omscs.io/notes/9E026C20-E04E-4CE6-B31F-F4EB073272EA.png)
- Key takeaway: allocators should place and coalesce blocks to preserve large contiguous free regions and limit external fragmentation.

## Linux Kernel Allocators

- Linux kernel uses two allocators: buddy for power-of-two page blocks, slab for fixed-size object caches.
- Buddy allocator recursively splits a free 64-unit region into halves (32→16→8→4) to satisfy successive 8- and 4-unit requests; freed buddy pairs merge back up, limiting but not eliminating fragmentation.
- Power-of-two granularity eases buddy lookup (addresses differ by one bit) yet causes internal waste when object sizes like the 1.7 KB task_struct don’t align.
- ![](https://assets.omscs.io/notes/ABC594B2-8DD5-4767-8E68-E91770CC47D5.png)
- Slab allocator layers caches on contiguous slabs, pre-creating object pools (e.g., task_struct); allocations pull exact-size objects, and new slabs appear only when caches empty.
- ![](https://assets.omscs.io/notes/4D7BC1F3-90FB-44C7-8D77-D9D128DB7B62.png)
- Slabs virtually eliminate internal/external fragmentation because each slot stores exactly one object of the designated type.


## Demand Paging

- Physical memory is smaller than the virtual space, so pages can live on disk and be fetched as needed—**demand paging** swaps pages between DRAM and a swap partition.
- A page swapped out has its present bit cleared; referencing it triggers a page fault that traps into the kernel.
- Kernel locates the page on disk, issues an I/O read, finds a free frame (not necessarily the original), updates the PTE’s PFN, and resumes the faulting instruction.
- ![](https://assets.omscs.io/notes/76DE9169-A186-412A-B18C-EBD53058BB3A.png)
- Some pages must never be swapped or must keep a fixed PFN (e.g., DMA buffers); these are **pinned** to remain resident in physical memory.

## Page Replacement

- Page-out work runs when memory usage exceeds a threshold and CPU load is low enough for the daemon to scan for reclaimable pages.
- Goal: evict pages unlikely to be used soon; LRU-style policies rely on hardware access bits to approximate recency.
- Prefer clean pages to avoid disk writes; dirty pages (tracked via dirty bit) must be flushed first.
- Non-swappable pages must be excluded from eviction.
- Linux tunables cover swap thresholds, page-replacement rates, and page categories (claimable, swappable) to guide decisions.
- Default Linux replacement is a two-pass, second-chance LRU variant that gives pages another look before eviction.

## Copy on write 

- MMU capabilities (translation, protection, access tracking) enable optimizations like **Copy-on-Write (COW)**.
- Process creation initially maps child virtual pages to the parent’s physical pages, with write protection to monitor modifications.
- ![](https://assets.omscs.io/notes/1763C799-DD05-4442-95AD-F57F32314875.png)
- Reads proceed without copying, saving memory and CPU work.
- First write triggers a fault; OS allocates a private page, copies data, updates the writer’s page table, and clears protection on the new page.
- Only pages actually written incur the copy cost—hence “copy on write.”

##  Failure Management Checkpointing

- Checkpointing saves process state periodically so failures can restart from a recent snapshot instead of from scratch.
- Simple/Naïve checkpointing pauses execution and copies the entire address space; smarter schemes write-protect pages, copy once, and then track dirty pages via the MMU to capture only diffs (incremental checkpoints).
- Incremental checkpoints complicate recovery because the full state must be rebuilt from multiple deltas.
- The same techniques underpin other services:
    - **Rewind-replay** debugging restarts from an earlier checkpoint and replays execution to reproduce bugs.
    - **Process migration** checkpoints state to another machine and resumes there, useful for disaster recovery or consolidation.
- Migration often loops through rapid incremental checkpoints until remaining dirty pages are small enough for one final pause-and-copy.
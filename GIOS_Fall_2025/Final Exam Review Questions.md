
# P3L1 Scheduling

[[P3L1_Scheduling]]

 **How does scheduling work?** [[P3L1_Scheduling#Scheduling Overview]]
 
 * The **CPU scheduler** decides how and when the processes (and their threads) access the shared CPUs.
* Scheduler schedules **user level and kernel level threads**.
*  Scheduler ensures CPU time is shared fairly and respects policy semantics.

**What are the basic steps and data-structures involved in scheduling a thread on the CPU**

- Core **data structure** is the **runqueue**: any structure (single queue, multi-queue, tree) that logically **orders runnable tasks**.
- **Policy** dictates **runqueue design**: FCFS, round-robin, shortest-job-first, or per-priority queues.

- Scheduling steps:
	- pick task from runqueue, 
	- dispatch to CPU, 
	- preempt if needed when higher-priority work arrives.




**What are the overheads associated with scheduling?**
* primary overhead with scheduling is one of **timing**
	* Specifically, the time to **context switch** between threads introduces some non-trivial amount of delay into the system
	* Context-switching frequently helps to **reduce the wait time** for any given task in the queue
		* but **frequent** intervention will **drop the throughput** and **increase the average completion time**



**Do you understand the tradeoffs associated with the frequency of preemption and scheduling/what types of workloads benefit from frequent vs. infrequent intervention of the scheduler (short vs. long timeslices)?**

* Workloads that are primarily **CPU-bound** will **benefit** from **longer timeslices**
	* They utilize the CPU completely when they are scheduled
* Workloads that are primarily **I/O-bound** will **benefit** from s**horter timeslices**
	* **frequent context-switching** can help to **hide latency**

 

**Can you work through a scenario describing some workload mix (few threads, their compute and I/O phases) and for a given scheduling discipline compute various metrics like average time to completion, system throughput, wait time of the tasks…**

(Formulas Will do later)

**Do you understand the motivation behind the [[P3L1_Scheduling#Runqueue Data Structures| multi-level feedback queue]]**
* To have a **single structure** that maintained **different timeslice levels** that threads could be easily moved through
	* Combines priority levels with distinct **policies**, using **feedback** to place each task on the most appropriate queue over time
**Why different queues have different timeslices**
* Queues have different timeslices because **tasks have different needs**.
	* **CPU-intensive tasks** are better off have **large timeslices**.
	* **I/O-intensive tasks** are better off having a **shorter timeslice**.
**How do threads move between these queues**
* The scheduler can **move threads** through this structure based on **preemption vs. yield observation**
	*  When a thread is **consistently preempted**, this is inline with the assumption that the thread is **compute-bound**. 
		* The scheduler can move this task to the end of a queue with a longer timeslice. 
	* When a thread c**onsistently yields** the CPU, this suggests that the thread is **I/O bound**. 
		* This thread should be moved to a queue with a smaller timeslice.
**Can you contrast this with the [[P3L1_Scheduling#Linux O(1) Scheduler | O(1) Scheduler]]?**
* O(1) scheduler uses **140 time-sharing priority levels**.
	* lower priority level =  higher time slice  (level 0 has highest time slice)
	* higher priority level = lower time slice (level 139 has lowest time slice)
- **Compute-bound** tasks get the **smallest timeslices**; 
- **I/O Interactive** tasks get the **largest**.
- Priorities adjust based on **sleep time**: CPU-bound tasks (no sleep) drop 5 levels; I/O-bound tasks (long sleep) rise 5 levels.
- **Each priority level** has an **active queue (currently runnable)** and an **expired queue**.
- Tasks run until their **full timeslice expires**, then move to the expired queue; when the active queue empties, the t**wo queues swap roles**.

**Do you understand what were the problems with the O(1) scheduler which led to the [[P3L1_Scheduling#Linux CFS Scheduler |CFS]]?**
* O(1) scheduler drawback: tasks in the expired queue wait until all active tasks finish their quanta, causing jitter for interactive workloads and no fairness guarantee.
* Fairness goal: each task should receive CPU time proportionate to its weight/priority within a window.
* CFS runqueue: self-balancing red-black tree ordered by virtual runtime (vruntime).
	*  Scheduling rule: always pick the leftmost (smallest vruntime) node; running task’s vruntime increases, and if it surpasses the new minimum, it is reinserted.
	- Priority impact: lower-priority tasks accumulate vruntime faster (time “moves faster”), so they’re preempted sooner; higher priorities advance more slowly.
	- Complexity: selecting next task remains O(1); insertions cost O(log n).
	- Potential future pressure: if task counts explode, even logarithmic inserts might prompt searches for fair schedulers with constant-time operations.

**Thinking about Fedorova’s paper on scheduling for [[P3L1_Scheduling#Scheduling on Multi-CPU Systems|chip multi processors]], what’s the goal of the scheduler she’s arguing for?**
- Fedorova **targets schedulers** for **hardware multithreaded CPUs** that **maximize utilization**.

**What are some performance counters that can be useful in identifying the workload properties (compute vs. memory bound) and the ability of the scheduler to maximize the system throughput.**
- Utilization is usually measured by **instructions per cycle (IPC)**; **CPU-bound** work shows **IPC ≈ 1**, **memory-bound** work shows **IPC near 0**.
- Hardware counters report IPC per context; Fedorova proposes also tracking **cycles per instruction (CPI = 1/IPC)**.
- **Compute-bound** tasks still have **CPI ≈ 1**, while **memory-bound** tasks have **CPI » 1**, so mixing CPIs can keep overall IPC near 1.
- **Synthetic workloads** showed **CPI** helps **predict better mixes**, but **real workloads** exhibit **smaller** **CPI variation**.
- **IPC counters** **identify** whether a task is **compute vs memory dominated;** cache-miss counters reinforce the diagnosis (many misses → memory-bound, few → compute-bound).


# P3L2 Memory Management

[[P3L2_Memory Management]]

**How does the OS map the memory allocated to a process to the underlying physical memory?**
- The OS maps the memory allocated to a process to the underlying physical memory primarily via page tables.
	- **Page tables** are kernel-maintained data structured that **map virtual addresses to physical addresses**
	- The **Memory Management Unit** (MMU) uses the mapped physical address to perform the memory access.

**What happens when a process tries to access a page not present in physical memory?**
- If the page is not present in physical memory (i.e. it has been swapped out),  the **MMU** will **generate a fault**.
- The OS will see that the **present bit is set to 0**, and will pull in the page from some **secondary, disk-based storage**.

**What happens when a process tries to access a page that hasn’t been allocated to it?**
- If a process tries to access a page that hasn't been allocated to it, the MMU will again generate a fault. 
- The OS will detect that this page hasn't been allocated and will perform some corrective action, potentially passing the fault along to the offending process as a SIGSEGV.

 **What happens when a process tries to modify a page that’s write protected/how does COW work?**
 - Attempting to write to a protected page triggers an MMU fault that would normally terminate the program.
 - This may happen frequently in the case when a child process has been forked from a parent process.
	 - Since a lot of the address space of the parent contains static data that will be used by the child, it doesn't make sense to explicitly copy all of the data over to a new address space of the child process.
	 - Instead, the OS will often just point the child to the page tables for the parent.
	 - Additionally, the OS will write-protect the memory associated with the parent.
		 - If the child tries to write to a page, the MMU will trap into the kernel, at which point the OS will copy over the page(s) to the child address space. 
		 - This process is called copy-on-write(COW) and makes sure that a child only copies over the pages it ends up writing.

**How do we deal with the fact that processes address more memory than physically available?**
- If we have a 32-bit architecture, with 4kB pages, each process can address 4GB of memory. 
- If we have 8GB of RAM, 3 concurrent processes can (if they really wanted to) exhaust our RAM and then some. 
- To deal with the situation where processes address more memory than is physically available, we have to introduced s**wapping/demand paging**.

**What’s demand paging?**
- Demand paging swaps selected pages to secondary storage (e.g., disk).
- Swapped pages get their present bit cleared, so any access triggers an MMU trap into the kernel.
- The kernel reloads the needed page into DRAM, possibly at a different physical address—virtual addressing hides this change.
- After reload, the CPU resumes at the faulting instruction, which now completes successfully.

**How does page replacement/swapping work?**
- A page swapping daemon selects pages to evict, typically using least-recently-used (LRU) heuristics because recently accessed pages are likely to be used again soon.
- The daemon checks the access bit to estimate recency and the dirty bit to avoid writing clean pages back to disk unnecessarily.
- Linux implements a second-chance LRU variant that performs two scans before actually swapping a page to disk.



**What’s the role of the TLB?**
-  Virtual addresses are first passed to the Translation Lookaside Buffer (TLB), a hardware-maintained cache of address translations.
	-  **TLB Hit:** If the physical address is present in the cache, access proceeds immediately.
	- **TLB Miss:** If not present, the OS walks the page table to find the translation.
	- **Update:** Once found, the mapping is written to the TLB, and access proceeds through the TLB
**How does address translation work?**
-  A virtual address typically consists of a Virtual Page Number (VPN) and an offset.
-  The VPN indexes into the page table to produce a Physical Frame Number (PFN).
- The PFN is combined with the offset to create the exact physical address.

**Do you understand the relationships between the size of an address, the size of the address space, the size of a page, the size of the page table…**
- For an address size of X bits, a process can address 2^X memory locations. 
- For a page-size of 2^Y bits, a page table will have 2^(X-Y) entries. 
- Since each entry in a page table is X bits, a page table will occupy 2^(X-Y) * X bits.
- In a 32-bit architecture, where addresses are 32 bits long, each page table can address up to 2^32 (4GB) of data. Since a physical memory page is often 4KB in size, this page table would have 2^20 entries. Since each entry is again 4 bytes (32 bits), the entire page table is 4MB in size.

**Do you understand the benefits of hierarchical page tables?**
- Hierarchical page tables allow us to map virtual addresses to physical addresses without maintaining an entry for every virtual address. 
- With a two level page tables, we have a page table directory whose entries are page tables. Each page table ends up addressing a smaller region of physical memory. 
- When a process needs to allocate some memory, the OS may address from an existing page table or it may generate a new page table. This way, the amount of mapping that the OS maintains more accurately reflects the amount of memory the process is actually using.

**For a given address format, can you workout the sizes of the page table structures in different layers?**
- Let's look at an example with 32 bit addresses that are broken into a 12 bit segment, a 10 bit segment, and a 10 bit segment. 
- The first segment indexes into the page table directory. This means that the OS can maintain 2^12 page tables in the directory. Each page table maintains 2^10 entries, with each entry able to address 2^10 bytes. This means that each page table can address 1MB.

# P3L3 IPC

[[P3L3_IPC]]

**For processes to share memory, what does the OS need to do?**
- For processes to share memory, the operating system needs to map some segment of physical memory into the virtual address spaces of the processes.

**Do they use the same virtual addresses to access the same memory?**
- There is no guarantee (and it is likely impossible) that processes will use the same virtual addresses to access the same memory.
	- It's the page tables that are important here. 
	- The OS needs to set up page tables such that some subset of virtual addresses for some group of processes point to the same block of physical memory. 
	- The virtual addresses can be "arbitrary": it's the mapping that matters.


**For processes to communicate using a shared memory-based communication channel, do they still have to copy data from one location to another?**
- When processes communicate with one another via a memory-based communication channel, data copying may be reduced but it is often not eliminated. 
- For data to be available to both processes, a process must move the data to a portion of the address space that points to underlying shared physical memory. 
- Data needs to be explicitly allocated from the virtual addresses the belong to the shared memory region. Otherwise, the data is local to the process.

**What are the tradeoffs between message-based vs. shared-memory-based communication?**
- In message-based IPC, the benefit is the simplicity. 
	- The developer doesn't have to manage any synchronization or any actual data management or channel setup. The OS handles everything. 
	- However, this comes at a cost. We have to context switch to and from the kernel to pass a message from one process to another, and this involves copying data twice: into and out of the kernel. This is relatively expensive.
- In shared-memory IPC, the benefit is the relatively low cost. 
	- While the shared memory mapping that the OS creates is relatively expensive, that cost is amortized over the lifetime of that memory channel. 
	- However, since the OS is completely out of the way, it's up to the developer to manage everything. Synchronization, channel management and data movement are no longer free.


**What are different ways you can implement synchronization between different processes (think what kids of options you had in Project 3).**
- Mutexes
- Conditional Variables
- Semaphores
- Message queues


# P3L4 Synchronization Constructs

[[P3L4_Synchronization Constructs]]

 **To implement a synchronization mechanism, at the lowest level you need to rely on a hardware atomic instruction. Why?**
 - Basically, we need atomic operations because the checking of a lock and the setting of a lock require multiple CPU cycles. 
 - We need these cycles to be performed together or not at all: atomically.
 
 **What are some examples?**
 - `test_and_set`, `read_and_increment`, and `compare_and_swap`

**Why are spinlocks useful?**
- On one hand, spinlocks are very simple to implement. 
- In addition, they are pretty simple to understand. You just burn through CPU cycles continuously checking the value of the lock until you are preempted. 
- As well, spinlocks have very low latency and very low delay. This can have some performance considerations.

**Would you use a spinlock in every place where you’re currently using a mutex?**
- You probably wouldn't want to use spin locks every place you are currently using a mutex, however. What's nice about mutexes is that, after checking a lock that isn't free, the process will immediately relinquish the CPU. 

**Do you understand why is it useful to have more powerful synchronization constructs, like reader-writer locks or monitors?**
- These constructs are more powerful because of their expressiveness. 

**What about them makes them more powerful than using spinlocks, or mutexes and condition variables?**
- Using a reader-writer lock instead of a two binary semaphores to control the access of shared state allows you to express implement the semantics of your application with higher-level code. 
- **Reduced Error Prone-ness:** Higher-level constructs abstract away complex details, minimizing the chance for common mistakes like locking the wrong mutex or signaling incorrect conditions.
- **Improved Expressiveness:** Unlike mutexes which often require manual proxy variables, these constructs directly support complex application semantics, making code cleaner and safer.

**Can you work through the evolution of the spinlock implementations described in the Anderson paper, from basic test-and-set to the queuing lock?** **Do you understand what issue with an earlier implementation is addressed with a subsequent spinlock implementation?**
- Test and Set
	- **Low Latency & Delay:** It spins directly on the atomic instruction, allowing it to detect a freed lock and execute immediately.
	- **High Contention:** It performs poorly under contention because every spin requires a memory access, flooding the shared interconnect with traffic.
	- ![](https://assets.omscs.io/notes/5CB26CEB-E617-4CBC-8123-3375C2C6875D.png)
- Test and test and set
	- Spins on Cache: It spins on the cached value rather than the atomic instruction, slightly increasing latency and delay compared to the previous lock.
	- Contention Varies: Performance depends heavily on the cache coherence strategy:
		* Write-Update: Good performance, as hardware updates the cache automatically.
		* Write-Invalidate: Awful performance, as every acquisition attempt generates significant coherence traffic.
	* ![](https://assets.omscs.io/notes/C46BDD52-2F95-4EB4-8FA3-D10C244CF546.png)
* Delay lock
	* Improved Contention: It introduces an artificial delay to prevent all threads from attempting the atomic instruction simultaneously, significantly reducing contention.
	* Worse Delay: While basic latency remains acceptable, the artificial wait time makes the overall delay much worse compared to other locks.
	* ![](https://assets.omscs.io/notes/525BCCBF-AF42-4E79-A432-B1108B420862.png)
- Queueing lock
	-  **Array-Based Structure:** Uses an array of flags where each thread is assigned a unique index (ticket) acting as a private lock.
	- **Atomic Ticket Assignment:** Threads join the queue by atomically incrementing a `queuelast` pointer (requires `read_and_increment` support).
	- **Local Spinning:** Threads spin only on their specific array element (`must_wait`), significantly reducing contention.
	- **Direct Handoff:** Releasing the lock involves setting the *next* thread's flag to `has_lock`, ensuring low delay.
	- **Trade-offs:** Offers superior contention and delay performance but requires specific hardware support and consumes more memory (linear space complexity).
	- 
	- ![](https://assets.omscs.io/notes/16939119-DCB7-4091-9ADF-06204B9178B4.png)

# P3L5 IO Management

[[P3L5_IO_Management]]


 **What are the steps in [[P3L5_IO_Management#Typical Device Access|sending a command to a device]] (say packet, or file block)?**
- make a system call specifying the appropriate operation. 
- The kernel will execute some formatting steps on the request before passing them down to the device drivers. 
- The device drivers will interact with the device itself via PIO or DMA. Alternatively, in OS bypass, a user-level linked library can be utilized by the application to write directly to the device registers.
- For the information to travel back to the CPU, it can take two routes.
	- The device can generate an interrupt to trap into the kernel. This allows the CPU to see information about a request as soon as it is available, but incurs some interrupt handling overheads.
	- Alternatively, the CPU can poll some status registers associated with a device

**What are the basic differences in using programmed I/O vs. DMA support?**
- In PIO, The CPU writes commands and data directly to device registers. While this requires no extra hardware, it is CPU-intensive for large transfers as the CPU must manage every chunk of data.
- Alternatively, with direct memory access, The device reads/writes directly to a memory buffer established by the CPU. This frees up the CPU during transfers but requires a dedicated DMA controller.

**For block storage devices, do you understand the basic [[P3L5_IO_Management#Virtual File System|virtual file system stack]], the purpose of the different entities?**
![](https://assets.omscs.io/notes/DE9A5B7E-FDE3-4C19-9D65-7BFFDC72A9B0.png)
- At the top there is the file API that user applications interact with. 
- Below the file interface is the filesystem. The filesystem will actually interact with the block devices via their device drivers. 
	- Different devices will have different drivers, which can have different APIs. As a result, the final level above the device drivers is the generic block layer, which provides a standard API to the filesystems for interacting with storage devices. 
	- In a virtual filesystem, there is a VFS layer on top of the actual filesystems that gives the appearance of a single cohesive filesystem.

**Do you understand the relationship between the [[P3L5_IO_Management#inodes|various data structures]] (block sizes, addressing scheme, etc.) and the total size of the files or the file system that can be supported on a system?**
- The file is the main abstraction on which the VFS operates. 
- The file is represented to the higher level applications by an integral file descriptor. Each file is maintained by the VFS through an [[P3L5_IO_Management#inodes|inode]] structure.
	- Inodes holds an index of all of the blocks for a given file. 
- To help with operations on directories, the OS will maintain a structure called a dentry which corresponds to a single path component that is being traversed. 
- Finally, there is a superblock that maintains overall information about how a filesystem is laid out on disk.

- The size of the inode can directly determine the size of the file. 
	- For example, if we have a 128B inode containing 4B pointers addressing 1kB blocks, we can only have files that are 32kB in size. To get around this, we can use indirect pointers. For example, we can point to a block of pointers.
	- A 1kB block of 4B pointers points to 256 blocks of 1kb of memory lets us address 256kB of memory.

 **For the virtual file system stack, we mention several [[P3L5_IO_Management#Disk Access Optimizations|optimizations]] that can reduce the overheads associated with accessing the physical device. Do you understand how each of these optimizations changes how or how much we need to access the device?**
 - Filesystems can use caching to place blocks in main memory and reduce the  the number of disk accesses. 
 - Filesystems can also utilize I/O scheduling. One of the largest overheads is actually moving the disk head, so the I/O scheduler can reassemble requests such that the disk head has to backtrack as infrequently as possible.
 - In addition, we can utilize prefetching. 
	 - Instead of just fetching a block that is requested, it may be optimal to fetch that block and the next three logical blocks (the inode can help us understand what those blocks are). 
 - In addition, we can use journalling. Instead of immediately writing every request to disk, we can instead apply them to a log. These writes can then periodically be applied to the proper disk locations.

# P3L6 Virtualization

[[P3L6_Virtualization]]

**What is [[P3L6_Virtualization#Defining Virtualization|virtualization]]? What’s the history behind it?**
 Virtualization lets multiple operating systems share one physical machine while each “guest” believes it owns the hardware.
- ![](https://assets.omscs.io/notes/A96F5139-EFB1-4038-A5E4-9128C63CF5F8.png)
- A virtual machine (VM) bundles an OS, its apps, and virtual hardware; guests/domains run side by side.

**What’s hosted vs.[[P3L6_Virtualization#Virtualization Models Bare-Metal|bare-metal]] [[P3L6_Virtualization#Virtualization Models Hosted|virtualization]] ?**
- Bare-metal (type 1) hypervisors sit directly on hardware, managing all resources for hosted VMs.![](https://assets.omscs.io/notes/A320790E-26BD-4370-8B87-47E08D04FA9C.png)
- Device support is tricky: hypervisors need drivers for every device. Solution: run a privileged service VM with full hardware access to host drivers and management utilities.

- Hosted (type 2) virtualization runs on top of a full host OS; the VMM module rides inside that OS and presents virtual hardware to guests while calling into existing drivers/services.![](https://assets.omscs.io/notes/2BE5A89A-9473-420B-B659-D5F010B2BB8D.png)
- Hosts can run native apps alongside guest VMs; less VMM code is needed since hardware management stays with the host OS.


**What’s [[P3L6_Virtualization#Paravirtualization|paravirtualization]], why is it useful ?**
- Paravirtualization modifies the guest OS so it knows it’s running atop a hypervisor and avoids privileged ops that would fail.
- Instead, it issues explicit hypercalls (analogous to syscalls) to request services; the hypervisor handles the request and returns control.

**What were the [[P3L6_Virtualization#x86 Virtualization in the past|problems with virtualizing x86]]?**
- Seventeen privileged instructions (e.g., POPF/PUSHF to toggle interrupts) failed silently outside ring 0—no trap, no hypervisor visibility.
- Guests believed the operations succeeded, leaving the hypervisor unable to emulate the intended behavior.

**How does protection of x86 used to work and how does it work now?**
- In the past
	- Pre-2005 x86 offered only rings; hypervisors took ring 0, pushing guest OSes to ring 1.
	- Seventeen privileged instructions (e.g., POPF/PUSHF to toggle interrupts) failed silently outside ring 0—no trap, no hypervisor visibility.
	- Guests believed the operations succeeded, leaving the hypervisor unable to emulate the intended behavior.
- Now:
	- [[P3L6_Virtualization#Binary Translation|Binary Translation]]
		- Binary translation rewrites guest code on the fly to avoid the 17 problematic x86 instructions while keeping the guest OS unmodified (“full virtualization”).
	- Paravirtualization
		- Paravirtualization modifies the guest OS so it knows it’s running atop a hypervisor and avoids privileged ops that would fail.
		- Instead, it issues explicit hypercalls (analogous to syscalls) to request services; the hypervisor handles the request and returns control.


**How were/are the virtualization problems on x86 fixed?**
- The virtualization problems on x86 were eventually fixed by the chip makers, which ensured that the 17 privileged hardware instructions did generate a trap. In the meantime, there were two additional solutions.

**How does device virtualization work?** 
- CPU/memory virtualization is simpler because ISAs (e.g., x86) are standardized—hardware differences hide beneath that level.
- Device virtualization is harder: hardware varies widely and interfaces lack consistent semantics.
- Hypervisors therefore rely on three main models to virtualize devices (introduced next).

**What a [[P3L6_Virtualization#Passthrough Model|passthrough]] vs. a [[P3L6_Virtualization#Split Device Driver Model|split-device model]]?**
- Passthrough
	- Passthrough (VMM-bypass) maps a device’s registers directly into a VM, giving that guest exclusive control.
	- ![](https://assets.omscs.io/notes/DB8B54EB-D3C3-4EB0-A8F4-71163B1426E0.png)
	- Sharing becomes awkward: the hypervisor must reassign the device between VMs, so concurrent access is impractical.
	- Guests must see exactly the device model they expect, eliminating virtualization’s usual hardware abstraction.
	- Binding a device to one VM complicates live migration because device-resident state has to move with the guest.

- Split-device Model
	-  Split drivers divide device control: a paravirtualized front-end runs in each guest VM, packaging requests for a back-end driver in the service VM/host.![](https://assets.omscs.io/notes/BECB3588-AFBF-42F2-9A1C-22916449B1B4.png)
	- Back-end driver can be the native OS driver; only the front-end needs custom code, limiting the model to paravirtual guests.
	- Benefits: no device emulation overhead and centralized back-end control simplifies device sharing and management.


# P4L1 RPC

[[P4L1_RPC]]

**What’s the [[P4L1_RPC#Why RPC|motivation]] for RPC?**
- Many client/server apps repeat the same boilerplate (connection setup, request/response flow); only the request details differ.

**What are the various [[P4L1_RPC#Steps in RPC|design points]] that have to be sorted out in implementing an RPC runtime (e.g., binding process, failure semantics, interface specification… )?**
- **Binding Options:**
    - **Global Registry:** Clients interact with a central registry. This creates a single point of high traffic maintenance.
    - **Machine-Based Registry:** Clients interact with a registry on a specific machine. This requires the client to know beforehand which machine the server is running on.
- **Failure Semantics:**
    - It is difficult for the RPC runtime to deliver meaningful failure messages because many potential errors occur outside the control of the RPC program itself.
- **Interface Specifications (IDLs):**
    - **Language-Specific:** Beneficial if you already know the language and want to avoid learning a new syntax, but limits interoperability.
    - **Language-Agnostic:** Beneficial for cross-language compatibility, but requires learning a new, separate definition language just to use RPC.******
 


**What's specifically done in [[P4L1_RPC#SunRPC Overview|Sun RPC]] for these design points – you should easily understand this from your project?**
- SunRPC (1980s Sun UNIX) assumes the server host is known and uses a per-machine registry for service lookup.
- It’s language-agnostic, using XDR for both interface definitions and data encoding.
- Pointers are allowed; their referenced data gets serialized.
- The runtime supports configurable retries on timeout and strives to return meaningful error codes.

**What’s [[P4L1_RPC#Marshalling|marshalling]]/[[P4L1_RPC#Unmarshalling|unmarschaling]]?**
- Marshaling turns separate arguments (e.g., i, j) into one contiguous payload.
	- ![](https://assets.omscs.io/notes/938F6FC8-08F2-4280-B95B-75AFA74E3B4A.png)

- Unmarshalling reads the incoming buffer, uses the procedure descriptor to parse out each argument’s bytes, and rebuilds server-side variables (e.g., i, j) with those values.
	- ![](https://assets.omscs.io/notes/CA66D7D2-E21F-484A-A0C5-B87185CE6515.png)

**How does an RPC runtime serialize and deserialize complex variable size data structures?** **What’s specifically done in Sun RPC/XDR?**
- An RPC system will have to specially encode variable length arrays. In SunRPC, variable lengths arrays are represented in code as structs with two fields. The first field will contain an integer representing the length of the data. The second field will be a pointer to the data. In addition, strings will be encoded with length and data fields, although they will be represented in memory as plain null-terminated strings.


# P4L2 Distributed File System

[[P4L2_Distributed_File_Systems]]


**What are some of the design options in implementing a distributed service?**
- **Upload/download model**: client pulls entire file, edits locally, then pushes back—fast local edits but costly for small changes and removes server visibility.![](https://assets.omscs.io/notes/B02657E2-D79B-43AB-87F4-7B75275CA3CB.png)
- **True remote access**: file stays on server; every operation goes over the network—ensures server-controlled consistency but incurs per-op network cost and limits scalability.![](https://assets.omscs.io/notes/4358EB0F-C95F-4106-A8ED-74E92E275263.png)

**What are the tradeoffs associated with a [[P4L2_Distributed_File_Systems#Stateless vs Stateful File Server|stateless vs. stateful]] design?** 
- **Stateless servers** keep zero per-client or per-file context, so every request must be self-contained. They fit extreme models (pure upload/download or pure remote access) but can’t support caching or consistency control. Upsides: no state-management overhead and easy recovery—just reboot and resume service.
- **Stateful servers** track clients, open files, access patterns, caches, and locks. This enables coordinated caching, strong consistency, relative reads, etc. Downsides: extra CPU/memory for state, need to checkpoint/recover state on failure, and clients must participate in consistency protocols.

- We can choose to have a stateful server or a stateless server. Stateless servers are easy; they just serve file content. That being said, stateless servers can't drive coherence mechanisms, so clients can't cache against stateless servers. Stateful servers are more complex, but also allow for more performant, expressive behavior.

**What are the tradeoffs (benefits and costs) associated with using techniques such as caching, replication, partitioning, in the implementation of a distributed service (think distributed file service).****
- **Caching**
	- DFS clients cache file blocks locally, but shared blocks need coherence: when one client updates, others must be told.
	- Coherence can be triggered on access, periodically, or at open time, and may be enforced by clients or the server.

	- Caching file content allows clients to not have to go to the server for every operation. This can improve performance for repeated reads.
	- The downside of this approach is that coherence mechanisms must now be introduced into the server.
	- If one client is reading from cache, and another client writes data  corresponding to the chunk that the first client is caching, we need some way to allow that client to fetch the updates.
- **Replication**
	- Replication copies the entire filesystem onto each server, enabling load balancing, high availability, and fault tolerance 
	- but write operations must be coordinated across replicas (synchronously or via async propagation with reconciliation, e.g., voting).

	-  If a node goes down, any other node can be used. This makes replicated systems highly available and fault tolerant. 
	- Unfortunately, these systems are harder to scale, since every node needs to hold all of the data.
	
- **Partitioning**
	- Partitioning assigns subsets of files to different servers, scaling capacity linearly and keeping writes local, 
	- but a partition failure makes its data unavailable and skewed access patterns can create hotspots; many systems combine both (partition + replicate each shard).

**The [[P4L2_Distributed_File_Systems#Sprite Distributed File System|Sprite]] caching paper motivates its design based on empirical data about how users access and share files. Do you understand how the empirical data translated in specific design decisions?**
- Sprite’s workload study showed ~33% writes, so caching (beyond simple write-through) could benefit the remaining 66% of operations (reads).
- Most files are short-lived: 75% stay open <0.5 s, 90% <10 s; many new files are deleted quickly (20–30% within 30 s, 50% within 5 min).
- File sharing was rare, so the designers concluded write-back-on-close (session semantics) wasn’t worth the overhead, though concurrency had to be supported.

- Sprite caches file data client-side but uses write-back:
	- every 30 s it flushes blocks that haven’t been modified in the last 30 s, banking on the observation that freshly written data often gets deleted quickly.
- When another client opens a file being written elsewhere, the server forces the writer to flush outstanding dirty blocks.
- All open requests go to the server, so directory contents aren’t cached locally.
- Concurrent writes disable caching entirely; updates serialize at the server, which is acceptable because such sharing is rare.

**Do you understand what type of data structures were needed at the servers’ and at the clients’ side to support the operation of the Sprite system (i.e., what kind of information did they need to keep track of, what kids of fields did they need to include for their [[P4L2_Distributed_File_Systems#File Access Operations in Sprite|per-file/per-client/per-server data structures]]).**
-  Clients track per-file :
	- cache status, 
	- cached blocks, 
	- dirty timers,
	- version numbers
- Server track per file
	- current readers, 
	- writer,
	- file version.


# P4L3 Distributed Shared Memory

[[P4L3_Distributed_Shared_Memory]]

**When sharing state, what are the tradeoffs associated with the sharing granularity?**
 - Cache-line coherence is too costly over the network for DSM, so sharing is managed at coarser granularity (variables, pages, or objects).
- Variables are still too fine-grained
- page granularity (good for OS-Level)
- objects (good for Language runtime)
- Coarser granularity risks false sharing (unrelated data on the same page/object triggers unnecessary coherence traffic).

  **For distributed state management systems (think distributed shared memory) what are the basic mechanisms needed to maintain consistence – e.g., do you why is it useful to use ‘home nodes’,**
  - What is a home node
	  - A home node is the “manager” for a piece of shared memory (like a page) in a distributed shared memory system.
- Why is it useful to have a home node:
	- Single authority: It knows where the page lives, who currently owns it, and who is caching it.
	- Consistency control: It coordinates reads/writes, locks, and invalidations/updates so all nodes see a coherent view.
	- Fast lookup: With a global index (replicated on all nodes), any node can quickly find the home node for a page.
	- Scalable metadata: Detailed state (who’s caching, lock status, dirty/clean) is kept only at the home node, reducing system-wide storage and sync overhead
  **why do we differentiate between a global index structure to find the home nodes and local index structures used by the home nodes to track information about the portion of the state they are responsible for.**
- What is the global index structure:
    - A replicated map on every node that takes a page/object ID and returns its home/manager node ID (who to contact).
- What is the local index structure:
    - Per-page metadata stored only on the home/manager node for its pages (sharers, owner, lock/dirty/access status, caching flags).
- Why differentiate them:
    - Replication of the global map enables fast lookup from any node; partitioning detailed metadata to home nodes reduces storage/sync overhead and scales management.

**Do you have some ideas how would you go about implementing a distributed shared memory system?**
- Goal: Intervene only on accesses to distributed shared memory, not local memory.
- Mechanism: Use the MMU to trigger faults that trap into the OS.
- Read path (remote page):
    - Invalid local access to remote address causes a fault.
    - OS detects the address is remote.
    - OS looks up the home/owner node via the global address → home/owner map.
    - OS requests data from the home node using IPC (message queue, socket, etc.).
    - On receipt, OS caches data (CPU/memory) and returns it to the requesting process.
- Write path (shared state):
    - Write-protect virtual addresses for shared memory.
    - Writes cause a fault, trapping into the kernel.
    - Kernel determines shared access and:
        - If not home: sends update request to the home node.
        - If home: applies update and broadcasts coherence messages to nodes holding replicas.
- Coherence tracking:
    - Home node maintains per-page metadata listing which nodes have cached/used the page.
    - Requests include the requester’s node ID to update sharer lists and route coherence.


**What are the different guarantees that change in the different models we mentioned – strict, sequential, causal, weak…**
- **Strict consistency:** Idealized, impossible in practice. Every write is seen by all nodes **instantly** and in real-time order.
- **Sequential consistency:** All nodes see the same single interleaving of operations; program **order is preserved** per process. Writes may be delayed, but everyone agrees on the order.
- **Causal consistency:** Only orders writes with potential **cause-effect links** (a write that follows a read). Concurrent, unrelated writes can be seen in different orders; program order is still preserved.
- **Weak/relaxed consistency:** Visibility is guaranteed only at **synchronization** points. Processes use ops like sync/acquire/release to publish or observe updates; between syncs, reads/writes may be stale.

# P4L4 Datacenter Technologies

[[P4L4 Datacenter Technologies]]

**When managing large-scale distributed systems and services, what are the pros and cons with adopting a homogeneous vs. a heterogeneous design?**
	-  **Homogeneous** cluster: each node can handle any request end-to-end.
		- Pros :
			- frontend load-balancing component can be simple
			- Send requests to each node in round robin manner
		- Cons:
			- No caching, front end does not  track which node hold the warm data
    - **Heterogeneous** cluster: nodes specialize in certain pipeline stages or request types.
    - 		- Pros :
			- Can leverage caching
			- 
		- Cons:
			- Frontend become more complex
			- Harder to scale
				- It's not longer enough to add or remove nodes in response to fluctuating traffic. In heterogenous systems, you'll need to understand which components of the system need to be scaled up or scaled down.

**Do you understand the history and motivation behind cloud computing, and basic models of cloud offerings?**
- In 2006, Amazon noticed its hardware was underutilized outside the holiday peak and launched AWS to rent out excess compute capacity to third parties, kickstarting modern cloud computing.

 - software as a service (SaaS): Provider owns app, data, infrastructure, and hardware; users consume the app (e.g., Gmail).
- platform as a service (PaaS): Provider owns infrastructure and hardware; developers deploy their apps and data on the provided runtime (e.g., Google App Engine).
- infrastructure as a service (IaaS): Provider owns hardware; developers bring apps/data and configure OS/middleware themselves (e.g., Amazon EC2).
- ![](https://assets.omscs.io/notes/81DBAA16-6AF2-4CEB-9840-665F49F5001B.png)


**Do you understand some of the enabling technologies that make cloud offerings broadly useful?**
- Virtualization makes resources fungible so they can be repurposed across customers.
- Provisioning and scheduling stacks (e.g., Mesos, YARN) spin up resources quickly and consistently.
- Big-data engines like Hadoop MapReduce and Spark handle large-scale processing/storage needs.
- Storage layer relies on distributed, often append-only filesystems, plus NoSQL databases and in-memory caches for scalable data access.
- Isolation tech enforces per-tenant resource slices for security/performance.
- Monitoring/telemetry tools (Flume, CloudWatch, Log Insight) give providers and customers visibility into infrastructure and application behavior.

**Do you understand what about the cloud scales make it practical?**
- Provider:
	- Law of large numbers: With many clients, aggregate demand is steady, so providers can size a fixed pool of hardware without chasing individual peaks.
	- Economies of scale: Hardware costs are amortized across many tenants, making cloud services cost-effective.
- Client
	- Elastic resources: Clouds let clients scale up/down quickly (e.g., Animoto scaled 100x in a week).
	- Elastic costs: Pay more only when using more; spending tracks traffic and revenue.
	- Benefit vs. on-prem: Avoid underutilization (paying too much) and maxed-out capacity (lost opportunity).
	- Data centers enable seamless scaling that traditional setups can’t provide.
**Do you understand what about the cloud scales make failures unavoidable?**
- Complex systems fail more often because total reliability multiplies component reliabilities.
- If each component succeeds with 95%, a single link → 95% success.
- With 30 such links in a chain: overall success = 0.95^30 ≈ 21%.
- More components → higher chance some part fails → lower end-to-end reliability.


## I/O Devices

The execution of applications doesn't rely on only the CPU and memory, but other hardware components as well. Some of these components are specifically tied to receiving inputs or directing outputs, and these are referred to as **I/O devices**.

Examples of I/O devices include
- keyboards
- microphones
- displays
- speakers
- mice
- network interface cards


## I/O Device Features

- Device landscape is highly diverse in form, function, and interfaces.![](https://assets.omscs.io/notes/95C30A1B-9A9B-4CFD-A7CA-967EF1BD431F.png)
- Host CPU talks to devices through control registers: command (issue actions), data (transfer payloads), and status (read device state).
- Internally, devices bundle their own microcontroller, memory, and specialized logic (e.g., analog/digital converters, network PHYs).![](https://assets.omscs.io/notes/DA5651B0-8568-45A2-A26E-B5DE1F699F9B.png)


## CPU Device Interconnect

- Device controllers sit between peripherals and the CPU complex, plugging into system interconnects such as PCI.![](https://assets.omscs.io/notes/38B48ADB-1EC5-4771-A87B-92D9102ADA69.png)
- Modern systems favor PCI Express for its higher bandwidth, lower latency, and greater device capacity, yet often keep PCI-X slots for backward compatibility.
- Other buses (e.g., SCSI for disks, legacy peripheral buses for keyboards) may coexist; the controller dictates which interconnect a device uses.
- Bridging controllers translate between differing bus standards so heterogeneous devices can share the system.

## Device Drivers

- OSes rely on device-specific drivers—software modules that handle access, management, and control for each hardware type.![](https://assets.omscs.io/notes/BE7972F7-6118-4819-95D2-EFE8A38524FB.png)
- Drivers translate high-level requests, manage errors/notifications, and encode all device-specific configuration.
- Hardware vendors supply drivers for the operating systems they target (e.g., downloading a printer driver from HP).
- OSes expose standardized driver frameworks so new devices integrate without tying the kernel to any fixed hardware set.


## Types of Devices


- Devices fall into block (random-access blocks, e.g., disks), character (serial get/put, e.g., keyboards), and network (streams of variable-size chunks) classes.
- Each class exposes standardized ops—read/write block for block devices, get/put char for character devices, etc.
- OS models devices as special files, letting higher layers reuse file APIs while drivers handle device-specific work.
- On Unix-like systems these device files live under /dev, managed by tmpfs/devfs.


## CPU/Device Interaction

- Devices often expose control registers via memory-mapped I/O: the CPU reads/writes specific physical addresses, with PCI Base Address Registers (BARs) reserving that space during boot.
- x86 also supports an I/O port model, using explicit in/out instructions that name the target port and payload.
- Communication paths: devices can raise interrupts (immediate notification but handler overhead and cache disruption) or be polled by the CPU (flexible scheduling, but can delay response and burn cycles if overused).


## Device Access PIO (Programmed IO)

- Programmed I/O (PIO) lets the CPU drive devices directly via their command and data registers—no extra hardware beyond the bus (e.g., PCI).
- Example: sending a 1500 B packet through a NIC with 8 B data registers takes 1 write to the command register plus 188 data-register writes, totaling 189 CPU operations.


## Device Access DMA (Direct Memory Access)

- DMA-capable devices add a DMA controller so the CPU issues commands but delegates data moves to hardware.
- Example NIC send: 1 command-register write plus 1 DMA setup (address + length) vs 188 data-register writes in PIO.
- DMA setup is costly, so PIO may win for tiny transfers; DMA excels on larger ones.
- Buffers used for DMA must stay resident in physical memory (“pinned”), since the controller can’t follow swapped-out pages.

## Typical Device Access

- User code issues a system call for device work; the kernel’s device stack may prep the data (e.g., build a TCP/IP packet).
- The kernel calls the device driver, which programs the hardware (PIO or DMA) without overwriting pending data.
- The device executes the request (e.g., NIC transmits), and completion events flow back driver → kernel → user.
- ![](https://assets.omscs.io/notes/1C2567BE-17C9-43AE-98C9-A864103CCE98.png)


## OS Bypass

- Operating-system bypass maps device registers/buffers directly into user space, letting processes drive hardware without kernel calls after setup.
- User-level drivers (vendor libraries) replace kernel drivers for day-to-day control while the OS keeps coarse control (e.g., enable/disable, permissions).
- Devices need enough register space to share per-process views and still expose management registers to the OS.
- Without the kernel mediating, the device must demultiplex inbound data to the correct user process itself.


## Sync vs. Async Access

- Synchronous I/O blocks the caller: the kernel enqueues the thread on the device’s wait queue until the response arrives.
- Asynchronous I/O lets the thread continue immediately; completion is detected later via polling or kernel notification.![](https://assets.omscs.io/notes/791986F3-95FD-400A-B5E2-0B0A27992CC8.png)
- Contrast with “fake” async (e.g., Flash using helper threads)—this is genuine OS-supported async behavior.

## Block Device Stack

- Applications see files—logical units backed by physical block storage—which they access via the standardized POSIX read/write API.
- The filesystem translates those calls into block operations: locating data, enforcing permissions, determining which portions to read/write.
- Block devices expose varied, protocol-specific interfaces, so the OS inserts a generic block layer that presents a uniform block API to filesystems while drivers handle device-specific details.


## Virtual File System

- Linux’s Virtual Filesystem (VFS) layer abstracts over multiple underlying filesystems—local, remote, or device-specific—so applications still use the same POSIX API.![](https://assets.omscs.io/notes/DE9A5B7E-FDE3-4C19-9D65-7BFFDC72A9B0.png)
- VFS defines a common set of filesystem abstractions every concrete filesystem must implement, hiding implementation differences and letting diverse storage backends appear as one coherent tree.


## Virtual File system Abstractions

- Files appear to applications as integer file descriptors that support ops like read, write, and close.
- Each file has an on-disk inode recording its data-block locations plus metadata (size, permissions, etc.), enabling scattered block placement.
- Path traversal uses in-memory dentries (one per path component) cached for reuse, speeding repeated directory lookups.
- The superblock describes a filesystem’s layout on disk, mapping where inodes and data blocks live and holding FS-specific metadata.


## VFS on Disk

- VFS structures are managed in software; dentries live only in memory, while files and inode metadata persist as blocks on disk.
- The superblock stores the allocation map, identifying which blocks contain data, which hold inodes, and which remain free for future use.


## ext2: Second Extended File systems

- Ext2 partitions begin with an unused boot block, then divide the remainder into block groups unrelated to the disk’s physical structure.![](https://assets.omscs.io/notes/E63DEDAE-89A8-4A35-B994-24482EC5FE27.png)
- Each group starts with a superblock (counts inodes/blocks, marks free-block start) and a group descriptor (bitmaps, free inode count, directory count).
- Bitmaps flag free vs allocated blocks and inodes for fast allocation.
- Inodes (128 B each) are numbered sequentially; each describes one file with ownership metadata and pointers to its data blocks.
- The rest of the group consists of the actual data blocks.

## inodes

- Inodes uniquely identify files and list all data blocks plus metadata (ownership, size, timestamps).
- ![](https://assets.omscs.io/notes/4FC8ECB4-6FB8-40AC-A066-FAB96A8CB28E.png)
- Adding capacity is simple: allocate a new block and append its pointer to the inode.
- Direct block pointers make both sequential and random access straightforward.
- Limitation: an inode’s fixed pointer array caps file size (e.g., 128 B inode with 4 B pointers addresses only 32 × 1 KB = 32 KB).


## inodes with Indirect Pointers

- Indirect pointers extend inode reach without enlarging the inode: direct entries map directly to 1 KB blocks; indirect entries point to blocks full of data pointers.![](https://assets.omscs.io/notes/04E687DF-CB15-4C86-A420-625B1288AEF9.png)
- Single indirect pointer covers 256 × 1 KB = 256 KB; double indirect covers 256 × 256 × 1 KB ≈ 64 MB.
- Trade-off: larger files become addressable, but accessing data through multi-level pointers adds extra disk reads (inode + pointer blocks + data block).


## Disk Access Optimizations

- Buffer caches hold recently read/written data in RAM, flushing to disk periodically (or via fsync) to cut down on disk trips.
- I/O schedulers reorder requests to minimize head movement, favoring sequential over random access (e.g., handle block 17 before 25 when both are queued).
- Prefetching grabs adjacent blocks proactively, trading extra bandwidth for lower latency via higher cache hit rates.
- Journaling logs intended updates before applying them, ensuring crash recovery while deferring random writes to more convenient times.



## Summary

- I/O systems encompass diverse peripherals linked through controllers and buses (PCIe, PCI-X, SCSI) that expose control/data/status registers for the CPU to drive.
- Device drivers encapsulate hardware-specific logic, presenting standardized interfaces so the OS can treat devices as files (/dev) across block, character, and network classes.
- Communication uses memory-mapped I/O or I/O ports, with programmed I/O suited to small transfers and DMA offloading large data moves while requiring pinned buffers.
- Kernels orchestrate device operations via system calls, drivers, and completion paths; optimizations include OS bypass, synchronous vs asynchronous APIs, buffer caches, schedulers, prefetching, and journaling.
- Linux’s VFS unifies filesystem access, mapping POSIX calls to concrete backends like ext2, whose block groups, inodes (with direct/indirect pointers), and superblocks organize disk data efficiently despite fixed inode pointer limits.
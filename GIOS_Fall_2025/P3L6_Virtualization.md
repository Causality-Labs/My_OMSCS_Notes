


## Defining Virtualization
- Virtualization lets multiple operating systems share one physical machine while each “guest” believes it owns the hardware.
- ![](https://assets.omscs.io/notes/A96F5139-EFB1-4038-A5E4-9128C63CF5F8.png)
- A virtual machine (VM) bundles an OS, its apps, and virtual hardware; guests/domains run side by side.
- A virtualization layer—hypervisor or virtual machine monitor—allocates real resources and enforces isolation among VMs.
- VMs strive to be efficient, isolated replicas of real machines.
- The VMM/hypervisor must ensure fidelity: exposed virtual hardware mirrors the physical platform’s architecture and device set.
- It must deliver near-native performance when a VM receives equivalent resources.
- It governs all resource access, enforcing safety and isolation between guests.


## Benefits of Virtualization

- Consolidation: many VMs share one host, cutting hardware count, admin effort, and power while sustaining workload capacity.
- Migration: decoupled OS/apps make VM setup, teardown, cloning, and relocation straightforward.
- Availability & reliability: failing hosts can be bypassed by spinning the VM on another machine; faults stay contained inside individual VMs.
- Research & legacy support: easy to boot experimental OS builds or keep old OSes running without dedicated hardware.

## Virtualization Models: Bare-Metal

- Bare-metal (type 1) hypervisors sit directly on hardware, managing all resources for hosted VMs.![](https://assets.omscs.io/notes/A320790E-26BD-4370-8B87-47E08D04FA9C.png)
- Device support is tricky: hypervisors need drivers for every device. Solution: run a privileged service VM with full hardware access to host drivers and management utilities.
- Xen uses dom0 as the privileged domain (running all drivers) and domU for guests, with Xen itself as the hypervisor.
- VMware ESX similarly relies on hypervisor-level drivers; targeting servers (fewer devices) lets VMware demand vendor support and expose remote/open APIs for management.


## Virtualization Models: Hosted

- Hosted (type 2) virtualization runs on top of a full host OS; the VMM module rides inside that OS and presents virtual hardware to guests while calling into existing drivers/services.![](https://assets.omscs.io/notes/2BE5A89A-9473-420B-B659-D5F010B2BB8D.png)
- Hosts can run native apps alongside guest VMs; less VMM code is needed since hardware management stays with the host OS.
- Linux’s KVM exemplifies this model: the kernel handles hardware, KVM provides the VMM, and QEMU emulates devices or intercepts critical instructions (e.g., I/O).
- Because KVM taps directly into Linux’s evolving codebase, it rapidly inherits new features and fixes.



- **Bare-metal (type 1)**: the hypervisor sits directly on the hardware, owns all devices, and carves out virtual machines. It may delegate driver work to a privileged service VM, but there’s no general-purpose host OS beneath the guests.
    
- **Hosted (type 2)**: a regular host operating system runs first, managing hardware and drivers. The hypervisor lives inside that OS as a module/app, so guests share the host’s services while native apps can run alongside them.


## Hardware Protection Levels

- x86 offers four privilege rings: ring 0 is highest (kernel/hypervisor), ring 3 least (apps).
- In virtualization, the hypervisor claims ring 0; guest OSes run at a lower privilege (e.g., ring 1) while their apps stay in ring 3.
- Newer CPUs add root vs non-root modes: hypervisor runs in root ring 0; guest OS/apps run in non-root rings.
- Guest privileged ops trigger VMExits (switch to root mode) so the hypervisor can emulate/mediate, then VMEntry resumes the guest.


## Processor Virtualization

- Guest code runs natively on hardware—no hypervisor involvement unless it oversteps allocated resources, so performance stays near bare metal.
- Privileged instructions trigger a trap to the hypervisor, which decides to deny (e.g., terminate the VM) or emulate the expected hardware behavior.
- This “trap-and-emulate” intervention must remain transparent to the guest OS.


## x86 Virtualization in the past

- Pre-2005 x86 offered only rings; hypervisors took ring 0, pushing guest OSes to ring 1.
- Seventeen privileged instructions (e.g., POPF/PUSHF to toggle interrupts) failed silently outside ring 0—no trap, no hypervisor visibility.
- Guests believed the operations succeeded, leaving the hypervisor unable to emulate the intended behavior.

## Binary Translation

- Binary translation rewrites guest code on the fly to avoid the 17 problematic x86 instructions while keeping the guest OS unmodified (“full virtualization”).
- The hypervisor inspects upcoming code blocks: if safe, they run natively; if a forbidden instruction appears, it’s replaced with an emulation sequence.
- Translation incurs overhead, mitigated by caching translated blocks and focusing on guest kernel code paths—an approach pioneered by VMware.


## Paravirtualization

- Paravirtualization modifies the guest OS so it knows it’s running atop a hypervisor and avoids privileged ops that would fail.
- Instead, it issues explicit hypercalls (analogous to syscalls) to request services; the hypervisor handles the request and returns control.
- Xen popularized this performance-focused approach.



## Memory Virtualization: Full Virtualizaation

- Full virtualization lets the guest OS believe it owns a contiguous “physical” address space starting at zero.
- Three address notions: guest apps use virtual addresses → guest OS maps them to guest-physical addresses → hypervisor maps guest-physical to host machine addresses.
- Without help, every reference would need a two-stage translation, imposing heavy overhead.
- Solution: the hypervisor maintains a shadow page table mapping guest virtual directly to machine addresses, keeping it synchronized and write-protecting guest tables so updates trap into the hypervisor.


## Memory Virtualization: Paravirtualization

- Paravirtual guests know they’re virtualized, so they can forgo the illusion of contiguous physical memory starting at zero.
- They register their own page tables with the hypervisor, eliminating the need for guest vs shadow tables.
- Guests lack write access to those tables; updates trap into the hypervisor, but paravirtualization lets them batch changes into a single hypercall.
- Modern hardware reduces many of these memory-virtualization overheads for both full and paravirtual setups.



## Device Virtualization

- CPU/memory virtualization is simpler because ISAs (e.g., x86) are standardized—hardware differences hide beneath that level.
- Device virtualization is harder: hardware varies widely and interfaces lack consistent semantics.
- Hypervisors therefore rely on three main models to virtualize devices (introduced next).


## Passthrough Model

- Passthrough (VMM-bypass) maps a device’s registers directly into a VM, giving that guest exclusive control.
- ![](https://assets.omscs.io/notes/DB8B54EB-D3C3-4EB0-A8F4-71163B1426E0.png)
- Sharing becomes awkward: the hypervisor must reassign the device between VMs, so concurrent access is impractical.
- Guests must see exactly the device model they expect, eliminating virtualization’s usual hardware abstraction.
- Binding a device to one VM complicates live migration because device-resident state has to move with the guest.


## Hypervisor Direct Model

- Hypervisor-direct virtualization intercepts every guest device access; the hypervisor emulates a generic device and forwards the request through its own I/O stack to the real driver.![](https://assets.omscs.io/notes/4F00CAC7-1C7B-4065-A509-F7B57DD7FBE3.png)
- Benefits: guests stay decoupled from physical hardware, making migration easy and allowing device sharing under hypervisor control.
- Drawbacks: emulation introduces latency, and the hypervisor must implement/maintain drivers for all supported devices.

## Split Device Driver Model

- Split drivers divide device control: a paravirtualized front-end runs in each guest VM, packaging requests for a back-end driver in the service VM/host.![](https://assets.omscs.io/notes/BECB3588-AFBF-42F2-9A1C-22916449B1B4.png)
- Back-end driver can be the native OS driver; only the front-end needs custom code, limiting the model to paravirtual guests.
- Benefits: no device emulation overhead and centralized back-end control simplifies device sharing and management.


## Hardware Virtualization

- AMD Pacifica and Intel Vanderpool (circa 2005) made x86 virtualization-friendly by forcing the 17 formerly silent privileged instructions to trap.
- Introduced root/non-root modes (host vs guest) and VM control blocks so hardware can manage vCPU state and trap policies.
- Added VM-tagged memory structures—extended page tables and tagged TLBs—so VM context switches avoid flushes.
- Enhanced I/O virtualization with features like multiqueue devices and precise interrupt routing, plus broader security/management support.
- New x86 instructions accompanied these capabilities, including ops to enter the new protection modes.

## x86 VT Revolution

![](https://assets.omscs.io/notes/57D4D274-194C-4906-8331-48AC6C00F7FE.png)



## Summary

- Virtualization inserts a hypervisor layer so multiple isolated VMs—each bundling its own OS and apps—share the same hardware while believing they have dedicated resources.
- Two main models appear: bare‑metal hypervisors that run directly on hardware (often delegating drivers to a privileged service VM) and hosted hypervisors that sit atop a general-purpose OS and reuse its device support.
- To keep performance and fidelity, hypervisors rely on trap-and-emulate or binary translation (for unmodified guests) and paravirtualization (guest-aware hypercalls), plus shadow or shared page tables to handle multi-level memory mappings efficiently.
- Device virtualization ranges from direct passthrough to hypervisor emulation or split drivers, and modern x86 hardware (Intel VT/AMD-V) adds root modes, nested paging, and improved I/O to make these techniques practical and high performance.
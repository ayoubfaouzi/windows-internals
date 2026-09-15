# Chapter 9: Virtualization Technologies

## The Windows hypervisor

- Hyper-V, also called the **Windows hypervisor**, is a **type-1 / native / bare-metal hypervisor**. This means it runs directly on the physical hardware rather than running as a normal application on top of an existing OS.
- However, Windows Hyper-V is slightly special compared to the classic mental model of a standalone bare-metal hypervisor:
  - The hypervisor runs below Windows.
  - The main Windows installation becomes the **root OS** or **root partition**.
  - Guest virtual machines run beside it as separate partitions.
  - The root OS is aware that the hypervisor exists and communicates with it.
  - VM management is integrated into Windows through normal mechanisms such as **WMI**, services, and other OS management APIs.
- This differs from a **type-2 / hosted hypervisor**, where the hypervisor runs like an application on top of a normal host OS. Examples would be classic desktop virtualization products where the host OS remains the true owner of the hardware.
- The root OS contains **enlightenments**. An **enlightenment** is a special optimization in the Windows kernel or in device drivers that detects that the system is running under a hypervisor and changes behavior accordingly.
- Instead of pretending the machine is fully native, enlightened (**paravritualized**) components cooperate with the hypervisor to do things more efficiently.

<p align="center"><img src="./assets/hyperv-architecture.png" width="600px" height="auto"></p>

- At the bottom of the virtualization architecture is the **hypervisor**. It is launched very early during boot and exposes services to the rest of the virtualization stack through the **hypercall interface**.
- The hypervisor startup begins during the Windows boot process. The **Windows Loader** decides whether to start:
  - the Hyper-V hypervisor
  - the Secure Kernel, when VBS-related features are enabled
- If the hypervisor is started, Windows uses **Hvloader.dll** to detect the hardware platform and load the correct hypervisor binary.
- Because virtualization extensions differ between CPU vendors and architectures, Windows ships different hypervisor binaries:

| Platform  | Hypervisor binary |
| --------- | ----------------- |
| Intel x64 | `Hvix64.exe`      |
| AMD x64   | `Hvax64.exe`      |
| ARM64     | `Hvaa64.exe`      |

### Partitions, processes, and threads

- The central isolation abstraction in Hyper-V is the **partition**.
- A partition represents an OS instance under the Windows hypervisor. Hyper-V does not mainly use the classic terms **host** and **guest**. Instead, it uses:

| Traditional term | Hyper-V term        |
| ---------------- | ------------------- |
| Host             | **Root partition**  |
| Guest VM         | **Child partition** |

- A partition contains, at the hypervisor level:
  - Assigned physical memory
  - One or more **virtual processors** / **VPs**
  - Local virtual APICs
  - Virtual timers
- Other things commonly associated with a VM, such as virtual motherboard, virtual devices, synthetic peripherals are not really **hypervisor concepts**. They belong mostly to the **virtualization stack** running in the root partition.
- A Hyper-V system always has at least one partition: the **root partition**.
- The root partition is where the main Windows OS runs. It is special because it provides:
  - The virtualization stack
  - Hardware device drivers
  - VM management infrastructure
  - Control over child partitions
  - I/O handling on behalf of guests
  - Management services/APIs
- Only the root partition has **full control** over the machine from the Windows virtualization model’s point of view.
- Even though the hypervisor itself is loaded very early by the Windows Loader, before the root OS fully exists, the hypervisor stays small. The root Windows OS later provides the larger virtualization infrastructure.
- Each guest OS runs inside a **child partition**.
  - A child partition can include tools/components that improve performance or manageability, such as Hyper-V integration components or **enlightened drivers**.
  - Child partitions usually do **not** access physical hardware directly. Their I/O operations are typically intercepted or routed through the **root partition**.
  - There are exceptions, such as certain **passthrough** or **direct device assignment** scenarios, but the general design is that the root partition owns the real hardware.
- Partitions are organized **hierarchically**.
  - The **root partition** controls the child partitions. For certain events occurring inside a child partition, the root receives notifications called **intercepts**.
  - These intercepts allow the root/virtualization stack to handle events that require emulation, management, policy enforcement, or I/O forwarding.

<p align="center"><img src="./assets/root-partition-components.png" width="200px" height="auto"></p>

- The root partition’s own hardware accesses are mostly passed through by the hypervisor, meaning the root OS can usually talk **directly** to the hardware through normal **Windows drivers**. Child partitions are much more restricted.
- A major design goal of the Windows hypervisor is to keep it small and modular, closer to a **microkernel-like** design rather than a large monolithic hypervisor.

```text
Hypervisor: small low-level isolation and scheduling layer
Root partition: Windows drivers + virtualization stack + VM management
Child partitions: guest operating systems
```

### Child partitions

- A **child partition** is an OS instance running alongside the root/parent partition. It corresponds to what is usually called a **guest VM**, although Hyper-V terminology uses **child partition** instead.
- Unlike the parent/root partition, a child partition has a heavily restricted view of the system.
- The **root partition** has broad access to: APIC, I/O ports, its own physical memory and real hardware through Windows drivers.
- But even the root partition does **not** access:
  - hypervisor physical memory
  - Secure Kernel physical memory
- A **child partition**, by contrast, is restricted to its own **Guest Physical Address (GPA) space**.
- The GPA space is the child’s view of “physical memory,” but it is not real machine physical memory directly. It is managed and translated by the hypervisor.

<p align="center"><img src="./assets/child-partition-components.png" width="200px" height="auto"></p>

```text
Child partition
    → talks to virtual/synthetic device
    → request goes to root virtualization stack
    → root uses real Windows hardware driver
    → physical device
```

- So child partitions are **consumers** of **virtualization services**, not providers of them.
- A child partition has **fewer** virtualization-related components than the parent partition. This is because it does not run the virtualization stack. It only needs enough support to communicate with the stack running in the root partition.
- In a Windows child partition, these components are usually **integration/enlightenment** components that improve:
  - Pperformance
  - Device access
  - Time synchronization
  - Shutdown/save/restore behavior
  - Synthetic device communication
  - Management from the root partition

### Processes and threads

The Windows hypervisor represents each VM as a **partition** (`VM_PARTITION`), mainly composed of guest physical memory and one or more **virtual processors** (`VM_VP`), where each VP is treated as a schedulable entity that the hypervisor scheduler dispatches onto physical CPUs.

For each VP, the hypervisor creates a **hypervisor thread** (`TH_THREAD`), which acts as the schedulable execution context for that VP and contains its stack, scheduling metadata, dispatch-loop entry point, pointer to the associated `VM_VP`, and pointer to the owning hypervisor process.

A **hypervisor process** (`TH_PROCESS`) represents the partition as a process-like container for its address space and execution state; it owns the list of `TH_THREAD` objects, scheduling information such as physical CPU affinity, and pointers to partition memory-management structures such as the memory compartment, reserved pages, and page-directory root.

👉 The key model is: **partition = VM container**, **hypervisor process = address-space/scheduling container for that partition**, **VP = virtual CPU**, and **hypervisor thread = schedulable unit backing a VP**.

When the hypervisor creates a new partition:

1. It builds a `VM_PARTITION`.
2. It creates a `TH_PROCESS` for that partition.
3. It creates one or more `VM_VP` objects.
4. For each `VM_VP`, it creates a corresponding `TH_THREAD`.
5. Those threads become schedulable units for the hypervisor scheduler.

### Enlightenments

- Enlightenments are **hypervisor-aware** code paths inside the Windows kernel and drivers that detect execution inside a child partition and replace expensive virtualized hardware behavior with more efficient cooperation with the hypervisor, usually through **hypercalls**.
- A typical example is a **long spin-wait loop**: instead of wasting a physical CPU while a VP waits, Windows can notify the hypervisor, allowing it to track the wait condition and potentially **schedule another VP** on that physical processor until the original VP can make progress.
- Other enlightenments optimize operations such as **interrupt-state transitions** and **APIC access**, where Windows coordinates directly with the hypervisor instead of triggering real **APIC accesses** that would then need to be **trapped** and **virtualized**.
- The important memory-management example is **TLB flushing**: on native multiprocessor Windows, flushing stale TLB entries often requires sending **IPIs** to other processors, but in a VM this would be inefficient because physical CPUs may currently be running **VPs from unrelated partitions**, so enlightened Windows issues a hypercall asking the hypervisor to flush only the **relevant TLB state** for that child partition.

### Partition’s privileges, properties, and version features

- When a partition is first created, usually by **VID.sys** (Virtualization Infrastructure Driver), it initially has **no VPs**, and this is the only window where VID/root can adjust the partition’s **privileges**; once even one VP starts executing, the hypervisor refuses later privilege changes 🤷‍♀️.
- A partition privilege defines what the enlightened OS inside that partition is allowed to do through **hypercalls** or **synthetic MSRs**, and the default set depends on the partition type: **both root and child partitions** get basic self-management privileges such as ➡️accessing runtime/reference time, SynIC timers/registers, virtual APIC assist page, hypercall MSRs/code page, VP idle/index/TSC state, VSM/per-VTL synthetic registers, AP startup, and fast hypercall support.
- The **root partition** gets the powerful management privileges: ➡️creating and referencing child partitions, depositing/withdrawing memory from a partition compartment, creating and managing connection ports, posting messages/signaling events, mapping hypervisor statistics pages, enabling/querying hypervisor debugging, scheduling child VPs, accessing child SynIC synthetic MSRs, and triggering enlightened system reset.
- The **child partition** has only **limited extra privileges**, mainly to generate an extended hypercall intercept into the root partition and to notify the root scheduler that an event has been signaled so the guest’s VP-backed thread can be prioritized or rescheduled.
- An **EXO partition** has no default privileges.
- Partition **properties** are different from privileges because they can be queried or changed at any time, and they cover runtime categories such as scheduler settings like ➡️ Cap/Weight/Reserve, suspend/resume time properties, hypervisor debugger configuration, virtual hardware resource properties such as TLB size or SGX support, and compatibility properties tied to the VM’s configured virtual hardware level.
- The VM’s **compatibility level**, stored in its configuration and passed by VID to the hypervisor, controls which virtual hardware features are exposed to the guest VP; for example, older compatibility levels before _Windows 10 RS1_ hide guest **PAT** support even if the physical CPU supports it, while newer compatibility levels allow the hypervisor to expose PAT registers to the guest.
- The **root partition** is special at boot because the hypervisor gives it the **highest compatibility level**, allowing the root Windows OS to use all hardware features supported by the physical platform.

## The hypervisor startup

After **HvLoader** loads the CPU-vendor-specific hypervisor image and builds the **hypervisor loader block**, it captures enough initial processor context for the hypervisor to start the first VP, switches into a newly created address space, and transfers execution to the hypervisor entry point, **`KiSystemStartup`**.

`KiSystemStartup` runs only on the **boot processor**, prepares the CPU for hypervisor execution, and initializes **`CPU_PLS`**, the hypervisor’s per-physical-processor structure, roughly analogous to NT’s **PRCB** and quickly addressable through the **GS segment**.

The real boot-processor initialization is then delegated to **`BmpInitBootProcessor`**, which queries platform-specific virtualization capabilities such as **EPT**, **VPID**, etc.., chooses the hypervisor scheduler, and initializes **nested-enlightenment** support for scenarios where Hyper-V itself runs as an **L1 hypervisor** under an **L0 hypervisor**.

Scheduler selection depends on system type: on Intel/AMD server systems the default is usually the **core scheduler**, while on client systems, including ARM64, the default is the **root scheduler**, though it can be overridden through the **`hypervisorschedulertype`** BCD option.

`BmpInitBootProcessor` then initializes the hypervisor’s major internal subsystems: the memory manager with its **PFN database** and **root compartment**, the hypervisor HAL, the process/thread subsystem according to the selected scheduler, the special **system process** and initial thread used for hypervisor-internal execution, the **Virtualization Abstraction Layer / VAL**, **SynIC**, **IOMMU**, and the **Address Manager**.

The **VAL** abstracts CPU virtualization differences across Intel, AMD, and ARM64, and contains the platform-specific code for features such as Intel unrestricted guest mode, EPT, SGX, MBEC, and equivalent virtualization mechanisms on other architectures.

The **Address Manager / AM** owns the mapping between a partition’s **guest physical address space / GPA**, called an **address domain**, and real **system physical memory**; older Hyper-V could use shadow page tables, but since Windows 8.1 it relies on hardware SLAT mechanisms such as Intel **EPT**, AMD **NPT**, and ARM64 stage-2 translation.

After these subsystems are initialized, the hypervisor completes the boot processor’s `CPU_PLS` by allocating the hardware-dependent virtualization control structures, such as **VMCS** on Intel or **VMCB** on AMD, enables virtualization with the first **VMXON**-equivalent operation, and finally initializes the per-processor interrupt-mapping structures.

### The creation of the root partition and the boot virtual processor

After the hypervisor is initialized, its first major job is to create the **root partition** and its first virtual processor, the **BSP VP**, which is the VP used to resume the Windows boot process under Hyper-V control.

Root partition creation follows roughly the same layered process as child partition creation: the **VM layer** configures maximum VTL **support**, assigns **partition privileges** according to type, and enables features according to the partition **compatibility** level, with the root partition receiving the maximum feature set; the **VP layer** initializes the virtualized CPUID state shared by all VPs in the partition and creates the hypervisor process backing the partition; and the **Address Manager** builds the initial GPA→SPA mapping, using EPT on Intel or NPT on AMD, with the root partition using identity mapping so its guest physical memory corresponds directly to system physical memory.

Once **SynIC**, **IOMMU**, and **intercept shared pages** are configured, the hypervisor creates the root partition’s BSP VP, represented by a large **`VM_VP`** structure containing platform-dependent CPU state, register/debug/XSAVE/stack data, the VP private address space, a pointer to the backing hypervisor thread, a pointer to the physical CPU currently executing it, and an array of **`VM_VPLC`** structures tracking per-VTL state.

<p align="center"><img src="./assets/VM_VP-data-structure.png" width="300px" height="auto"></p>

`VmAllocateVp` allocates the memory for `VM_VP`, its platform-specific area, and one `VM_VPLC` per supported VTL from the partition compartment, copies the initial processor context captured earlier by **HvLoader**, creates and attaches to the VP private address space when address-space isolation is enabled, and then creates the VP’s backing thread.

A key detail is that VP construction then continues inside the newly **created backing thread**: the main hypervisor system thread waits, the scheduler selects the new thread, and that thread runs **`ObConstructVp`**, which attaches the current `CPU_PLS` to the VP, sets **VTL 0** active, initializes platform-specific VP state through the **VAL**, allocates per-VTL VMCS/VMCB structures and per-VTL SLAT tables, initializes the real-mode emulator, and sets the VMCS host-state to return to the hypervisor’s VAL dispatch loop.

The VP layer also allocates shared guest/hypervisor pages such as the **hypercall page**, and for each VTL the **assist** and **intercept message** pages, which are used to expose code/data paths between the guest and the hypervisor.

When construction finishes, the VP dispatch thread activates the VP and its **SynIC**, restores the initial register context into the VMCS/VMCB if this is the root BSP VP, signals that VP initialization is complete so the main system thread can enter idle, and enters the **VAL dispatch loop**.

Because the VP is new, the VAL dispatch loop prepares it for first execution and performs **`VMLAUNCH`**, causing execution to resume exactly where **HvLoader** transferred control to the hypervisor, except now Windows boot continues inside the newly created **root partition** rather than directly on raw hardware.

<p align="center"><img src="./assets/hyper-v-startup.png" width="800px" height="auto"></p>

## The hypervisor memory manager

Hyper-V manages physical memory through **memory compartments**. Before startup, `Hvloader.dll` estimates and reserves the pages required by the hypervisor and root partition, including memory for the IOMMU, PFN database, SLAT tables, and HAL address space. These pages are then used to create the root compartment.

<p align="center"><img src="./assets/hypervisor-memory-compartment.png" width="500px" height="auto"></p>

A compartment owns a set of deposited physical pages organized by NUMA node and tracked through the hypervisor’s PFN database. Child compartments receive pages from the root and return their remaining pages when destroyed. If an internal hypervisor allocation fails, the system crashes; if a child partition lacks memory, Hyper-V returns `INSUFFICIENT_MEMORY`, allowing the root partition to deposit more pages with `HvDepositMemory` before retrying the operation.

Each compartment also receives a private virtual-address range called a **zone**. Hyper-V uses a single root page table with reserved entries that dynamically switch mappings between compartment zones and virtual-processor address spaces.

### Partitions’ physical address space

Each partition has a physical address space that uses **second-level address translation** (SLAT) to map **guest physical addresses** (GPAs) to **system physical addresses** (SPAs). SLAT is called EPT by Intel, NPT by AMD, and Stage 2 Address Translation by ARM. A guest therefore performs two translations: its OS translates virtual addresses to GPAs, and the processor uses SLAT to translate those GPAs to SPAs.

<p align="center"><img src="./assets/slat.png" width="500px" height="auto"></p>

The root partition uses an **identity mapping**, where each GPA maps to the same SPA. Hyper-V creates separate SLAT tables for every supported **Virtual Trust Level** (VTL); these tables contain similar mappings but apply different permissions to isolate the VTLs. Hypervisor-owned pages are reserved and inaccessible to the partition.

For a **child** partition, Hyper-V initially creates only the **root SLAT** table for each VTL. The VID driver then allocates physical pages from the root partition, asks Hyper-V to pre-create the required SLAT hierarchy, and maps the pages using the `HvMapGpaPages` hypercall. If the child’s memory compartment lacks pages for these structures, VID deposits additional memory into it. Hyper-V finally builds the mappings with VTL-specific protections, using **large pages** where possible to reduce the number of translation levels.

### Address space isolation

Speculative-execution attacks can allow a guest VM to infer sensitive data left in shared CPU resources by the hypervisor or root partition. Hyper-V mitigates this through **HyperClear**, which combines the **core scheduler**, **virtual-processor address-space isolation**, and **sensitive-data scrubbing**. The core scheduler prevents **sibling SMT** threads from simultaneously running virtual processors belonging to different partitions, reducing cross-VM leakage through shared caches.

<p align="center"><img src="./assets/hyper-clear-mitigation.png" width="400px" height="auto"></p>

Although the hypervisor uses one global page-table root, it reserves **two PML4 entries** - covering a 1-TB virtual-address range - for a VP’s **private data**. During VP creation and thread switches, Hyper-V replaces these entries so that the executing thread can access only its current VP’s stack and private structures. “Private address space” is therefore technically a private **range** dynamically inserted into the global address space.

Within this range, memory zones (`MM_ZONE`) organize per-VP secrets. A zone contains page directories that can be attached or detached by changing its **PDPTEs**, while switching the entire VP-private range requires updating only two PML4 entries. This makes address-space isolation relatively inexpensive while preventing a hypervisor thread from accessing another VP’s sensitive data.

<p align="center"><img src="./assets/hypervisor-private-address-spaces-and-private-memory-zones.png" width="500px" height="auto"></p>

### Dynamic memory

Hyper-V dynamic memory adjusts a VM’s physical memory according to its workload, reclaiming unused pages from underutilized VMs and assigning them to VMs under memory pressure. It combines the **NT memory manager’s** hot-add and hot-remove support, Hyper-V’s SLAT mappings, and communication between the guest’s `Dmvsc.sys` driver and the root partition’s `Vmdynmem.dll` module over VMBus.

Windows supports memory hot-plugging through its **sparse PFN database**. It reserves enough virtual-address space to describe the maximum possible physical memory but maps PFN entries only for memory that is actually present. Hot-added memory receives new PFN mappings and is placed on the free-page list, while removed memory has its PFNs marked unusable without releasing the reserved PFN virtual-address space, allowing the same range to be added again later.

Every second, `Dmvsc.sys` reports guest memory-pressure statistics to the root partition. The **VMMS balancer** uses these reports and the root’s available memory to decide whether memory should be added or reclaimed. For hot-add, the root allocates physical pages, VID maps them into the guest through SLAT, and the guest calls `MmAddPhysicalMemory`. For removal, the guest uses `MmRemovePhysicalMemory` after ensuring the pages are free, zeroed, or safely pageable; VID then removes their guest mappings and returns the underlying pages to the root partition.

## Hyper-V schedulers

The Hyper-V scheduler determines which virtual processor runs on each physical processor, particularly when the system has more virtual processors than available hardware threads. At the end of each time slice, it selects the next virtual processor to execute. Hyper-V supports three scheduler implementations and exposes a common scheduler API that redirects scheduling operations to the active implementation.

### The classic scheduler

The classic scheduler uses a **round-robin** policy in which runnable virtual processors normally receive equal time slices. It supports VP affinity and NUMA-aware placement but generally does not know what a guest VP is executing. One exception is the **spin-lock enlightenment**, through which a Windows guest informs Hyper-V that it is actively waiting on a lock, allowing the scheduler to preempt it early and run another VP instead of wasting CPU cycles.

Because equal scheduling can perform poorly on **oversubscribed** systems, the classic scheduler provides three controls: **reservations** guarantee a VM a minimum percentage of CPU capacity, **limits cap** its maximum CPU consumption, and **weights** determine its relative scheduling priority after all reservations have been satisfied.

### Core Scheduler

SMT exposes multiple logical processors from one physical core, but those processors share execution resources and caches. If the classic scheduler places VPs from different VMs on sibling SMT threads, one VM could potentially observe information belonging to another through side-channel attacks.

The **core scheduler**, introduced with Windows Server 2016, addresses this by scheduling entire virtual cores onto physical cores. A physical core’s sibling threads may run only VPs belonging to the same VM; if a VM has no VP available for one sibling, that logical processor remains unused rather than running a VP from another VM. This creates a stronger isolation boundary while still allowing a guest to recognize and use SMT normally.

Its fundamental scheduling object is the **scheduling unit**, representing either an SMT **group of VPs** or one VP for a non-SMT VM. Reservations, limits, and weights are applied to this unit, which can be blocked, resumed, or migrated between physical cores. The gang scheduler assigns its VP threads to sibling logical processors, while per-core dispatchers perform thread switching and maintain execution state. A global scheduler manager balances scheduling units across physical cores.

### Root Scheduler

The root scheduler, introduced in Windows 10 RS4, **delegates** guest VP scheduling to the **NT scheduler** in the **root** partition. It was designed primarily for lightweight, virtualization-based containers such as *Windows Defender Application Guard*, allowing container workloads to be scheduled and measured like ordinary host workloads.

For each guest VP, the VID driver creates a kernel-mode **VP-dispatch thread** inside the VM’s minimal **VMMEM** process. The NT scheduler schedules these as normal threads while applying VM-specific policies. Each thread repeatedly invokes `HvDispatchVp`, which switches from the root VP to the guest VP and lets it execute until it blocks, generates an intercept, receives a root-directed interrupt, or is preempted when its time slice expires. The dispatch thread then waits if the VP is blocked or processes any returned intercept, sometimes forwarding it to the user-mode VM Worker Process.

<p align="center"><img src="./assets/hyper-v-core-scheduler.png" width="400px" height="auto"></p>

Hyper-V communicates scheduling events to the root through a **shared page** and **synthetic interrupts**. This design gives the root VP priority and centralized control, but context switching is more expensive because switching between guest VPs requires returning through the root partition. Migrating a guest VP between physical processors may also require its previous processor to flush the saved VP context before execution can continue.

## Hypercalls and the hypervisor TLFS

Hypercalls allow root and child partitions to request services from the hypervisor. Executing the architecture-specific instruction - `VMCALL` on Intel, `VMMCALL` on AMD, or `HVC` on ARM64 - causes a VM exit, after which the hypervisor reads a 64-bit control value containing the hypercall code, properties, and calling convention.

<p align="center"><img src="./assets/hyper-v-hypercall-structure.png" width="400px" height="auto"></p>

**Standard hypercalls** pass input and output through **guest-memory buffers**, **fast hypercalls** pass up to **16 bytes** directly through **registers**, and **extended fast hypercalls** use additional `XMM` registers to pass up to **112 bytes**. A **simple** hypercall performs one operation, while a **rep** hypercall processes a list of elements and returns the number completed.

<p align="center"><img src="./assets/hyper-v-hypercall-result.png" width="400px" height="auto"></p>

Long operations use hypercall continuation. After ~50 microseconds, Hyper-V may return to the caller without advancing the instruction pointer, allowing interrupts and other VPs to run. The same hypercall instruction is executed again later to continue the operation.

Windows drivers normally invoke hypercalls through `WinHvr.sys` in the root partition or `WinHv.sys` in a child partition instead of executing the platform-specific instruction directly. The available interfaces and calling conventions are documented in the Hyper-V *Top-Level Functional Specification* (TLFS).

## Intercepts

Host intercepts allow the root partition to emulate physical hardware for unmodified guest OSs. Hyper-V can trap operations such as **I/O-port** and **MSR accesses**, **CPUID**, **exceptions**, **register accesses**, and **hypercalls**, giving the root partition an opportunity to reproduce the behavior the guest expects from real hardware.

The VM Worker Process registers the required intercepts through the VID driver, which installs them using `HvInstallIntercept`. When an intercepted event occurs, Hyper-V suspends the guest VP and sends a message to the **root through SynIC**. The root’s synthetic interrupt handler forwards the event to VID, which processes it - possibly with other virtualization-stack components - and then clears the VP’s intercept suspension so guest execution can resume.

## The synthetic interrupt controller (SynIC)

The SynIC virtualizes interrupt delivery for root and child partitions. Each VP receives a SynIC for every **supported VTL**. It handles both **external interrupts**, originating from devices or other partitions, and **synthetic interrupts**, generated by the hypervisor and directed to a specific VP.

- Hyper-V supports three levels of **APIC virtualization**:
  - In the **standard mode**, guest APIC accesses and interrupt delivery cause **VM exits**, allowing the hypervisor to **emulate** the APIC and **inject** interrupts through the VMCS or VMCB.
  - **APIC emulation** reduces exits by maintaining **APIC state** in a hardware-supported **virtual-APIC page**.
  - **Posted interrupts** are the most efficient option because compatible device interrupts can be delivered directly to a guest **without a VM exit**, which is particularly useful for directly assigned devices.

SynIC uses interrupt descriptors to identify the interrupt’s destination, target VP, vector, and VTL. External interrupts normally go to the root partition or hypervisor, although directly assigned devices can target a child partition.

<p align="center"><img src="./assets/hyper-v-physical-interrupt-descriptor.png" width="300px" height="auto"></p>

Synthetic interrupts are checked whenever a VP is scheduled and are important for enlightened services such as timers and Virtual Secure Mode. The root can also inject a virtual interrupt into a child using `HvAssertVirtualInterrupt`.

### Inter-partition communication

SynIC supports communication between partitions through **messages** and **events**, both delivered to a target VP using synthetic interrupts. Communication uses a pre-created connection associated with a destination port. VMBus relies heavily on this mechanism: its root driver creates a port with `HvCreatePort`, connects it using `HvConnectPort`, and sends data with `HvPostMessage`.

Each SynIC provides 16 synthetic interrupt sources, allowing up to 16 message queues. Messages have a fixed size of 256 bytes, of which 240 bytes are payload. The target partition receives them through a **Synthetic Interrupt Message Page (SIMP)** shared with the hypervisor.

- Hyper-V supports three port types:
  - **Message ports** provide **ordered** delivery of small messages and are mainly used to establish or tear down VMBus channels.
  - **Event ports** carry notification flags and are used for synchronization, such as notifying a partition that data is available in a VMBus ring buffer.
  - **Monitor ports** optimize event signaling by letting the sender set a bit in shared memory, which the hypervisor later detects and converts into an interrupt, avoiding an immediate VM exit for every signal.

## The Windows hypervisor platform API and EXO partitions

Security features such as VSM require Hyper-V to run even when the user is not running traditional VMs. Because Hyper-V starts before Windows and owns the processor’s virtualization extensions, third-party hypervisors cannot access those extensions directly. The **Windows Hypervisor Platform (WHP)** solves this by allowing products such as VMware, QEMU, VirtualBox, and Android emulators to run their own virtualization stacks on top of Hyper-V.

WHP exposes user-mode APIs through `WinHvPlatform.dll` and `WinHvEmulation.dll`, backed by the VID and WinHvr drivers. A third-party VMM uses these APIs to create and configure a partition, create its VPs, allocate and map guest physical memory, initialize firmware and registers, and execute each VP with `WHvRunVirtualProcessor`. Execution returns when the guest produces an exit that must be handled by the third-party virtualization stack.

<p align="center"><img src="./assets/hyper-v-platform-api.png" width="500px" height="auto"></p>

Most requests reach VID through `\Device\VidExo`. Performance-sensitive operations can instead invoke the hypervisor directly from user mode through a **doorbell page**: accessing this specially marked invalid page causes a VM exit that Hyper-V interprets as a hypercall, avoiding a transition through the Windows kernel.

WHP creates minimal **EXO partitions**, which require the root scheduler. EXO partitions are **VA-backed**, use the third-party VMM process itself to host guest memory, support **only VTL 0**, and omit Hyper-V-specific synthetic privileges. They must also implement their own timing because Hyper-V does not provide them with a virtual clock interrupt source.

## Nested Virtualization

Nested virtualization allows a guest VM to run its own hypervisor and nested VMs. The bare-metal **L0 hypervisor** exposes emulated virtualization extensions to an L1 guest, which runs the **L1 hypervisor**. That hypervisor can then create and run **L2 guests**.

This capability is implemented largely in software: virtualization instructions executed by L1 cause VM exits to L0, which validates and emulates their effects. Nested virtualization must be explicitly enabled for the L1 VM; otherwise, executing virtualization instructions generates a general-protection exception.

On Intel systems, Hyper-V implements nested virtualization through VT-x emulation and nested address translation. L0 maintains three structures for an L2 VP: a **nested VMCS** containing Hyper-V’s software bookkeeping, a **virtual VMCS** containing the virtualization state visible and modifiable by L1, and a **physical VMCS** loaded into the processor by L0 when L2 executes. When L1 issues `VMLAUNCH`, L0 intercepts it, prepares the physical VMCS from the virtualized state, and starts the L2 VP.

<p align="center"><img src="./assets/hyper-v-nested-vmcs.png" width="500px" height="auto"></p>


### Emulation of the VT-x virtualization extensions

Hyper-V can emulate Intel VT-x for both enlightened and nonenlightened L1 hypervisors, although only Hyper-V nested inside Hyper-V is officially supported. With a **nonenlightened** L1, every VT-x instruction causes an exit to L0. When L1 activates its guest VMCS with `VMPTRLD`, L0 associates it with a nested VMCS and redirects subsequent VMCS accesses to a virtual VMCS.

When L1 executes `VMLAUNCH`, L0 copies the L2 guest state from the **virtual VMCS** into a **real hardware VMCS**, configures its host fields to return control to L0, prepares nested address translation, and enters L2. Any L2 VM exit first returns to L0, which restores L1 and presents the event to it as a **synthetic VM exit**. L1 handles the event and eventually executes `VMRESUME`, which L0 intercepts to restart L2.

Because trapping every VMCS operation is expensive, an enlightened L1 can use an **enlightened VMCS**, a shared memory page that both L0 and L1 can access directly. L1 modifies virtualization state in this page without executing trapping VT-x instructions, and L0 synchronizes it with the virtual VMCS when L2 is entered. This substantially reduces nested-virtualization overhead.

> Note It is worth mentioning that for nonenlightened scenarios, the L0 hypervisor supports another technique for preventing VMEXITs while managing nested virtualization data, called **shadow VMCS**. Shadow VMCS is a hardware optimization very similar to the enlightened VMCS.

### Nested address translation

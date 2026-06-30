# Chapter 10 Memory Management

## Introduction to the Memory Manager

- Because the VAS might be larger or smaller than the PM on the machine, the memory manager has two primary tasks:
  - **Translating, or mapping**, a process’s VAS into PM so that when a thread running in the context of that process reads or writes to the VAS, the correct PA is referenced. (The subset of a process’s VAS that is **physically resident** is called the **working set**).
  - **Paging** some of the contents of memory to disk when it becomes overcommitted — that is, when running threads or system code try to use more PM than is currently available — and bringing the contents back into PM when needed.

### Memory Manager Components

- Memory manager consists of the following components:
  - A set of **executive system services** for allocating, deallocating, and managing virtual memory, most of which are exposed through the Windows API or kernel-mode device driver interfaces.
  - **Translation-not-valid** and **access fault trap handler** for resolving hardware-detected memory management exceptions and making virtual pages resident on behalf of a process.
  - Six **key top-level routines**, each running in one of six different kernel-mode threads in the `System` process:
    - **Balance set manager** (`KeBalanceSetManager`, priority 16). It calls an inner routine, the **working set manager** (`MmWorkingSetManager`), once per second as well as when free memory falls below a certain threshold. The working set manager drives the overall memory management policies, such as **working set trimming, aging, and modified page writing**.
    - **Process/stack swapper** (`KeSwapProcessOrStack`, priority 23) performs both process and kernel thread **stack inswapping** and **outswapping**. The balance set manager and the thread-scheduling code in the kernel awaken this thread when an inswap or outswap operation needs to take place.
    - **Modified page writer** (`MiModifiedPageWriter`, priority 17) writes **dirty pages** on the modified list back to the appropriate paging files. This thread is awakened when the size of the modified list needs to be reduced.
    - **Mapped page writer** (`MiMappedPageWriter`, priority 17) writes dirty pages in mapped files to disk (or remote storage). It is awakened when the size of the modified list needs to be reduced or if pages for mapped files have been on the modified list for more than 5 minutes. This second modified page writer thread is necessary because it can generate page faults that result in requests for free pages. If there were no free pages and there was only one modified page writer thread, the system could deadlock waiting for free pages.
    - **Segment dereference thread** (`MiDereferenceSegmentThread`, priority 18) is responsible for cache reduction as well as for page file growth and shrinkage. (For example, if there is no VAS for paged pool growth, this thread trims the page cache so that the paged pool used to anchor it can be freed for reuse.)
    - **Zero page thread** (`MmZeroPageThread`, base priority 0) zeroes out pages on the free list so that a cache of zero pages is available to satisfy future demand-zero page faults. Unlike the other routines described here, this routine is not a top-level thread function but is called by the top-level thread routine Phase1Initialization. MmZeroPageThread never returns to its caller, so in effect the Phase 1 Initialization thread becomes the zero page thread by calling this routine. Memory zeroing in some cases is done by a faster function called `MiZeroInParallel`.

### Internal Synchronization

- the memory manager is **fully reentrant** and supports simultaneous execution on multiprocessor systems.
- achieved by using several different internal synchronization mechanisms, such as spinlocks, to control access to its own internal data structures. Some of theses accesses are:
  - Dynamically allocated portions of the system VAS
  - System working sets
  - Kernel memory pools
  - The list of loaded drivers
  - The list of paging files
  - Physical memory lists
  - Image base randomization (ASLR) structures
  - Each individual entry in the page frame number (PFN) database.

### Examining Memory Usage

- **vmmap**, **rammap** or **process explorer** are nice tools to examine memory usage.

  ```c
      0: kd> !vm
  Page File: \??\C:\pagefile.sys
  Current:   4193784 Kb  Free Space:   4193780 Kb
  Minimum:   4193784 Kb  Maximum:     12581352 Kb

  Physical Memory:          1048446 (    4193784 Kb)
  Available Pages:           820027 (    3280108 Kb)
  ResAvail Pages:            973277 (    3893108 Kb)
  Locked IO Pages:                0 (          0 Kb)
  Free System PTEs:        33494979 (  133979916 Kb)
  Modified Pages:              4801 (      19204 Kb)
  Modified PF Pages:           4771 (      19084 Kb)
  Modified No Write Pages:        0 (          0 Kb)
  NonPagedPool Usage:          9217 (      36868 Kb)
  NonPagedPool Max:          774140 (    3096560 Kb)
  PagedPool  0:               32876 (     131504 Kb)
  PagedPool  1:                4124 (      16496 Kb)
  PagedPool  2:                   0 (          0 Kb)
  PagedPool  3:                   0 (          0 Kb)
  PagedPool  4:                  57 (        228 Kb)
  PagedPool Usage:            37057 (     148228 Kb)
  PagedPool Maximum:       33554432 (  134217728 Kb)
  Processor Commit:             274 (       1096 Kb)
  Session Commit:              7647 (      30588 Kb)
  Syspart SharedCommit 0
  Shared Commit:               9386 (      37544 Kb)
  Special Pool:                   0 (          0 Kb)
  Kernel Stacks:               8220 (      32880 Kb)
  Pages For MDLs:              2331 (       9324 Kb)
  Pages For AWE:                  0 (          0 Kb)
  NonPagedPool Commit:            0 (          0 Kb)
  PagedPool Commit:           37108 (     148432 Kb)
  Driver Commit:               3082 (      12328 Kb)
  Boot Commit:                    0 (          0 Kb)
  System PageTables:              0 (          0 Kb)
  VAD/PageTable Bitmaps:       2214 (       8856 Kb)
  ProcessLockedFilePages:         0 (          0 Kb)
  Pagefile Hash Pages:            0 (          0 Kb)
  Sum System Commit:          70262 (     281048 Kb)
  Total Private:             107649 (     430596 Kb)
  Misc/Transient Commit:      55573 (     222292 Kb)
  Committed pages:           233484 (     933936 Kb)
  Commit limit:             2096416 (    8385664 Kb)
  ```

## Services Provided by the Memory Manager

- the Windows API has three groups of functions for managing memory in applications:
  - **heap functions** (_Heapxxx_ and the older interfaces _Localxxx_ and _Globalxxx_, which internally make use of the _Heapxxx_ APIs), which may be used for allocations **smaller than a page**;
  - virtual memory functions, which operate with page granularity (_Virtualxxx_);
  - and **memory mapped file functions** (`CreateFileMapping`, `CreateFileMappingNuma`, `MapViewOfFile`, `MapViewOfFileEx`, and `MapViewOfFileExNuma`).

### Large and Small Pages

- page size:
  | Architecture | Small Page Size| Large Page Size | Small Pages per Large Page|
  |--------------|-----------------|-----------------|---------------------------|
  |x86 | 4 KB| 4 MB (2 MB if PAE enabled) |1,024 (512 with PAE) |
  |x64 | 4 KB | 2 MB | 512 |
  |IA64 | 8 KB | 16 MB |2,048 |
- primary advantage of large pages is **speed of address translation** for references to other data within the large page.
- first reference to any byte within a large page will cause the hardware’s **translation look-aside buffer TLB** to have in its cache the information necessary to translate references to any other byte within the large page.
- to take advantage of large pages on systems with more than 2 GB of RAM, Windows maps with large pages:
  - the **core OS images** (`Ntoskrnl.exe` and `Hal.dll`)
  - as well as **core OS data** (such as the initial part of **nonpaged pool** and the data structures that describe the state of each physical memory page).
  - also automatically maps **I/O space requests** (calls by device drivers to `MmMapIoSpace`) with large pages if the request is of satisfactory large page length and alignment.
  - user mode apps can use `MEM_LARGE_PAGE` during mem alloc.
  - drivers can set _LargePageDrivers_.
- few notes regarding large pages:
  - allocating large pages could fail as freeing physical memory does become **fragmented** as the system runs.
  - tt is not possible to specify anything but **read/write** access to large pages.
  - the memory is also always **nonpageable**, because the page file system does not support large page.
  - if a large page contains, for example, both **read-only code** and **read/write data**, the page must be marked as **read/write**, which means that the code will be writable. This means that device drivers or other kernel-mode code could, as a result of a bug, modify what is supposed to be **read-only** OS or driver code without causing a memory access violation.

### Reserving and Committing Pages

- pages in a process VAS are **free, reserved, committed, or shareable**.
- **committed** and **shareable** pages are pages that, when accessed, ultimately translate to valid pages in physical memory.
- **shared pages** are usually mapped to a view of a section, which in turn is part or all of a file, but may instead represent a portion of page file space.
  - all shared pages can potentially be shared with other processes.
  - sections are exposed in the Windows API as **file mapping objects**.

### Commit Limit

- **commitment or commit charge**; this is the first of the two numbers, which represents the total of all committed virtual memory in the system.
- **system commit limit** or simply the commit limit, on the amount of committed virtual memory that can exist at any one time.
  - corresponds to the current total size of all paging files, plus the amount of RAM that is usable by the OS.
  <p align="center"><img src="https://i.imgur.com/WwRPq6Y.png" width="400px" height="auto"></p>

### Locking Memory

- pages can be locked in memory in two ways:
  - Windows applications can call the `VirtualLock` function to lock pages in their process working set.
    - pages locked using this mechanism remain in memory until explicitly unlocked or until the process that locked them terminates.
    - the number of pages a process can lock **can’t exceed its minimum working set size minus eight pages**.
    - therefore, if a process needs to lock more pages, it can increase its working set minimum with the `SetProcessWorkingSetSizeEx`.
  - device drivers can call the kernel-mode functions `MmProbeAndLockPages`, `MmLockPagableCodeSection`, `MmLockPagableDataSection`, or `MmLockPagableSectionByHandle`.
    - pages locked using this mechanism remain in memory until explicitly unlocked.
    - the last three of these APIs **enforce no quota** on the number of pages that can be locked in memory because the resident available page charge is obtained when the driver first loads; this ensures that it can never cause a system crash due to overlocking.
    - for the first API, quota charges must be obtained or the API will return a failure status.

### Allocation Granularity

- Windows aligns **each region** of reserved process address space to begin on an integral boundary defined by the value of the system allocation granularity, which can be retrieved from the Windows `GetSystemInfo` or `GetNativeSystemInfo` function.
- this value is **64 KB**, a granularity that is used by the memory manager to efficiently allocate metadata (for example, VADs, bitmaps, and so on) to support various process operations.
- Windows kernel-mode code isn’t subject to the same restrictions; it can reserve memory on a **single-page granularity** (although this is not exposed to device drivers for the reasons detailed earlier).
- Windows ensures that the **size and base** of the region is a **multiple** of the system page size, whatever that might be.

### Shared Memory and Mapped Files

- **shared memory** can be defined as memory that is visible to more than one process or that is present in more than one process VAS.
- for example, if two processes use the same DLL, it would make sense to load the referenced code pages for that DLL into physical memory **only once** and share those pages between all processes that map the DLL. <p align="center"><img src="https://i.imgur.com/ORc9BlC.png" width="300px" height="auto"></p>
- a **section object** can be connected to an **open file on disk (called a mapped file)** or **to committed memory (to provide shared memory)**.
- sections mapped to committed memory are called **page-file-backed sections** because the pages are written to the paging file (**as opposed to a mapped file**) if demands on physical memory require it.
- as with any other empty page that is made visible to user mode (such as private committed pages), shared committed pages are **always zero-filled** when they are first accessed to ensure that no sensitive data is ever leaked.
- to create a section object, call the Windows `CreateFileMapping` or `CreateFileMappingNuma` function, specifying the file handle to map it to (or **INVALID_HANDLE_VALUE** for a page-file-backed section).
- a section object can refer to files that are **much larger** than can fit in the address space of a process.
- to access a very large section object, a process can map only the portion of the section object that it requires (called a view of the section) by calling the `MapViewOfFile`, `MapViewOfFileEx`, or `MapViewOfFileExNuma` function and then specifying the range to map.
- mapping views permits processes to **conserve address space** because only the views of the section object needed at the time must be mapped into memory.

### Protecting Memory

- Windows provides memory protection in four primary ways.
  1. all **systemwide data structures** and **memory pools used by kernel-mode** system components can be accessed only while in **kernel** mode.
     - user-mode threads can’t access these pages
     - if they attempt to do so, the hardware generates a fault, which in turn the memory manager reports to the thread as an access violation.
  2. each process has a separate, **private address space**, protected from being accessed by any thread belonging to another process.
     - even shared memory is not really an exception to this because each process accesses the shared regions using addresses that are part of its **own VAS**.
  3. in addition to the implicit protection virtual-to-physical address translation offers, all processors supported by Windows provide some form of **hardware-controlled memory protection** (such as read/write, read-only, and so on).
  4. shared memory section objects have standard Windows **ACLs** that are checked when processes attempt to open them, thus limiting access of shared memory to those processes with the proper rights.

### No Execute Page Protection

- **No execute** page protection (also referred to as **data execution prevention (DEP)**) causes an attempt to transfer control to an instruction in a page marked as “no execute” to generate an access fault.
  - prevent stack-based overflows
  - catch poorly written programs that don’t correctly set permissions on pages from which they intend to execute code.
- in a illegal reference:
  - in kernel, fails with `ATTEMPTED_EXECUTE_OF_NOEXECUTE_MEMORY` bugcheck code.
  - in user mode, fails with `STATUS_ACCESS_VIOLATION` exception.
- on 32-bit x86 systems that support DEP, bit 63 in the page table entry (PTE) is used to mark a page as nonexecutable.
  - DEP feature is available only when the processor is running in **Physical Address Extension (PAE)** mode.
  - requires loading the PAE kernel `%SystemRoot%\System32\Ntkrnlpa.exe`.
  - the OS loader automatically loads the PAE kernel on 32-bit systems that support hardware DEP.
  - applied only to **thread stacks** and **usermode pages**, not to paged pool and session pool.
  - processes depends on the value of the BCD nx option (OptIn, OptOut, AlwaysOn, AlwaysOff)
- on 64-bit versions of Windows,:
  - execution protection is **always applied to all 64-bit processes and device drivers**.
  - can be disabled only by setting the nx BCD option to _AlwaysOff_.
  - applied to **thread stacks** (both user and kernel mode), **usermode pages** not specifically marked as **executable**, **kernel paged pool**, and **kernel session pool**.
- on Windows client versions (both 64-bit and 32-bit) execution protection for 32-bit processes is configured by default to apply only to **core Windows operating system executables (OptIn)**.
- even if you force DEP to be enabled, there are still other methods through which applications can disable DEP for their own image:
  - image loader will verify the signature of the executable against known **copy-protection mechanisms** (such as SafeDisc and SecuROM) and disable execution protection to provide compatibility with older copy protected software such as computer games.
- if the system is in _OptIn_ or _OptOut_ mode and executing a 32-bit process, the `SetProcessDEPPolicy` function allows a process to dynamically disable DEP or to permanently enable it.

#### Software Data Execution Prevention

- for older processors that do not support hardware no execute protection, Windows supports limited **software DEP**.
  - reduces SEH exploits.
  - if image is not built with **/SAFESEH**, software DEP guards against SEH exploits with **Structured Exception Handler Overwrite Protection (SEHOP)**.
  - **ALSR** make it harder to know the location of the function pointed to by the symbolic exception registration record.
- to futher mitiguate again SEH exploits when /SAFESEH is not present:
  - a mechanism called **Image Dispatch Mitigation** ensures that the SEH handler is located within the same image section as the function that raised an exception, which is normally the case for most programs (although not necessarily, since some DLLs might have exception handlers that were set up by the main executable, which is why this mitigation is off by default).
  - Executable Dispatch Mitigation further makes sure that the SEH handler is located within an executable page—a less strong requirement than Image Dispatch Mitigation, but one with fewer compatibility issues.
- two other methods for software DEP that the system implements are:
  - **stack cookies**: relies on the compiler to insert special code at the beginning and end of each potentially exploitable function.
    - the code saves a special numerical value (the cookie) on the stack on entry and validates the cookie’s value before returning to the caller saved on the stack.
    - the cookie value is computed for each boot when executing the first user-mode thread, and it is saved in the `KUSER_SHARED_DATA` structure. The image loader reads this value and initializes it when a process starts executing in user mode.
  - the cookie value that is calculated is also saved for use with the `EncodeSystemPointer` and `DecodeSystemPointer` APIs, which implement pointer encoding
    - static pointers that are dynamically called runs the risk of having malicious code **overwrite the pointer values** with code that the malware controls.
    - These APIs provide similar protection but with a **per-process cookie** (created on demand) instead of a **per-system cookie**.

### Copy-on-Write

- copy-on-write page protection is an optimization the memory manager uses to conserve physical memory.
- defers making a copy of the pages until the page is written to.
- two processes are sharing three pages, each marked copy-on-write, but neither of the two processes has attempted to modify any data on the pages.
<p align="center"><img src="https://i.imgur.com/qbiqSC3.png" width="400px" height="auto"></p>
- it a thread in either process writes to a page, a memory management fault is generated.
  - memory manager sees that the write is to a copy-on-write page,
  - it allocates a new read/write page in physical memory,
  - copies the contents of the original page to the new page
  - updates the corresponding page-mapping information in this process to point to the new location,
  - the newly copied page is now private to the process that did the writing and isn’t visible to the other process still sharing the copy-on-write page.
  <p align="center"><img src="https://i.imgur.com/Hx0IPIE.png" width="400px" height="auto"></p>

### Address Windowing Extensions

- an app that needs to make more than 2 GB (or 3 GB) of data easily available in a single process could do so via **file mapping**, remapping a part of its address space into various portions of a large file.
- Windows provides a set of functions called **Address Windowing Extensions (AWE)** to allow a 32-bits process to allocate more physical memory than can be represented in its VAS.
- it then can access the physical memory by mapping a portion of its VAS into selected portions of the physical memory at various times.
- allocating and using memory via the AWE functions is done in three steps:
  1. allocating the physical memory to be used: `AllocateUserPhysicalPages` or `AllocateUserPhysicalPagesNuma`.
  2. creating one or more regions of VAS to act as windows to map views of the physical memory. The application uses the Win32 `VirtualAlloc(Ex)`, or `VirtualAllocExNuma` function with the **MEM_PHYSICAL** flag.
  3. to actually use the memory, the app uses `MapUserPhysicalPages` or `MapUserPhysicalPagesScatter` to map a portion of the physical region allocated in step 1 into one of the virtual regions, or windows, allocated in step 2.
  <p align="center"><img src="https://i.imgur.com/UD6vywq.png" width="450px" height="auto"></p>

## Kernel-Mode Heaps (System Memory Pools)

- at system initialization, the memory manager creates two dynamically sized memory pools, or heaps, that most kernel-mode components use to allocate system memory:
  - **nonpaged pool**: consists of ranges of system virtual addresses that are guaranteed to reside in physical memory at all times + can be accessed from any IRQL.
  - **paged pool**: a region of virtual memory in system space that can be paged into and out of the system. Device drivers that don’t need to access the memory from DPC/dispatch level or above can use paged pool. It is accessible from any process context.
- systems start with **four paged pools** and **one nonpaged pool**; more are created, up to a maximum of **64**, depending on the number of NUMA nodes on the system.
- in addition to paged/nonpaged pool, there is a pool region in **session space**, which is used for data that is common to all processes in the session.
- also, a **special pool**: allocations from special pool are surrounded by pages marked as no-access to help isolate problems in code that accesses memory before or after the region of pool it allocated.

### Pool Sizes

- initial size: 3% of ram, if less than 40 MB, use 40 MB insted as long as 10% of RAM results in more than 40MB; otherwise 10 percent of RAM is chosen as a minimum.

| Pool Type | Maximum on 32-Bit                                    | Maximum on 64-Bit Systems                              |
| --------- | ---------------------------------------------------- | ------------------------------------------------------ |
| Nonpaged  | 75% of physical memory or 2 GB, whichever is smaller | 75% of physical memory or 128 GB, whichever is smaller |
| Paged     | 2 GB                                                 | 128 GB                                                 |

```c
Page File: \??\C:\pagefile.sys
  Current:   4193784 Kb  Free Space:   4193780 Kb
  Minimum:   4193784 Kb  Maximum:     12581352 Kb

Physical Memory:          1048446 (    4193784 Kb)
Available Pages:           826358 (    3305432 Kb)
ResAvail Pages:            974026 (    3896104 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:        33496170 (  133984680 Kb)
Modified Pages:              4825 (      19300 Kb)
Modified PF Pages:           4808 (      19232 Kb)
Modified No Write Pages:        0 (          0 Kb)
NonPagedPool Usage:          9111 (      36444 Kb)
NonPagedPool Max:          774140 (    3096560 Kb)
PagedPool  0:               32829 (     131316 Kb)
PagedPool  1:                4109 (      16436 Kb)
PagedPool  2:                   0 (          0 Kb)
PagedPool  3:                   0 (          0 Kb)
PagedPool  4:                  54 (        216 Kb)
PagedPool Usage:            36992 (     147968 Kb)
PagedPool Maximum:       33554432 (  134217728 Kb)
Processor Commit:             176 (        704 Kb)
Session Commit:              7651 (      30604 Kb)
Syspart SharedCommit 0
Shared Commit:               9231 (      36924 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:               7707 (      30828 Kb)
Pages For MDLs:              2331 (       9324 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:            0 (          0 Kb)
PagedPool Commit:           36992 (     147968 Kb)
Driver Commit:               3082 (      12328 Kb)
Boot Commit:                    0 (          0 Kb)
System PageTables:              0 (          0 Kb)
VAD/PageTable Bitmaps:       1993 (       7972 Kb)
ProcessLockedFilePages:         0 (          0 Kb)
Pagefile Hash Pages:            0 (          0 Kb)
Sum System Commit:          69163 (     276652 Kb)
Total Private:             102655 (     410620 Kb)
Misc/Transient Commit:      55148 (     220592 Kb)
Committed pages:           226966 (     907864 Kb)
Commit limit:             2096416 (    8385664 Kb)
```

- use _poolmon_ to monitor pool usage; or:

```c
 kd> !poolused 2

*** CacheSize too low - increasing to 128 MB

Max cache size is       : 134217728 bytes (0x20000 KB)
Total memory in cache   : 157892 bytes (0x9b KB)
Number of regions cached: 292
2257 full reads broken into 2626 partial reads
    counts: 2241 cached/385 uncached, 85.34% cached
    bytes : 371524 cached/139204 uncached, 72.74% cached
** Transition PTEs are implicitly decoded
** Prototype PTEs are implicitly decoded
..
 Sorting by NonPaged Pool Consumed

               NonPaged                  Paged
 Tag     Allocs         Used     Allocs         Used

 Cont      1094      6525216          0            0	Contiguous physical memory allocations for device drivers
 EtwB       128      5041136          4       163840	Etw Buffer , Binary: nt!etw
 AmlC        31      2031616          0            0	ACPI AMLI Pooltags
 Pool         5      1705552          0            0	Pool tables, etc.
 AmlH         3      1572864          0            0	ACPI AMLI Pooltags
 ...

 kd> !poolused 4
..
 Sorting by Paged Pool Consumed

               NonPaged                  Paged
 Tag     Allocs         Used     Allocs         Used

 CM31         0            0      19797     91705344	Internal Configuration manager allocations , Binary: nt!cm
 CM25         0            0       2764     12546048	Internal Configuration manager allocations , Binary: nt!cm
 MmRe         0            0        958     11071552	ASLR relocation blocks , Binary: nt!mm
 MmSt         0            0       2808      6790672	Mm section object prototype ptes , Binary: nt!mm
 CcPD         0            0          8      4314976	Prefetcher trace dump , Binary: nt!ccpf
 CIcr         0            0         89      2976080	Code Integrity allocations for image integrity checking , Binary: ci.dll
 Ntff         5         1600       2109      2598288	FCB_DATA , Binary: ntfs.sys
 ...
```

### Look-Aside Lists

- Windows provides a fast memory allocation mechanism called **look-aside lists**.
- the difference between pools and look-aside lists:
  - general pool allocations can **vary** in size, a look-aside list contains only **fixed-sized** blocks.
  - although the general pools are more flexible in terms of what they can supply, look-aside lists are **faster** because they don’t use any **spinlocks**.
- contents and sizes of the various system look-aside lists:

```c
1: kd> !lookaside

Lookaside "nt!CcTwilightLookasideList" @ 0xfffff80002c49280  Tag(hex): 0x6b576343 "CcWk"
    Type           =       0000  NonPagedPool
    Current Depth  =          0  Max Depth  =         34
    Size           =         32  Max Alloc  =       1088
    AllocateMisses =        534  FreeMisses =        467
    TotalAllocates =        759  TotalFrees =        715
    Hit Rate       =         29% Hit Rate   =         34%

Lookaside "nt!IopSmallIrpLookasideList" @ 0xfffff80002c49d00  Tag(hex): 0x73707249 "Irps"
    Type           =       0000  NonPagedPool
    Current Depth  =          0  Max Depth  =          4
    Size           =        280  Max Alloc  =       1120
    AllocateMisses =        217  FreeMisses =         66
    TotalAllocates =        263  TotalFrees =        113
    Hit Rate       =         17% Hit Rate   =         41%
...

Total NonPaged currently allocated for above lists =        0
Total NonPaged potential for above lists           =    75624
Total Paged currently allocated for above lists    =        0
Total Paged potential for above lists              =   367832
```

## Heap Manager

- manages allocations inside **larger memory areas** reserved using the page granularity memory allocation functions.
- the allocation granularity in the heap manager is relatively small: **8 bytes** on 32-bit systems, and **16 bytes** on 64-bit. systems.
- legacy APIs (prefixed with either **Local** or **Global**) are provided to support older Windows applications, which also internally call the heap manager, using some of its specialized interfaces to support legacy behavior.

### Types of Heaps

- each process has at least one heap: the **default process heap**.
- defaults to 1 MB in size, but it can be made bigger by using the `/HEAP` linker flag
- this size is just the initial reserve, however—it will expand automatically as needed.
- processes can also create additional **private heaps** with the `HeapCreate`.

### Heap Manager Structure

- the heap manager is structured in two layers: an **optional front-end layer** and the **core heap**.
- the core heap handles the basic functionality and is mostly common across the usermode and kernel-mode heap implementations.
- the core functionality includes the management of blocks inside segments, the management of the segments, policies for extending the heap, committing and decommitting memory, and management of the large blocks. <p align="center"><img src="https://i.imgur.com/pIABTFh.png" width="450px" height="auto"></p>
- for user-mode heaps only, an optional front-end heap layer can exist on top of the existing core functionality.
- the only front-end supported on Windows is the **Low Fragmentation Heap (LFH)**. Only one front-end layer can be used for one heap at one time.

### Heap Synchronization

- the heap manager supports **concurrent access** from multiple threads by default.
- however, if a process is **single threaded** or uses an external mechanism for **synchronization**, it can tell the heap manager to avoid the overhead of synchronization by specifying `HEAP_NO_SERIALIZE` either at heap creation or on a per-allocation basis.

### The Low Fragmentation Heap

- many Windows apps have relatively small heap memory usage (~ less than 1 MB).
  - for this class of apps, the heap manager’s best-fit policy helps keep a low memory footprint for each process.
  - however, this strategy does not scale for **large processes** and **multiprocessor** machines.
  - in these cases, memory available for heap usage might be reduced as a result of **heap fragmentation**.
  - performance can suffer in scenarios where only certain sizes are often used concurrently from different threads scheduled to run on different processors.
  - this happens because several processors need to modify the same memory location (i.e the head of the look-aside list for that particular size) at the same time, thus causing significant contention for the corresponding cache line.
- LFH avoids fragmentation by managing allocated blocks in predetermined different block-size ranges called **buckets**.
- LFH chooses the bucket that maps to the **smallest block** large enough to hold the required size.

| Buckets | Granularity | Range        |
| ------- | ----------- | ------------ |
| 1–32    | 8           | 1–256        |
| 33–48   | 16          | 257–512      |
| 49–64   | 32          | 513–1,024    |
| 65–80   | 64          | 1,025–2,048  |
| 81–96   | 128         | 2,049–4,096  |
| 97–112  | 256         | 4,097–8,194  |
| 113–128 | 512         | 8,195–16,384 |

### Heap Security Features

- heap manager have some techniques to minimize heap-based exploits:
  - metadata used by the heap for internal management is packed with high degree of randomization
  - blocks are subjects to integrity check to the header to detect buffer overruns.
  - randomization of the base address (or handle).
- by using `HeapSetInformation` with _HeapEnableTerminationOnCorruption_ class, a process can opt in for an automatic termination in case if anomalies.
- dump metadata fields from a heap block:

```c
0:000> !heap -i 001a0000
Heap context set to the heap 0x001a0000
0:000> !heap -i 1e2570
Detailed information for block entry 001e2570
Assumed heap : 0x001a0000 (Use !heap -i NewHeapHandle to change)
Header content : 0x1570F4EC 0x0C0015BE (decoded : 0x07010006 0x0C00000D)
Owning segment : 0x001a0000 (offset 0)
Block flags : 0x1 (busy )
Total block size : 0x6 units (0x30 bytes)
Requested size : 0x24 bytes (unused 0xc bytes)
Previous block size: 0xd units (0x68 bytes)
Block CRC : OK - 0x7
Previous block : 0x001e2508
Next block : 0x001e25a0
```

### Heap Debugging Features

- the heap manager leverages the 8 bytes used to store internal metadata as a consistency checkpoint which helps detecting potential heap errors and bugs:
  - **enable tail checking**: the end of the each block carries a signature that is checked when the block is released.
  - **enable free checking**: free block is filled with a pattern that is checked at various points when the heap manager needs to access the block.
  - **parameter checking**: extensive checking of the parameteres passed to the heap functions.
  - **heap validation** the entire heap is validated at each heap call.
  - **heap tagging and strack traces support**: supports specifying tags for allocation and/or captures user-mode stack traces for the heap calls to help narrow the possible causes of a heap error.

### Pageheap

- because the tail and free checking might be discovering corruptions that occurred well before the problem was detected, an additional heap debugging capability, called **pageheap**, is provided that directs all or part of the heap calls to a different heap manager.

### Fault Tolerant Heap

- Windows includes a feature called the **fault tolerant heap (FTH)** in an attempt to mitigate heap corruption problems.
- FTH is implemented in two primary components:
  - the detection component, or FTH server, and;
  - the mitigation component, or FTH client.
- the FTH client is an application compatibility shim, it intercepts the calls to the heap routines and redirects them to its own code. The FTH code implements a number of “mitigations” that attempt to allow the application to survive despite various heap-related errors.
- the activity of the fault tolerant heap can be observed in the Event Viewer.

## Virtual Address Space Layouts

- three main types of data are mapped into the VAS in Windows:
  - per-process private code and data
  - sessionwide code and data
  - systemwide code and data.
- the information that describes the process VAS, called **page tables**.
  - each process bas its own set of page tables.
  - they are stored in kernel mode.
- **session space** contains information that is common to each session:
  - consists of the processes and other system objects (such as the window station, desktops, and windows) that represent a single user’s logon session
  - each session has a session-specific paged pool area used by Win32k.sys to allocate session-private GUI data structures.
  - each session has its own copy of the Windows subsystem process (Csrss.exe) and logon process (Winlogon.exe).
- **system space** contains global OS code and data structures visible by kernel-mode code regardless of which process is currently executing. System space consists of the following components:
  - **system code** contains the OS image, HAL, and device drivers used to boot the system.
  - **nonpaged pool**: nonpageable system memory heap.
  - **paged pool**: pageable system memory heap.
  - **system cache**: VAS used to map files open in the system cache.
  - **system page table entries (PTEs)**: pool of system PTEs used to map system pages such as I/O space, kernel stacks, and memory descriptor lists.
  - **system working set lists**: the working set list data structures that describe the three system working sets (the system cache working set, the paged pool working set, and the system PTEs working set).
  - **system mapped views**: used to map Win32k.sys, the loadable kernel-mode part of the Windows subsystem, as well as kernel-mode graphics drivers it uses.
  - **hyperspace**: a special region used to map the process working set list and other per-process data that doesn’t need to be accessible in arbitrary process context. Hyperspace is also used to temporarily map physical pages into the system space. One example of this is invalidating page table entries in page tables of processes other than the current one (such as when a page is removed from the standby list).
  - **crash dump information**: reserved to record information about the state of a system crash.
  - **HAL** usage System memory reserved for HAL-specific structures.

### x86 Address Space Layouts

- x86 virtual address space layouts: <p align="center"><img src="https://i.imgur.com/eb4oAzh.png" width="450px" height="auto"></p>

### x86 System Address Space Layout

- 32-bit versions of Windows implement a **dynamic system address space** layout by using a virtual address allocator.
- many kernel-mode structures use dynamic address space allocation. These structures are therefore not necessarily **virtually contiguous** with themselves:
  - Nonpaged pool
  - Special pool
  - Paged pool
  - System page table entries (PTEs)
  - System mapped views
  - File system cache
  - File system structures (metadata)
  - Session space

### x86 session space

- for systems with **multiple** sessions, the code and data unique to each session are mapped into system
  address space but **shared** by the processes in that session. <p align="center"><img src="https://i.imgur.com/VmYnKuG.png" width="260px" height="auto"></p>
- you can list the active sessions with the `!session` command as follows:

```c
lkd> !session
Sessions on machine: 3
Valid Sessions: 0 1 3
Current Session 1
```

- you can set the active session using the `!session –s` command and display the address of the session data structures and the processes in that session with the `!sprocess` command:

```c
0: kd> !session -s 1
Sessions on machine: 2
Implicit process is now fffffa80`050acb00
.cache forcedecodeptes done
Using session 1l

0: kd> !sprocess
Dumping Session 1

_MM_SESSION_SPACE fffff88004553000
_MMSESSION        fffff88004553b40
PROCESS fffffa80050acb00
    SessionId: 1  Cid: 0198    Peb: 7fffffdd000  ParentCid: 0188
    DirBase: 93f80000  ObjectTable: fffff8a000e8ca20  HandleCount: 369.
    Image: csrss.exe

PROCESS fffffa80050e45c0
    SessionId: 1  Cid: 01d4    Peb: 7fffffdb000  ParentCid: 0188
    DirBase: a0806000  ObjectTable: fffff8a000ea7f90  HandleCount: 126.
    Image: winlogon.exe

PROCESS fffffa80054d2b00
    SessionId: 1  Cid: 0558    Peb: 7fffffd3000  ParentCid: 0208
    DirBase: 96537000  ObjectTable: fffff8a00b5a3e20  HandleCount: 220.
    Image: taskhost.exe

PROCESS fffffa8005567b00
    SessionId: 1  Cid: 060c    Peb: 7fffffdf000  ParentCid: 036c
    DirBase: 95260000  ObjectTable: fffff8a001f4b9c0  HandleCount:  74.
    Image: dwm.exe

PROCESS fffffa80055f2b00
    SessionId: 1  Cid: 0658    Peb: 7fffffd8000  ParentCid: 05fc
    DirBase: 9448e000  ObjectTable: fffff8a000f60d70  HandleCount: 756.
    Image: explorer.exe

PROCESS fffffa8005665b00
    SessionId: 1  Cid: 0730    Peb: 7fffffda000  ParentCid: 03b8
    DirBase: a7aea000  ObjectTable: fffff8a001fabce0  HandleCount:  85.
    Image: taskeng.exe

PROCESS fffffa800571bb00
    SessionId: 1  Cid: 0784    Peb: 7fffffd5000  ParentCid: 0658
    DirBase: 8f1e3000  ObjectTable: fffff8a001a4c0f0  HandleCount: 252.
    Image: vmtoolsd.exe
...
```

- to view the details of the session, dump the `MM_SESSION_SPACE` structure using the dt command, as follows:

```c
0: kd> dt nt!_MM_SESSION_SPACE fffff88004553000
   +0x000 ReferenceCount   : 0n17
   +0x004 u                : <unnamed-tag>
   +0x008 SessionId        : 1
   +0x00c ProcessReferenceToSession : 0n18
   +0x010 ProcessList      : _LIST_ENTRY [ 0xfffffa80`050acce0 - 0xfffffa80`05c45c10 ]
   +0x020 LastProcessSwappedOutTime : _LARGE_INTEGER 0x0
   +0x028 SessionPageDirectoryIndex : 0xa1704
   +0x030 NonPagablePages  : 0x72
   +0x038 CommittedPages   : 0x1761
   +0x040 PagedPoolStart   : 0xfffff900`c0000000 Void
   +0x048 PagedPoolEnd     : 0xfffff920`bfffffff Void
   +0x050 SessionObject    : 0xfffffa80`050a7390 Void
   +0x058 SessionObjectHandle : 0xffffffff`800001dc Void
   +0x060 ResidentProcessCount : 0n17
   +0x064 SessionPoolAllocationFailures : [4] 0
   +0x078 ImageList        : _LIST_ENTRY [ 0xfffffa80`050aa930 - 0xfffffa80`050b5480 ]
   +0x088 LocaleId         : 0x409
   +0x08c AttachCount      : 0
   +0x090 AttachGate       : _KGATE
   +0x0a8 WsListEntry      : _LIST_ENTRY [ 0xfffff800`02c7c3d0 - 0xfffff880`037bb0a8 ]
   +0x0c0 Lookaside        : [21] _GENERAL_LOOKASIDE
   +0xb40 Session          : _MMSESSION
   +0xb98 PagedPoolInfo    : _MM_PAGED_POOL_INFO
   +0xc00 Vm               : _MMSUPPORT
   +0xc90 Wsle             : 0xfffff900`00812488 _MMWSLE
   +0xc98 DriverUnload     : 0xfffff960`001b2d78     void  +fffff960001b2d78
   +0xcc0 PagedPool        : _POOL_DESCRIPTOR
   +0x1e00 PageDirectory    : _MMPTE
   +0x1e08 SessionVaLock    : _KGUARDED_MUTEX
   +0x1e40 DynamicVaBitMap  : _RTL_BITMAP
   +0x1e50 DynamicVaHint    : 0xf
   +0x1e58 SpecialPool      : _MI_SPECIAL_POOL
   +0x1ea0 SessionPteLock   : _KGUARDED_MUTEX
   +0x1ed8 PoolBigEntriesInUse : 0n263
   +0x1edc PagedPoolPdeCount : 0xc
   +0x1ee0 SpecialPoolPdeCount : 0
   +0x1ee4 DynamicSessionPdeCount : 0x1b
   +0x1ee8 SystemPteInfo    : _MI_SYSTEM_PTE_TYPE
   +0x1f30 PoolTrackTableExpansion : (null)
   +0x1f38 PoolTrackTableExpansionSize : 0
   +0x1f40 PoolTrackBigPages : 0xfffffa80`050ad000 Void
   +0x1f48 PoolTrackBigPagesSize : 0x200
   +0x1f50 IoState          : 6 ( IoSessionStateLoggedOn )
   +0x1f54 IoStateSequence  : 7
   +0x1f58 IoNotificationEvent : _KEVENT
   +0x1f70 CreateTime       : 0xbdfbd3d
   +0x1f78 CpuQuotaBlock    : (null)
```

- to view session space memory utilization:

```c
0: kd> !vm 4
Page File: \??\C:\pagefile.sys
  Current:   4193784 Kb  Free Space:   4181660 Kb
  Minimum:   4193784 Kb  Maximum:     12581352 Kb
...

Terminal Server Memory Usage By Session:

Session ID 0 @ fffff88002dfa000:
Paged Pool Usage:     3060 Kb
NonPaged Usage:        316 Kb
Commit Usage:         6484 Kb

Session ID 1 @ fffff88004536000:
Paged Pool Usage:        0 Kb
NonPaged Usage:        324 Kb
Commit Usage:          324 Kb
...
```

### System Page Table Entries

- are used to dynamically map system pages such as **I/O space, kernel stacks**, and the mapping for **memory descriptor lists**.
- to see how many system PTEs are available:

```c
0: kd> !sysptes
System PTE Information
Total System Ptes 307168
starting PTE: c0200000
free blocks: 32 total free: 3856 largest free block: 542

Kernel Stack PTE Information
Unable to get syspte index array - skipping bins
starting PTE: c0200000
free blocks: 165 total free: 1503 largest free block: 75
0: kd> ? nt!MiSystemPteInfo
Evaluate expression: -2100014016 = 82d45440
0: kd> dt _MI_SYSTEM_PTE_TYPE 82d45440
nt!_MI_SYSTEM_PTE_TYPE
+0x000 Bitmap : _RTL_BITMAP
+0x008 Flags : 3
+0x00c Hint : 0x2271f
+0x010 BasePte : 0xc0200000 _MMPTE
+0x014 FailureCount : 0x82d45468 -> 0
+0x018 Vm : 0x82d67300 _MMSUPPORT
+0x01c TotalSystemPtes : 0n7136
+0x020 TotalFreeSystemPtes : 0n4113
+0x024 CachedPteCount : 0n0
+0x028 PteFailures : 0
+0x02c SpinLock : 0
+0x02c GlobalMutex : (null)
```

- use `!sysptes 4` to show a list of allocators:

```c
lkd>!sysptes 4
0x1ca2 System PTEs allocated to mapping locked pages
VA MDL PageCount Caller/CallersCaller
ecbfdee8 f0ed0958 2 netbt!DispatchIoctls+0x56a/netbt!NbtDispatchDevCtrl+0xcd
f0a8d050 f0ed0510 1 netbt!DispatchIoctls+0x64e/netbt!NbtDispatchDevCtrl+0xcd
ecef5000 1 20 nt!MiFindContiguousMemory+0x63
ed447000 0 2 Ntfs!NtfsInitializeVcb+0x30e/Ntfs!NtfsInitializeDevice+0x95
ee1ce000 0 2 Ntfs!NtfsInitializeVcb+0x30e/Ntfs!NtfsInitializeDevice+0x95
ed9c4000 1 ca nt!MiFindContiguousMemory+0x63
eda8e000 1 ca nt!MiFindContiguousMemory+0x63
efb23d68 f8067888 2 mrxsmb!BowserMapUsersBuffer+0x28
efac5af4 f8b15b98 2 ndisuio!NdisuioRead+0x54/nt!NtReadFile+0x566
f0ac688c f848ff88 1 ndisuio!NdisuioRead+0x54/nt!NtReadFile+0x566
efac7b7c f82fc2a8 2 ndisuio!NdisuioRead+0x54/nt!NtReadFile+0x566
ee4d1000 1 38 nt!MiFindContiguousMemory+0x63
efa4f000 0 2 Ntfs!NtfsInitializeVcb+0x30e/Ntfs!NtfsInitializeDevice+0x95
efa53000 0 2 Ntfs!NtfsInitializeVcb+0x30e/Ntfs!NtfsInitializeDevice+0x95
eea89000 0 1 TDI!DllInitialize+0x4f/nt!MiResolveImageReferences+0x4bc
ee798000 1 20 VIDEOPRT!pVideoPortGetDeviceBase+0x1f1
f0676000 1 10 hal!HalpGrowMapBuffers+0x134/hal!HalpAllocateAdapterEx+0x1ff
f0b75000 1 1 cpqasm2+0x2af67/cpqasm2+0x7847
f0afa000 1 1 cpqasm2+0x2af67/cpqasm2+0x6d82
```

### 64-Bit Address Space Layouts

- theoretical 64-bit VAS is **16 exabytes**.
- unlike on x86 systems, where the default address space is divided in two parts, the 64-bit address is divided into a number of different size regions whose components match conceptually the portions of user, system, and session space:

| Region                | IA64 x64               |
| --------------------- | ---------------------- | ---------------------- |
| Process Address Space | 7,152 GB               | 8,192 GB               |
| System PTE Space      | 128 GB                 | 128 GB                 |
| System Cache          | 1 TB                   | 1 TB                   |
| Paged Pool            | 128 GB                 | 128 GB                 |
| Nonpaged Pool         | 75% of physical memory | 75% of physical memory |

- **large address space aware** apps running under **Wow64** will actually receive all **4** GB of user address space available.
- 64-bit applications linked **without** `/LARGEADDRESSAWARE` are constrained to the first 2 GB of the process VAS, just like 32-bit applications.
- The IA64 address space layout: <p align="center"><img src="https://i.imgur.com/G0ehzMF.png" height="auto"></p>
- x64 address space layout: <p align="center"><img src="https://i.imgur.com/0QCpmWT.png" height="auto"></p>

### x64 Virtual Addressing Limitations

- to simplify chip architecture and avoid unnecessary overhead, particularly in address translation, AMD’s and Intel’s current x64 processors implement only **256 TB** of VAS.
  - only the low-order 48 bits of a 64-bit virtual address are implemented.
  - the high-order 16 bits (bits 48 through 63) must be set to the same value as the highest order implemented bit (bit 47), in a manner similar to sign extension in two’s complement arithmetic.
  - an address that conforms to this rule is said to be a **canonical** address.

#### Windows x64 16-TB Limitation

- Windows on x64 has a further limitation: of the 256 TB of VAS available on x64 processors, Windows at present allows only the use of a little more than **16 TB**.
- this is split into two 8-TB regions:
  - a user mode, per-process region starting at 0 toward **0x000007FFFFFFFFFF**
  - a kernel-mode, systemwide region starting at “all Fs” and working toward **0xFFFFF80000000000** for most purposes.
- a number of Windows mechanisms have made, and continue to make, assumptions about usable bits in addresses:
  - **Pushlocks**, **fast references**, **Patchguard DPC contexts**, and **singly linked lists** are common examples of data structures that use bits within a pointer for nonaddressing purposes.
  - **singly linked lists**, combined with the lack of a CPU instruction in the original x64 CPUs required to “port” the data structure to 64-bit Windows, are responsible for this **memory addressing limit** on Windows for x64.
- kernel-mode data structures that do not involve SLISTs are not limited to the 8-TB address space range. System page table entries, hyperspace, and the cache working set all occupy virtual addresses below 0xFFFFF80000000000 because these structures do not use SLISTs.

#### Dynamic System Virtual Address Space Management

#### System Virtual Address Space Quotas

### User Address Space Layout

- just as address space in the kernel is dynamic, the user address space is also built dynamically:
  - addresses of the thread stacks, process heaps, and loaded images are dynamically computed (if its images support it) through a mechanism known as **Address Space Layout Randomization (ASLR)**.
- at the OS level, user address space is divided into a few well-defined regions of memory:
  - the **executable and DLLs** themselves are present as **memory mapped** image files,
  - followed by **the heap(s)** of the process and the **stack(s)** of its thread(s).
  - apart from these regions (and some reserved system structures such as the **TEBs** and **PEB**), all other memory allocations are run-time dependent and generated thanks to ASLR + DEP. <p align="center"><img src="https://i.imgur.com/BqhCCb0.png" height="auto"></p>

#### Image Randomization

- for executables, the load offset is calculated by computing a delta value each time an executable is loaded.
  - this value is a pseudo-random 8-bit number from **0x10000 to 0xFE0000**, calculated by taking the current processor’s time stamp counter (TSC), shifting it by four places, and then performing a division modulo 254 and adding 1 (so it can never load at the address in the PE header).
  - this number is then multiplied by the allocation granularity of 64 KB.
  - this delta is then **added** to the executable’s **preferred load address**, creating one of 256 possible locations within 16 MB of the image address in the PE header.
- for DLLs, computing the load offset begins with a **per-boot**, **systemwide** value called the **image bias**, which is computed by `MiInitializeRelocations` and stored in `MiImageBias`.
  - this value corresponds to the TSC of the current CPU when this function was called during the boot cycle, shifted and masked into an 8-bit value, which provides 256 possible values.
  - this value is computed only **once per boot** and **shared** across the system to allow DLLs to remain shared in physical memory and **relocated only once**.

#### Stack Randomization

- the next step in ASLR is to randomize the location of the initial thread’s stack (and, subsequently, of each new thread).
- enabled unless the flag `StackRandomizationDisabled` was enabled for the process.
- consists of first selecting one of 32 possible stack locations separated by either **64 KB** or **256 KB**.
- this base address is selected by finding the first appropriate free memory region and then choosing the *x*th available region, where _x_ is once again generated based on the current processor’s TSC shifted and masked into a 5-bit value (which allows for 32 possible locations).
- once this base address has been selected, a new TSC-derived value is calculated, this one 9 bits long. The value is then multiplied by 4 to maintain alignment, which means it can be as large as 2,048 bytes (half a page).
- tt is **added** to the **base address** to obtain the **final stack base**.

#### Heap Randomization

- ASLR randomizes the location of the initial process heap (and subsequent heaps) when created in user mode.
- the `RtlCreateHeap` function uses another pseudo-random, TSC-derived value to determine the base address of the heap.
- this value, 5 bits this time, is multiplied by **64 KB** to generate the final base address, starting at 0, giving a possible range of **0x00000000 to 0x001F0000** for the initial heap.
- he range **before the heap base address** is manually deallocated in an attempt to force an access violation if an attack is doing a brute-force sweep of the entire possible heap address range.

#### ASLR in Kernel Address Space

- ASLR is also active in kernel address space.
- there are **64** possible load addresses for 32-bit drivers and **256** for 64-bit drivers.
- relocating user-space images requires a significant amount of work area in kernel space, but if kernel space is tight, ASLR can use the user-mode address space of the System process for this work area.

## Address Translation

### x86 Virtual Address Translation

- each page of VAS is associated with a system-space structure called a **page table entry (PTE)**, which contains the physical address to which the virtual one is mapped.
- there may not even be **any PTEs** for regions that have been marked as **reserved or committed** but never **accessed**, because the page table itself might be allocated only when the first page fault occurs.
- non-PAE x86 systems use a **two-level** page table structure to translate virtual to physical addresses. A 32-bit virtual address mapped by a normal 4-KB page is interpreted as two fields:
  - the **virtual page number** and the byte within the page, called the **byte offset**.
  - the virtual page number is further divided into two subfields, called the **page directory index** and the **page table index**:
  <p align="center"><img src="https://i.imgur.com/gyAWAAx.png"  height="auto"></p>
- the format of a physical address on an x86 non-PAE system is: <p align="center"><img src="https://i.imgur.com/K6xybX9.png"  height="auto"></p>
- the byte offset does not participate in, and does not change as a result of, address translation. It is simply copied from the virtual address to the physical address.
- translating a valid virtual address (x86 non-PAE): <p align="center"><img src="https://i.imgur.com/4RBL2h7.png"  height="auto"></p>
- page tables can be **paged out** or **not yet created**, and in those cases, the page table is first made resident before proceeding. If a flag in the PDE indicates that it describes a **large page**, then it simply contains the PFN of the target large page, and the rest of the virtual address is treated as the byte offset within the large page.
- if the PTE’s valid bit is **clear**, this triggers a page fault. The OS’s memory management fault handler (pager) locates the page and tries to make it valid.

#### Page Directories

- on non-PAE x86 systems, each process has a **single page directory**.
- physical address of the process page directory is stored in the `KPROCESS` block.
  - but it is also mapped virtually at address _0xC0300000_.
- CPU obtains the location of the page directory from a privileged CPU register called **CR3**.
- is page aligned.
- is composed of **page directory entries (PDEs)**, each of which is 4 bytes long.
- page tables are created **on demand**, so the page directory for most processes points only to a small set of page tables.
- the physical address of the currently running process’s page directory:

```c
lkd> !process -1 0
PROCESS 857b3528 SessionId: 1 Cid: 0f70 Peb: 7ffdf000 ParentCid: 0818
DirBase: 47c9b000 ObjectTable: b4c56c48 HandleCount: 226.
Image: windbg.exe
```

- to see the page directory’s virtual address for the PTE of a particular virtual address:

```c
lkd> !pte 10004
VA 00010004
PDE at C0300000 PTE at C0000040
contains 6F06B867 contains 3EF8C847
pfn 6f06b ---DA--UWEV pfn 3ef8c ---D---UWEV
```

- the page tables that describe system space are **shared among all processes**.
- and session space is shared only among processes in a session.
- to avoid having multiple page tables describing the same virtual memory, when a process is created, the page directory entries that describe system space are **initialized to point to the existing** system page tables.
- if the process is part of a session, **session space page tables** are also **shared** by pointing the session space page directory entries to the existing session page tables.

#### Page Tables and Page Table Entries

- valid PTEs have two main fields: the page frame number (PFN) of the physical page containing the data or of the physical address of a page in memory, and some flags that describe the state and protection of the page: <p align="center"><img src="https://i.imgur.com/ebUANud.png"  height="auto"></p>
- PTE Status and Protection Bits:

| Name of Bit    | Meaning                                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------------------------ |
| Accessed       | Page has been accessed.                                                                                            |
| Cache disabled | Disables CPU caching for that page.                                                                                |
| Copy-on-write  | Page is using copy-on-write                                                                                        |
| Dirty          | Page has been written to.                                                                                          |
| Global         | Translation applies to all processes. (For example, a translation buffer flush won’t affect this PTE.)             |
| Large page     | Indicates that the PDE maps a 4-MB page (or 2 MB on PAE systems).                                                  |
| Owner          | Indicates whether user-mode code can access the page or whether the page is limited to kernel-mode access.         |
| Prototype      | The PTE is a prototype PTE, which is used as a template to describe shared memory associated with section objects. |
| Valid          | Indicates whether the translation maps to a page in physical memory.                                               |
| Write through  | Marks the page as write-through or (if the processor supports the page attribute table)                            |
| write-combined | This is typically used to map video frame buffer memory.                                                           |
| Write          | Indicates to the MMU whether the page is writable.                                                                 |

# Linux Kernel OS Assignments

A collection of three Linux kernel patches written as part of the 2025 Winter Semester K22 : Operating Systems course , at National and Kapodistrian University of Athens. The assignments were implemented in cooperation with [Adamantia Malioti-Thermou](https://github.com/dendrouj). Each assignment implements a non-trivial kernel feature from scratch — spanning system calls, process scheduling, and memory management.

---

## Assignment 1 — `k22tree`: Process Tree System Call

**Patch:** `hmwk1.patch`

### Overview

This patch adds a new Linux system call, `sys_k22tree` (syscall 467), that lets a user-space program snapshot the entire process tree of the running kernel in a single call.

### What it does

The syscall takes a user-space buffer and a capacity integer (`ne`). It walks the kernel's process tree using an explicit DFS stack (iterative, not recursive, to be safe in kernel context), visits every process group leader in pre-order, fills in a `struct k22info` entry for each one, and copies the results back to user space. On return, `ne` is overwritten with the number of entries actually written, and the syscall's return value is the total number of processes in the system — even if the buffer was too small to hold all of them.

Each `struct k22info` entry (defined in `include/uapi/linux/k22info.h`) captures:

- Executable name (`comm`)
- PID, parent PID (resolved to the thread group leader to handle threaded processes correctly)
- First child PID and next sibling PID (for reconstructing the tree client-side)
- Voluntary and involuntary context switch counts (`nvcsw`, `nivcsw`)
- Monotonic start time in nanoseconds

### Key design decisions

- The traversal holds `tasklist_lock` for reading throughout the walk, which is the standard kernel mechanism for safely iterating over `task_struct` entries.
- Threads are intentionally skipped during output but their children are still pushed onto the DFS stack, so child processes of threaded parents appear correctly.
- The parent is reported as the thread group leader rather than the raw `real_parent`, ensuring a consistent tree even when a child was spawned from a non-leader thread.
- A kernel-side buffer (`kbuf`) accumulates results before a single `copy_to_user` call, minimising the time spent crossing the user/kernel boundary.
- The stack and buffer are allocated with `kmalloc_array` sized to `2× estimated_processes + 1000` to give headroom for processes that spawn between the count pass and the walk pass.

---

## Assignment 2 — `SCHED_GRR`: Group Round-Robin Scheduler
**Note:** The patch for this assignment is not publicly available at the professor's
request, to preserve the assignment integrity for future students.
The description below documents the design and implementation for reference. 
This assignment received a grade of 100%.

### Overview

This patch implements a brand-new Linux scheduling class called **GRR (Group Round-Robin)**, registered as `SCHED_GRR` (policy 8). It sits between the real-time (`SCHED_RR`/`SCHED_FIFO`) classes and the fair (`SCHED_CFS`) class in the scheduler hierarchy, and becomes the default policy for the entire system when compiled in.

### What it does

GRR maintains two scheduling groups:

- `GRR_DEFAULT` — for general-purpose tasks
- `GRR_PERFORMANCE` — for latency-sensitive or high-priority tasks

Each group has a dedicated set of CPU cores assigned to it at runtime. Within each group, tasks are scheduled in a standard round-robin fashion with a fixed time slice.

Two new system calls are added to manage the groups from user space:

- `sched_assign_ncores_to_group(int ncores, int group)` (syscall 467) — reassigns a given number of CPU cores to the specified group, rebalancing the CPU allocation between `GRR_DEFAULT` and `GRR_PERFORMANCE` dynamically.
- `sched_assign_process_to_group(pid_t pid, int group)` (syscall 468) — moves a process into a scheduling group.

### Key design decisions

- A new `sched_grr_entity` is embedded in every `task_struct` (behind a `CONFIG_GRR_SCHED` guard), tracking per-task fields such as the group membership, time used in the current slice, total and previous runtime, and (on SMP) the last CPU the task ran on and the last time it was load-balanced.
- The scheduler class (`grr_sched_class`) is placed above `fair_sched_class` and below `rt_sched_class` in the class chain, which is verified with `BUG_ON` assertions at boot.
- SMP load balancing is implemented as a periodic hook (`periodic_load_balance_grr`) called from the scheduler tick, which redistributes tasks across the cores assigned to a group when imbalances are detected.
- Kthreads are automatically placed under `SCHED_GRR` when the config is enabled, making GRR truly system-wide.
- The CPU-to-group allocation is tracked in a dedicated `grr_cpu_allocation` structure, and reassignment takes effect immediately without requiring a full scheduler restart.

---

## Assignment 3 — `/dev/ept`: Expose Page Tables Device

**Patch:** `hmwk3.patch`

### Overview

This patch adds a character device `/dev/ept` (Expose Page Tables) that allows a privileged user-space process to `mmap` a read-only view of its own page tables. It is implemented as a misc device backed by a custom VM fault handler in a new file `mm/ept.c`.

### What it does

When user space `mmap`s `/dev/ept`, it gets a virtual address range that acts as a window into the process's PTE (Page Table Entry) pages. Accessing `ept[N]` yields the raw 4 KB page of PTEs that covers virtual page `N`. If that part of the address space has no PTE table yet (the mapping is absent or uses a huge page), the fault handler returns the zero page so accesses never segfault.

This makes it possible for a privileged tool to inspect the physical address mappings of any virtual address without going through `/proc/pid/pagemap` or other indirect interfaces.

### Key design decisions

- The fault handler (`ept_fault`) computes the target virtual address from `vmf->pgoff`, walks the four-level page table hierarchy (`pgd → p4d → pud → pmd → pte page`), and uses `vmf_insert_pfn` to insert the PTE page's PFN directly into the user VMA, so the kernel's own page tables are shared (read-only) with user space.
- Huge pages and transparent huge pages are explicitly rejected by the fault handler since they have no separate PTE-level page to expose.
- The VMA is flagged `VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP | VM_IO`, and `VM_MAYWRITE` is cleared to prevent copy-on-write issues that would trigger a `BUG_ON` inside `vmf_insert_pfn`.
- Access is restricted to `CAP_SYS_ADMIN` (checked in both `ept_open` and `ept_mmap`), and write mappings are rejected outright.
- To prevent stale EPT entries after the underlying page tables change, `ept_invalidate_mapping` is called from two hook points: `free_pte_range` (when a PTE page is freed) and `pmd_install` (when a new PTE page is installed). Invalidation happens *after* releasing the page table spinlock to avoid deadlocking with `zap_page_range_single`, which also acquires that lock.
- The `pte_alloc` macro and `__pte_alloc` / `pmd_install` functions are extended to accept a `unsigned long address` parameter throughout the kernel (touching `mm/memory.c`, `mm/gup.c`, `mm/mremap.c`, `mm/pagewalk.c`, `mm/userfaultfd.c`, and related headers) so that invalidation can be triggered at the correct granularity.

---

## Repository structure

```
hmwk1.patch   # Assignment 1: k22tree system call
hmwk2.patch   # Is intentionally omitted at the professor's request
hmwk3.patch   # Assignment 3: /dev/ept page table device
```

Each patch targets **Linux 6.14**.



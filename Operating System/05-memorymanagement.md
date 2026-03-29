## 1. The Problem: Physical Memory Is a Precious, Shared Resource

Imagine a hotel with a set number of rooms (physical memory). Several tour groups (processes) want to stay simultaneously. Each group thinks it has the entire hotel to itself (virtual address space), but in reality they must share the limited rooms. The **OS is the concierge** that:

- **Isolates** groups – one group cannot enter another’s room.
- **Gives the illusion** of infinite rooms (virtual memory larger than physical).
- **Efficiently uses** rooms by swapping groups in and out.

Without memory management, processes would directly access physical memory, leading to:

- **Corruption** (one process overwriting another’s data).
- **Wasted space** (each process would need a fixed chunk, even if it doesn’t use it all).
- **Complexity** (programmers would have to manage physical addresses).

---

## 2. Core Concepts: Physical vs Virtual Addresses

- **Physical address**: The actual location in RAM hardware (e.g., `0x7f3a1000`).
- **Virtual address**: The address seen by a process (e.g., `0x00401000`). Each process has its own **virtual address space**, starting at 0.

The OS, with hardware help (MMU – Memory Management Unit), translates every virtual address to a physical address on the fly.

**Analogy**: In a hotel, each tour group receives a map (virtual address space) that shows “your room is suite 100, 101, …”. The concierge (MMU) translates “suite 100” to actual room 302 on the third floor.

---

## 3. Paging: The Foundation

### 3.1 Pages and Frames

- **Page**: A fixed-size block of **virtual memory** (e.g., 4 KB).
- **Frame**: A fixed-size block of **physical memory** of the same size.

The virtual address space is divided into pages. The physical memory is divided into frames. The OS maintains a **page table** per process that maps each virtual page to a physical frame (or marks it as not present).

### 3.2 Address Translation (Simple)

A virtual address consists of two parts:

- **Page number** (high bits) – index into page table.
- **Offset** (low bits) – offset within the page.

**Example**: 32-bit address, 4 KB page size → 12 bits for offset, 20 bits for page number.

Translation:

1. Extract page number from virtual address.
2. Look up page table entry (PTE) for that page number.
3. If present, the PTE gives the physical frame number.
4. Physical address = (frame number << page shift) + offset.

---

## 4. Page Table Structure

A simple page table is an array of **Page Table Entries (PTEs)** indexed by virtual page number. Each PTE contains:

- **Frame number** (if page is in memory)
- **Present bit** (1 if page is in physical memory)
- **Protection bits** (read/write/execute)
- **Referenced, dirty bits** (for page replacement)
- **Other flags** (e.g., for caching)

**Analogy**: The page table is like the **hotel’s room assignment ledger** – each tour group (process) has its own ledger mapping their “suite numbers” (virtual pages) to actual rooms (frames).

### 4.1 Multi-Level Page Tables

A flat page table for a 64-bit address space would be impossibly large. So modern OSes use **multi-level page tables** (e.g., 4-level on x86-64).

- Only the levels that are actually used are allocated.
- Each level is an array of pointers to the next level.

**Why?** Saves memory and allows sparse mappings.

---

## 5. Virtual Memory: More Than Just Mapping

Virtual memory enables several powerful features:

### 5.1 Demand Paging

Pages are loaded into physical memory **only when needed** (on a page fault). This reduces memory usage and speeds up process startup.

- A **page fault** occurs when a process accesses a virtual page whose PTE has the present bit = 0.
- The OS handles the fault:
  1. Finds a free physical frame (or evicts one using a replacement algorithm).
  2. Loads the page from disk (swap area or executable file).
  3. Updates the PTE with the frame number and sets present bit.
  4. Restarts the instruction.

**Analogy**: You only bring your luggage (page) from the storage room (disk) to your hotel room when you actually need it.

### 5.2 Swapping

When physical memory is full, the OS can **swap out** some pages to disk (swap space) to free frames.

### 5.3 Memory Protection

The OS sets protection bits per page. If a process tries to write to a read‑only page, the hardware triggers a protection fault → OS sends `SIGSEGV` and kills the process.

### 5.4 Sharing

Multiple processes can map the same physical frame into their virtual address spaces (e.g., shared libraries, memory‑mapped files). This saves memory and enables inter‑process communication.

**Example**: `libc.so` is loaded once into physical memory, but every process sees it in its own virtual address space.

### 5.5 Copy‑on‑Write (CoW)

When a process `fork()`s, instead of copying the entire address space, the OS shares the parent’s pages as **read‑only**. When either process tries to write, a page fault occurs, and the OS copies the page (makes it private). This is efficient and is used in Linux’s `fork()`.

---

## 6. Hardware Support: TLB (Translation Lookaside Buffer)

Page table lookups are frequent – every memory access would require multiple memory accesses (for multi‑level page tables), which is slow.  
The **TLB** is a hardware cache that stores recent virtual‑to‑physical translations.

- On a memory access, the CPU first checks the TLB.
- **TLB hit**: translation is done in 1 cycle.
- **TLB miss**: the CPU walks the page table in hardware (or via software, depending on architecture) and loads the translation into TLB.

**Analogy**: The concierge keeps a small notebook (TLB) of recently used room assignments, so they don’t have to flip through the big ledger each time.

---

## 7. Code Examples: Simulating Page Table Lookup

Let’s build a tiny in‑memory simulation of a page table and address translation.  
We’ll use Python to make it clear.

```python
# Simulate a simple page table (flat)
# Each page table entry: (frame_number, present, read_write)

PAGE_SIZE = 4096  # 4 KB
NUM_PAGES = 256   # virtual address space 1 MB (256 * 4KB)

class SimplePageTable:
    def __init__(self):
        # Initialize all entries as not present
        self.entries = [None] * NUM_PAGES

    def map_page(self, virtual_page, frame, writable=True):
        # Mark the page as present and mapped to a physical frame
        self.entries[virtual_page] = (frame, True, writable)

    def translate(self, virtual_address):
        page_number = virtual_address // PAGE_SIZE
        offset = virtual_address % PAGE_SIZE
        if page_number >= NUM_PAGES:
            raise ValueError("Address out of range")
        entry = self.entries[page_number]
        if entry is None:
            raise PageFault("Page not present")
        frame, present, writable = entry
        if not present:
            raise PageFault("Page not present")
        physical_address = (frame << 12) + offset  # page shift = 12 bits
        return physical_address, writable

class PageFault(Exception):
    pass

# Example usage
pt = SimplePageTable()
pt.map_page(0, 5)   # virtual page 0 mapped to physical frame 5
pt.map_page(1, 10)  # virtual page 1 mapped to physical frame 10

# Translate address 4096 (which is page 1, offset 0)
virt_addr = 4096
phys_addr, writable = pt.translate(virt_addr)
print(f"Virtual {hex(virt_addr)} -> Physical {hex(phys_addr)}")

# Access a non‑mapped page
try:
    pt.translate(8192)  # page 2, not mapped
except PageFault as e:
    print("Page fault:", e)
```

This simulation shows the core of address translation. In a real OS, the page table is stored in kernel memory, and the MMU does the lookup.

---

## 8. A More Realistic Simulation: Multi‑Level Page Table

We can also simulate a two‑level page table to understand how memory is saved.

```python
# Two-level page table: 10 bits for page directory, 10 bits for page table, 12 bits offset
# Virtual address: [10 bits PD index] [10 bits PT index] [12 bits offset]

PAGE_SIZE = 4096
NUM_PT_ENTRIES = 1024  # 10 bits
NUM_PD_ENTRIES = 1024

class TwoLevelPageTable:
    def __init__(self):
        # Page directory: each entry points to a page table (list) or None
        self.page_directory = [None] * NUM_PD_ENTRIES

    def map_page(self, virtual_address, frame):
        # Extract indices
        addr = virtual_address
        pd_idx = (addr >> 22) & 0x3FF      # bits 22-31
        pt_idx = (addr >> 12) & 0x3FF      # bits 12-21
        # Ensure page table exists
        if self.page_directory[pd_idx] is None:
            self.page_directory[pd_idx] = [None] * NUM_PT_ENTRIES
        # Set the page table entry
        self.page_directory[pd_idx][pt_idx] = frame

    def translate(self, virtual_address):
        pd_idx = (virtual_address >> 22) & 0x3FF
        pt_idx = (virtual_address >> 12) & 0x3FF
        offset = virtual_address & 0xFFF
        pt = self.page_directory[pd_idx]
        if pt is None or pt[pt_idx] is None:
            raise PageFault("Page not present")
        frame = pt[pt_idx]
        return (frame << 12) + offset

# Example
tlb_sim = TwoLevelPageTable()
tlb_sim.map_page(0x00400000, 5)   # address 4MB, page 0x4000? Actually 4MB = 0x400000
print(hex(tlb_sim.translate(0x00401000)))  # should be 0x5000? Actually frame 5 at 0x5000 plus offset
```

---

## 9. Page Replacement Algorithms

When physical memory is full and a page fault occurs, the OS must evict a page to make room. Common algorithms:

- **FIFO** – simple, but can evict frequently used pages.
- **LRU (Least Recently Used)** – evict the page that hasn’t been used for the longest time. Hard to implement exactly in hardware, but approximated.
- **Clock (Second Chance)** – a circular list with a “referenced” bit; gives a page a second chance.
- **NRU (Not Recently Used)** – uses referenced and dirty bits.

**Analogy**: The hotel concierge, when all rooms are full, must ask a guest to move to storage. He picks the guest who has stayed the longest without using the room (LRU) or uses a round‑robin (Clock).

---

## 10. Performance Considerations

- **TLB miss cost**: can be hundreds of cycles because it triggers a page table walk (multi‑level).
- **Page fault cost**: thousands to millions of cycles (disk I/O).
- **Large pages** (e.g., 2 MB or 1 GB) reduce TLB misses and page table overhead, but increase internal fragmentation.

Modern OSes use **huge pages** for performance‑critical applications (databases, VMs).

---

## 11. Memory‑Mapped Files

Virtual memory allows files to be mapped directly into a process’s address space. The file becomes a virtual memory region; reads/writes to that region are reflected in the file (with proper flushing).

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

void* map = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
// Now accessing map[i] reads/writes the file.
```

This is extremely efficient because the OS handles paging the file content in and out automatically.

---

## 12. Common Pitfalls & Misconceptions

- **“Virtual memory is just swapping to disk”** – No, swapping is one part; virtual memory also provides isolation and abstraction.
- **“More RAM means no page faults”** – Even with huge RAM, page faults occur for demand paging of executable code and memory‑mapped files.
- **“Page tables are stored in MMU”** – No, page tables are in main memory; the MMU only caches them (TLB).
- **“TLB miss always causes page fault”** – No, TLB miss triggers a hardware page table walk; only if the PTE says not present does a page fault occur.

---

## 13. Summary

- **Memory management** using paging and virtual memory is the cornerstone of modern OSes.
- **Virtual addresses** give each process its own isolated address space, while **paging** maps those addresses to physical frames via **page tables**.
- **Demand paging** loads pages on demand, enabling efficient memory usage and the illusion of more memory than physically available.
- **TLB** accelerates translation by caching recent mappings.
- **Multi‑level page tables** make large address spaces feasible.
- **Page replacement algorithms** decide which pages to evict when memory is full.

_“Virtual memory is the great illusionist of the OS – it makes every process feel like the sole owner of a vast memory palace, while the kernel quietly juggles the limited physical resources.”_

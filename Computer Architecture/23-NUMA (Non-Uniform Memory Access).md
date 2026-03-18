NUMA (Non-Uniform Memory Access) is the architecture of choice for modern multi-socket systems, where each processor has its own local memory, but all processors can access the memory of other processors via a shared interconnect. This design scales beyond the limits of a single shared memory bus, but introduces non-uniformity: accessing local memory is faster than remote memory. As a software engineer, understanding NUMA is essential for optimizing performance on large servers, databases, and high-performance computing applications.

---

## 1. Why NUMA? The Limits of SMP

In a traditional Symmetric Multiprocessing (SMP) system, all CPUs share a single memory bus and a single memory controller. As the number of cores increases, the bus becomes a bottleneck, and contention grows. Adding more cores saturates the bus, limiting scalability.

NUMA solves this by giving each processor (or group of cores) its own local memory and a dedicated memory controller. Processors are connected via a high-speed interconnect (e.g., AMD's Infinity Fabric, Intel's Ultra Path Interconnect - UPI, or ARM's CCIX). Each processor and its local memory form a **NUMA node**. A CPU can access its local memory quickly, while accessing memory from another node (remote) is slower because it must traverse the interconnect.

---

## 2. NUMA Architecture Overview

### 2.1 Basic Components

- **NUMA node**: A set of CPUs (cores) and a portion of physical memory that is directly attached to them.
- **Memory controller**: Integrated into each CPU, manages local DRAM.
- **Interconnect**: High-speed links between nodes for remote memory access.
- **Memory hierarchy**: Each node has its own memory (e.g., DDR4), and possibly multiple levels of cache (L1/L2/L3 shared among cores on that node).

### 2.2 Latency and Bandwidth

Access times vary significantly:

- **Local access**: ~50-100 ns (typical DRAM latency).
- **Remote access**: 1.5x to 3x higher latency (depending on interconnect and distance, e.g., 2-hop remote). Bandwidth may also be lower due to interconnect sharing.

Thus, to maximize performance, threads should primarily access memory that is local to the node they run on.

### 2.3 Example: Two-Socket NUMA System

```
Node 0                Node 1
+-------+             +-------+
| CPU 0 |<---------->| CPU 1 |
+-------+  Interconnect +-------+
   |                        |
Local Memory 0          Local Memory 1
```

CPU 0 accessing memory in Node 0 is fast; accessing memory in Node 1 requires going through the interconnect, incurring higher latency.

---

## 3. Operating System Support for NUMA

Modern operating systems are NUMA-aware. They provide mechanisms to:

- Discover the NUMA topology (which CPUs belong to which nodes, memory ranges of each node).
- Schedule threads on CPUs close to the memory they use.
- Allocate memory from a specific node (or with policies like "allocate on the node where the thread runs").

### 3.1 Linux NUMA Features

- **NUMA policies**: `MPOL_DEFAULT`, `MPOL_BIND`, `MPOL_PREFERRED`, `MPOL_INTERLEAVE`.
- **System calls**: `mbind`, `set_mempolicy`, `get_mempolicy` for controlling memory allocation.
- **`numactl`**: Command-line tool to launch processes with specific NUMA policies and bindings.
- **`libnuma`**: Library for programmatic control.
- **`hwloc`**: Portable abstraction for hardware topology (including NUMA).

### 3.2 First-Touch Policy

A key concept: on most systems, memory pages are allocated on the node where the first access (touch) occurs. This is the **first-touch policy**. It means that to have memory local to a thread, that thread (or a thread on the same node) must initialize the memory. If a thread on Node 0 allocates memory but a thread on Node 1 first writes to it, the page will be allocated on Node 1, causing remote accesses later.

---

## 4. Programming for NUMA

### 4.1 Discovering Topology with `numactl`

Before writing code, you can inspect your system's NUMA layout:

```bash
numactl --hardware
```

Example output:

```
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32768 MB
node 0 free: 12345 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32768 MB
node 1 free: 23456 MB
node distances:
node   0   1
  0:  10  20
  1:  20  10
```

The distances show relative latency (10 is local, 20 is remote).

### 4.2 Binding Processes with `numactl`

To run a process on a specific node (both CPU and memory binding):

```bash
numactl --cpunodebind=0 --membind=0 ./my_program
```

To interleave memory across nodes (useful for some workloads):

```bash
numactl --interleave=all ./my_program
```

### 4.3 Programmatic Control with `libnuma`

`libnuma` provides functions to set memory policies and CPU affinity.

**Example: Allocate memory on a specific node and bind thread to that node**

```c
#include <stdio.h>
#include <stdlib.h>
#include <numa.h>
#include <numaif.h>
#include <sched.h>
#include <unistd.h>

int main() {
    // Check if NUMA is available
    if (numa_available() < 0) {
        printf("NUMA not supported\n");
        return 1;
    }

    int node = 0;  // we want node 0
    size_t size = 1024 * 1024 * 100; // 100 MB

    // Allocate memory using mmap (malloc may not respect policy for huge pages)
    void *ptr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // Bind the memory to node 0 using mbind
    unsigned long nodemask = 1UL << node; // bits for nodes
    if (mbind(ptr, size, MPOL_BIND, &nodemask, sizeof(nodemask)*8, 0) != 0) {
        perror("mbind");
        munmap(ptr, size);
        return 1;
    }

    // Now first-touch will allocate on node 0
    // Touch the memory to force allocation
    for (size_t i = 0; i < size; i += 4096) {
        ((char*)ptr)[i] = 0;
    }

    // Bind current thread to CPUs on node 0
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    struct bitmask *cpus = numa_allocate_cpumask();
    numa_node_to_cpus(node, cpus); // get CPUs of node 0
    for (int i = 0; i < cpus->size; i++) {
        if (numa_bitmask_isbitset(cpus, i)) {
            CPU_SET(i, &cpuset);
        }
    }
    numa_free_cpumask(cpus);

    if (sched_setaffinity(0, sizeof(cpuset), &cpuset) != 0) {
        perror("sched_setaffinity");
    }

    printf("Thread running on node %d, memory allocated on node %d\n",
           numa_node_of_cpu(sched_getcpu()),
           numa_node_of_addr(ptr)); // needs libnuma function? not standard; but we know

    // ... work ...

    munmap(ptr, size);
    return 0;
}
```

Compile with `-lnuma`.

### 4.4 Using `hwloc` for Portable Topology

`hwloc` (Portable Hardware Locality) provides a cross-platform API to discover NUMA topology and bind threads/memory.

**Example: hwloc basics**

```c
#include <hwloc.h>
#include <stdio.h>

int main() {
    hwloc_topology_t topology;
    hwloc_topology_init(&topology);
    hwloc_topology_load(topology);

    int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
    if (depth == HWLOC_TYPE_DEPTH_UNKNOWN) {
        printf("No NUMA nodes\n");
        return 1;
    }

    // Iterate over NUMA nodes
    hwloc_obj_t obj = NULL;
    while ((obj = hwloc_get_next_obj_by_depth(topology, depth, obj)) != NULL) {
        printf("NUMA node %d: CPUs:", obj->logical_index);
        hwloc_bitmap_t cpuset = obj->cpuset;
        // Print CPUs
        for (unsigned i = 0; i < hwloc_bitmap_last(cpuset); i++) {
            if (hwloc_bitmap_isset(cpuset, i)) printf(" %d", i);
        }
        printf("\n");
    }

    // Bind current thread to a specific node
    hwloc_obj_t node = hwloc_get_obj_by_depth(topology, depth, 0); // node 0
    hwloc_set_cpubind(topology, node->cpuset, HWLOC_CPUBIND_THREAD);

    hwloc_topology_destroy(topology);
    return 0;
}
```

Compile with `-lhwloc`.

---

## 5. Optimization Strategies for NUMA

### 5.1 Thread and Data Placement

- **Bind threads to CPUs on the same node** that will access the data they work on.
- **Allocate memory on the same node** as the thread that will primarily use it.
- Use **first-touch** to your advantage: initialize data in parallel, with each thread initializing the portion that will be local to it.

**Example: Parallel initialization with first-touch**

```c
#include <omp.h>
#include <stdlib.h>

int main() {
    size_t N = 100000000;
    double *data = (double*)malloc(N * sizeof(double));
    if (!data) return 1;

    #pragma omp parallel for
    for (size_t i = 0; i < N; i++) {
        data[i] = 0.0;  // first touch; each page is allocated on the node of the touching thread
    }

    // Now each page is local to the thread that initialized it.
    // Subsequent parallel loops will have good locality if threads are bound to same CPUs.
    // But careful: thread affinity must be consistent.
    return 0;
}
```

With OpenMP, you can set thread affinity using environment variables (`OMP_PLACES=cores`, `OMP_PROC_BIND=true`).

### 5.2 Avoiding Remote Memory Access

- **Data partitioning**: Split data so that each thread works on a contiguous chunk that fits in its local memory.
- **Thread migration**: If a thread migrates to a different node, its memory remains on the original node, leading to remote accesses. Pin threads to specific cores to prevent migration.
- **Use NUMA-aware allocators**: Some allocators (e.g., `tcmalloc`, `jemalloc`) can be configured to be NUMA-aware.

### 5.3 Interleaving for Bandwidth

For some workloads where all threads access the entire dataset uniformly (e.g., some scientific simulations), interleaving memory across nodes can improve bandwidth by utilizing multiple memory controllers. Use `numactl --interleave=all` or `MPOL_INTERLEAVE`.

### 5.4 Detecting and Measuring NUMA Effects

Use tools like `perf` to count remote memory accesses:

```bash
perf stat -e remote_accesses,remote_accesses_uncore ./my_program
```

Also, `numastat` shows memory allocation per node.

---

## 6. Pitfalls and Common Mistakes

- **False sharing across nodes**: If two threads on different nodes modify variables in the same cache line, the line will bounce across the interconnect, causing severe slowdowns. Align data to cache lines and separate thread-private data.
- **Allocating memory with `malloc` after binding**: `malloc` may allocate from any node; use `mbind` or `numa_alloc_onnode` to enforce node placement.
- **Thread migration**: OS may move threads between nodes. Use `sched_setaffinity` to pin.
- **First-touch not honored** if memory is allocated with `MAP_POPULATE` or by a different thread. Ensure the thread that will use the data is the one that touches it first.

---

## 7. Case Study: Database Optimization

A database server with multiple NUMA nodes:

- Partition the database buffer pool into per-node regions.
- Pin worker threads to specific nodes.
- Route client connections to threads on the same node as the data they need (if possible).
- Use NUMA-aware memory allocation for query execution structures.

This can dramatically reduce remote memory traffic and improve throughput.

---

## 8. Conclusion

NUMA is a scalable architecture that brings both opportunities and challenges. By understanding its non-uniform nature and using OS-provided tools and APIs, you can place threads and memory optimally to maximize performance. Key takeaways:

- Know your system's NUMA topology.
- Bind threads to CPUs and allocate memory on the same node.
- Use first-touch to initialize data where it will be used.
- Avoid remote memory access and false sharing.
- Measure and tune using `perf`, `numastat`, and `numactl`.

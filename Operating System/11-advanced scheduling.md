## 1. Beyond Basic Scheduling

Basic scheduling (FCFS, SJF, RR) works for simple scenarios, but modern systems demand:

- **Multi‑core** – scheduling across multiple CPUs (SMP).
- **Real‑time** – guaranteeing that time‑critical tasks meet deadlines.
- **Energy efficiency** – minimising power consumption on mobile and embedded devices.
- **Fairness** – ensuring that interactive tasks remain responsive while batch jobs progress.
- **Quality of Service (QoS)** – isolating workloads in containerised environments.

Advanced schedulers address these with sophisticated data structures, heuristics, and hardware awareness.

---

## 2. Multi‑core Scheduling (SMP)

In a Symmetric Multi‑Processing (SMP) system, all cores are equal. The scheduler must distribute threads across cores.

### 2.1 Global vs. Per‑Core Schedulers

- **Global scheduler**: one queue of runnable threads; on each core, the scheduler picks the next thread. Simple but suffers from contention on the queue and poor cache affinity.
- **Per‑core scheduler**: each core has its own run queue. Load balancing moves threads between cores to keep workloads even.

Modern OSes use per‑core schedulers with load balancing.

### 2.2 Affinity and Cache Considerations

**Processor affinity** binds a thread to a specific core (or set of cores). This improves cache hit rates (the thread’s data stays in that core’s cache). The OS automatically tries to keep a thread on the same core after it runs (soft affinity).

**Hard affinity** can be set via `sched_setaffinity()` in Linux:

```c
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(2, &set);          // bind to core 2
sched_setaffinity(0, sizeof(set), &set);
```

### 2.3 Load Balancing

The scheduler periodically checks core loads and **pushes** or **pulls** threads to balance. In Linux CFS, this is done by a **load balancer** running on each core, triggered by timers and idle events.

**Balancing algorithms**:

- **Push**: when a core is idle, it pulls work from busy cores.
- **Pull**: busy cores can push threads to idle cores.
- **Steal**: idle cores actively “steal” threads from other run queues.

---

## 3. Real‑Time Scheduling

Real‑time systems must meet **deadlines**. Schedulers for real‑time are often **priority‑based** and **preemptive**.

### 3.1 Fixed‑Priority Scheduling

Each thread has a static priority. The highest‑priority ready thread runs. Preemptive: if a higher‑priority thread becomes ready, it preempts the currently running one.

**Rate Monotonic Scheduling (RMS)**: Priorities assigned inversely to period (shorter period → higher priority). Optimal for independent periodic tasks.

**Deadline Monotonic Scheduling (DM)**: Priorities assigned inversely to relative deadline.

### 3.2 Dynamic Priority Scheduling

**Earliest Deadline First (EDF)**: The thread with the earliest deadline runs. EDF is optimal for preemptive systems (can achieve 100% utilization). However, overload handling is tricky.

### 3.3 Linux Real‑Time Scheduling Classes

Linux provides three real‑time scheduling policies:

- **SCHED_FIFO**: First‑in, first‑out within same priority; runs until blocked or preempted by a higher‑priority thread.
- **SCHED_RR**: Round‑robin within same priority with a time quantum.
- **SCHED_DEADLINE**: Implements EDF with a CBS (Constant Bandwidth Server) for temporal isolation.

**Example** – setting a thread to SCHED_FIFO (requires root):

```c
struct sched_param param;
param.sched_priority = 50;   // 1‑99, higher number = higher priority
pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
```

### 3.4 Real‑Time Scheduling Challenges

- **Priority inversion**: a low‑priority thread holds a lock needed by a high‑priority thread. Solved by **priority inheritance** (e.g., in pthread mutexes with `PTHREAD_PRIO_INHERIT`).
- **Interrupt latency**: interrupts preempt real‑time threads; careful design needed.
- **Jitter**: variation in response time.

---

## 4. Energy‑Aware Scheduling

On mobile and embedded devices, saving power is critical. Modern CPUs have **DVFS** (Dynamic Voltage and Frequency Scaling) and **core parking**.

### 4.1 DVFS and P‑States

The scheduler can request lower operating points (P‑states) when CPU load is low. In Linux, the `cpufreq` governor (e.g., `ondemand`, `powersave`, `performance`) interacts with the scheduler.

### 4.2 big.LITTLE and Heterogeneous Architectures

ARM’s big.LITTLE (and Intel’s hybrid architecture) combine high‑performance cores (big) with energy‑efficient cores (LITTLE). The scheduler must assign tasks appropriately.

- **Scheduler‑driven** (e.g., Linux EAS – Energy‑Aware Scheduler): uses energy models to decide which core type to place a thread on.
- **Task migration** for power‑saving.

### 4.3 Energy‑Aware Scheduling in Linux (EAS)

EAS is part of the CFS and uses the **Energy Model (EM)** to estimate energy cost. It aims to run tasks on the most energy‑efficient core that meets performance requirements.

EAS is enabled when `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` is active.

---

## 5. Modern Scheduler Implementations

### 5.1 Linux CFS (Completely Fair Scheduler)

CFS replaced the O(1) scheduler in Linux 2.6.23. Its key ideas:

- **No fixed time slices**; instead, it aims for **fair CPU distribution** over a _target latency_.
- Uses a **red‑black tree** indexed by `vruntime` (virtual runtime). The task with the smallest `vruntime` runs.
- `vruntime` advances slower for higher‑priority tasks (using nice values).
- Preemption happens when a new task has smaller `vruntime` than the current task (or after a scheduler tick).

**Key parameters**:

- `sched_latency_ns` – target latency (e.g., 6 ms). Over this period, all runnable tasks should get a slice.
- `sched_min_granularity_ns` – minimum time a task runs before being preempted (e.g., 0.75 ms).

**CFS on multi‑core**: each core has its own runqueue (per‑CPU) and a load balancer moves tasks.

**Code snippet – debugging CFS via `/proc`**:

```bash
cat /proc/sys/kernel/sched_latency_ns
cat /proc/sys/kernel/sched_min_granularity_ns
```

### 5.2 Windows Scheduler

Windows uses a **priority‑based, preemptive scheduler** with 32 priority levels (0‑31).

- Priorities 16‑31 are **real‑time**; 1‑15 are **dynamic**; 0 is the zero‑page thread.
- Threads have a **current priority** that can be temporarily boosted (e.g., for I/O completion).
- Scheduler uses **quantum** (time slice) that depends on the thread’s priority and whether it’s interactive.

**Priority boosting**: I/O‑bound threads get a temporary priority increase to improve responsiveness.

**Scheduler affinity**: can be set via `SetProcessAffinityMask` or `SetThreadAffinityMask`.

### 5.3 macOS / XNU Scheduler

XNU (the kernel of macOS and iOS) uses a **multi‑level feedback queue** (MLFQ) with **priority bands**. It also includes a **real‑time** and **timeshare** classes. On Apple Silicon, it uses **performance/power‑efficiency** cores with the **E‑core/P‑core** model.

---

## 6. Scheduling in Virtualized Environments

When a hypervisor runs multiple VMs, it must schedule virtual CPUs (vCPUs) on physical CPUs. Two main approaches:

- **Bare‑metal scheduler** (e.g., KVM’s kernel‑based scheduler): each vCPU is a thread in the host, scheduled by the host OS’s scheduler. This gives good isolation but may cause **co‑scheduling** issues (a VM with multiple vCPUs may have them scheduled at different times, causing spinlock inefficiency).
- **Hypervisor‑aware scheduling**: the hypervisor may use **co‑scheduling** to run all vCPUs of a VM together, or use **paravirtualisation** to allow the guest to be aware of host scheduling.

**Credit scheduler** (Xen): each VM gets a certain number of “credits” per period; schedules vCPUs based on credits.

---

## 7. Advanced Scheduling Concepts

### 7.1 Priority Inversion and Inheritance

Classic example: a high‑priority thread H needs a lock held by a low‑priority thread L; L is preempted by a medium‑priority thread M, causing H to wait indefinitely.

**Priority inheritance**: when H blocks on L, L temporarily inherits H’s priority, so M can’t preempt L. Once L releases the lock, its priority reverts.

### 7.2 Scheduling Anomalies and the CFS Fairness Model

CFS aims for perfect fairness, but in practice, interactivity can suffer if `sched_min_granularity` is too large. The **autogroup** feature (enabled by default) groups threads from the same session, ensuring that one shell cannot starve another.

### 7.3 Gang Scheduling

Used in parallel systems (e.g., MPI). All threads of a parallel job are scheduled simultaneously. Avoids spinning on locks. Not common in general‑purpose OSes.

### 7.4 Lottery Scheduling

Probabilistic scheduling: each thread gets lottery tickets; the scheduler randomly picks a ticket. Provides proportional share and avoids starvation.

---

## 8. Tuning and Observability

As a developer, you can:

- Set thread priorities and affinity.
- Use `nice` / `renice` for CPU‑bound tasks.
- Use `taskset` to bind processes to cores.
- Use `chrt` to set real‑time scheduling.
- Observe scheduler behavior via `/proc/sched_debug`, `perf sched`, and `ftrace`.

**Example – monitoring scheduling events with `perf`**:

```bash
perf sched record -- sleep 1
perf sched latency
perf sched map
```

---

## 9. Summary

- **Multi‑core scheduling** requires per‑core run queues and load balancing.
- **Real‑time scheduling** uses fixed or dynamic priorities (FIFO, RR, EDF) with features like priority inheritance.
- **Energy‑aware scheduling** leverages DVFS and heterogeneous cores (EAS, big.LITTLE).
- **CFS** (Linux) provides fairness via virtual runtime; **Windows** uses dynamic priority boosting.
- **Virtualisation** adds another layer of scheduling with co‑scheduling challenges.
- Understanding these advanced schedulers helps you design responsive, efficient, and real‑time systems.

_“A scheduler is the conductor of the hardware orchestra—it decides when each instrument plays, ensuring the symphony of tasks is harmonious.”_

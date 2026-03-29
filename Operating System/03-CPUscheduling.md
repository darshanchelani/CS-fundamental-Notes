## 1. Why CPU Scheduling Matters

Think of the CPU as a **world‑class chef** in a busy restaurant kitchen.  
Orders (processes/threads) arrive from waiters. The chef can only cook one dish at a time, but multiple orders are pending.  
A **scheduler** is the head chef who decides:

- Which order to cook next (priority? first‑come? shortest dish first?)
- How long to work on it before switching (time slice)
- When to pause one order and resume another (preemption)

Without a good scheduler, some customers starve (starvation), the kitchen wastes time switching between orders (context‑switch overhead), and overall throughput suffers.

In OS terms, the scheduler’s goals are:

- **Fairness** – every process gets a reasonable share.
- **Efficiency** – keep the CPU busy.
- **Low latency** – interactive tasks respond quickly.
- **Throughput** – maximize completed work per time.
- **Starvation freedom** – no process waits indefinitely.

---

## 2. Core Concepts

### 2.1 Preemptive vs Non‑preemptive Scheduling

- **Non‑preemptive**: Once a process gets the CPU, it holds it until it voluntarily yields (e.g., waiting for I/O or termination). Simple but can lead to long waits.
- **Preemptive**: The scheduler can forcibly pause a running process at any time (using timer interrupts) and give the CPU to another. This is what modern OSes use.

### 2.2 Scheduler Invocation Points

The scheduler runs when:

- A process goes from running to waiting (e.g., I/O request).
- A process terminates.
- A timer interrupt occurs (preemptive systems).
- A process is created (new process must be scheduled).
- A waiting process becomes ready (e.g., I/O completes).

### 2.3 Context Switch

Switching from one process/thread to another involves saving the state of the old one (registers, program counter) and loading the state of the new one. This overhead is critical—schedulers aim to keep it minimal.

---

## 3. Classic Scheduling Algorithms

Let’s walk through each with a restaurant analogy and code simulation.

### 3.1 First‑Come, First‑Served (FCFS)

- **How it works**: Processes are served in order of arrival. Non‑preemptive.
- **Analogy**: Customers are served in the order they line up.
- **Pros**: Simple, fair in order.
- **Cons**: Convoy effect – a long CPU‑bound process blocks shorter ones; poor interactive response.

**Example** – Processes: P1 (24 ms), P2 (3 ms), P3 (3 ms) in that order.  
Average waiting time = (0 + 24 + 27)/3 = 17 ms.  
If P2 and P3 had arrived earlier, they’d finish quickly.

---

### 3.2 Shortest Job First (SJF) / Shortest Remaining Time First (SRTF)

- **Non‑preemptive SJF**: Choose the process with the smallest total burst time.
- **Preemptive SRTF**: At each moment, run the process with the shortest remaining time. This is optimal for minimising average waiting time.
- **Analogy**: Cook the dish that takes the least time first (shortest job) to keep the queue moving.
- **Problem**: Starvation possible if long jobs keep arriving; need to know future burst times (prediction used).

**Example** (SRTF):  
Arrival: P1(0ms, burst 8), P2(1ms, burst 4), P3(2ms, burst 2)  
Timeline: 0‑1: P1, 1‑2: P2? Actually at time 1, P1 left 7, P2 4 → P2 runs until time 2, then P3 arrives (burst 2) – preempts P2 (left 3). Then P3 runs from 2‑4, P2 from 4‑7, P1 from 7‑15.

---

### 3.3 Round Robin (RR)

- **How it works**: Each process gets a fixed time quantum (time slice). After quantum expires, it’s preempted and moved to end of ready queue.
- **Analogy**: The chef spends 5 minutes on each order, then rotates to the next, cycling through all.
- **Pros**: Fair, responsive.
- **Cons**: Quantum too small → excessive context switches; too large → behaves like FCFS.

**Example**: Quantum = 4 ms.  
Processes: P1 (burst 24), P2 (3), P3 (3).  
Timeline:  
0‑4: P1, 4‑7: P2, 7‑10: P3, 10‑14: P1, 14‑18: P1, 18‑22: P1, 22‑24: P1 (finish).  
Average waiting time = (6 + 4 + 7)/3 = 5.67 ms (much better than FCFS).

---

### 3.4 Priority Scheduling

- **How it works**: Each process has a priority; the highest priority runs. Can be preemptive or non‑preemptive.
- **Problem**: Starvation – low‑priority processes may never run.
- **Solution**: Aging – gradually increase priority of waiting processes.

**Analogy**: VIP customers are served first. After a while, regular customers get VIP status.

---

### 3.5 Multilevel Queue & Multilevel Feedback Queue (MLFQ)

- **MLFQ**: Used in modern OSes (e.g., Linux, Windows). Processes move between queues based on behaviour.
  - Multiple queues with different priorities.
  - Higher priority queues get smaller time quanta.
  - CPU‑bound processes sink to lower queues; interactive (I/O‑bound) processes stay high.
- **Analogy**: Airport security – priority lanes for frequent fliers, but if someone in the regular lane waits too long, they get expedited.

---

## 4. Code Simulation: Building a Simple Scheduler

Let’s implement a **Round Robin** scheduler in Python to see how it works under the hood.  
We’ll simulate processes with arrival times and burst times.

```python
import collections

class Process:
    def __init__(self, pid, arrival, burst):
        self.pid = pid
        self.arrival = arrival
        self.burst = burst
        self.remaining = burst
        self.completion = 0
        self.waiting = 0
        self.turnaround = 0

def round_robin(processes, quantum):
    # Sort by arrival
    processes.sort(key=lambda p: p.arrival)
    ready_queue = collections.deque()
    time = 0
    idx = 0
    n = len(processes)
    completed = 0

    while completed < n:
        # Add all processes that have arrived by current time
        while idx < n and processes[idx].arrival <= time:
            ready_queue.append(processes[idx])
            idx += 1

        if not ready_queue:
            # No process ready, jump to next arrival
            time = processes[idx].arrival
            continue

        current = ready_queue.popleft()
        # Run for quantum or remaining burst, whichever less
        run_time = min(current.remaining, quantum)
        current.remaining -= run_time
        time += run_time

        # Re‑check arrivals during this run
        while idx < n and processes[idx].arrival <= time:
            ready_queue.append(processes[idx])
            idx += 1

        if current.remaining == 0:
            # Process finished
            current.completion = time
            current.turnaround = current.completion - current.arrival
            current.waiting = current.turnaround - current.burst
            completed += 1
        else:
            # Not finished, put back at end of queue
            ready_queue.append(current)

    # Print results
    print("PID\tArrival\tBurst\tCompletion\tTurnaround\tWaiting")
    for p in processes:
        print(f"{p.pid}\t{p.arrival}\t{p.burst}\t{p.completion}\t\t{p.turnaround}\t\t{p.waiting}")
    avg_wait = sum(p.waiting for p in processes) / n
    avg_turn = sum(p.turnaround for p in processes) / n
    print(f"\nAverage waiting time: {avg_wait:.2f} ms")
    print(f"Average turnaround time: {avg_turn:.2f} ms")

# Example
if __name__ == "__main__":
    procs = [
        Process(1, 0, 24),
        Process(2, 0, 3),
        Process(3, 0, 3)
    ]
    round_robin(procs, quantum=4)
```

**Output** (quantum 4):

```
PID	Arrival	Burst	Completion	Turnaround	Waiting
1	0	24	30		30		6
2	0	3	7		7		4
3	0	3	10		10		7

Average waiting time: 5.67 ms
Average turnaround time: 15.67 ms
```

You can extend this to simulate SJF, priority, etc. This kind of simulation helps internalise how scheduling policies affect metrics.

---

## 5. Real‑World Schedulers

### 5.1 Linux Completely Fair Scheduler (CFS)

- Uses a **red‑black tree** to keep tasks sorted by virtual runtime (`vruntime`).
- Each task gets a time slice proportional to its weight (priority). CFS ensures that over a period, all runnable tasks receive an equal share of CPU.
- It’s **O(log N)** for picking next task, and it’s **work‑conserving**.

**Key idea**: Instead of fixed quanta, it assigns a target latency and distributes CPU so that no task gets less than a minimum granularity.

### 5.2 Windows Scheduler

- Uses a **priority‑based, preemptive** scheduler with 32 priority levels.
- The highest priority ready thread runs. Priorities are dynamic: I/O‑bound threads get boosted; CPU‑bound threads decay.
- Uses quantum adjustments to improve responsiveness.

### 5.3 Real‑Time Schedulers (e.g., Linux RT)

- **SCHED_FIFO**: First‑in, first‑out, non‑preemptive among same priority, but preempts lower priorities.
- **SCHED_RR**: Round‑robin among same priority with time quanta.
- **SCHED_DEADLINE**: Earliest deadline first (EDF) – guarantees timing constraints.

---

## 6. Advanced Topics

### 6.1 Context‑Switch Overhead

Modern CPUs have hundreds of registers. Switching between processes also flushes the TLB (Translation Lookaside Buffer), causing a temporary performance hit.  
**Balancing act**: Quantum must be large enough to amortise overhead, but small enough to maintain responsiveness.

### 6.2 Multi‑Core Scheduling

- **Symmetric Multiprocessing (SMP)**: Each core runs its own scheduler; load balancing moves tasks between cores.
- **Affinity**: Binding processes to specific cores improves cache hit rates.
- **Push vs Pull** migration: When a core is idle, it “pulls” work from busy cores.

### 6.3 Energy‑Aware Scheduling

Modern schedulers consider power consumption. On mobile devices, they use **EAS (Energy‑Aware Scheduling)** to pack tasks onto a subset of cores and idle others, or use big.LITTLE architectures (high‑performance vs. energy‑efficient cores).

---

## 7. How to Tune Scheduler Behaviour

As a developer, you can influence scheduling:

- **Nice values** (Unix): `nice()` to lower priority, `renice` to adjust.
- **Thread priorities** (`pthread_setschedparam`).
- **Cgroups** (Linux) – limit CPU shares for groups of processes.
- **Real‑time priorities** – careful: they can starve system processes.

---

## 8. Common Pitfalls & Misconceptions

- **“More threads = faster”** – Not if they’re CPU‑bound and you have limited cores. Threads add scheduler overhead.
- **“Round robin is always fair”** – Fair in terms of CPU time, but not necessarily response time if quantum is too large.
- **“Real‑time priority means instant execution”** – Only if it’s the highest runnable thread; it still must be scheduled and may wait for resources.

---

## 9. Summary

CPU scheduling is the **pulse of the operating system**. It’s what makes multitasking possible, balancing the conflicting needs of responsiveness, fairness, and throughput.

- **Algorithms** – from simple FCFS to advanced MLFQ and CFS.
- **Trade‑offs** – quantum size, preemption, priority management.
- **Real‑world** – Linux’s CFS, Windows’ dynamic priority, real‑time extensions.

When you write multi‑threaded code, understanding the scheduler helps you design for performance. When your application feels sluggish, you’ll know whether to blame a poor scheduling policy or your own thread design.

_“The scheduler is the operating system’s diplomat – it negotiates between demands of processes, users, and hardware, striving for harmony.”_

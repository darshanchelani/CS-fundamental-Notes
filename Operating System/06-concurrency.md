## 1. The Concurrency Problem

Imagine a **busy kitchen** with multiple chefs (threads) sharing limited resources: a single oven (critical section), a pantry (shared data), and a dishwasher (I/O). Without coordination, chaos ensues:

- Two chefs grab the same ingredient and tear the package.
- One chef puts a cake in the oven while another takes it out prematurely.
- They all wait for each other to finish and end up blocking forever.

In operating systems, **concurrency** means multiple threads (or processes) executing simultaneously. They share memory and resources. **Synchronisation** is the discipline that prevents:

- **Race conditions** – the outcome depends on the arbitrary interleaving of operations.
- **Data corruption** – inconsistent state due to overlapping reads/writes.
- **Deadlock** – each thread waits for another, so none proceed.

The OS provides primitives (mutexes, semaphores) to enforce orderly access.

---

## 2. Critical Sections and Race Conditions

A **critical section** is a piece of code that accesses a shared resource (e.g., a global variable) and must not be executed by more than one thread at a time.

**Analogy**: A single‑stall restroom. If two people enter simultaneously, it’s a disaster. A lock on the door ensures mutual exclusion.

**Race condition example** (without synchronisation):

```c
#include <pthread.h>
#include <stdio.h>

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;   // critical section
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Counter = %d\n", counter);  // often less than 2000000
    return 0;
}
```

The `counter++` operation is not atomic; it involves read, modify, write. Two threads can interleave, losing increments. We need mutual exclusion.

---

## 3. Mutex (Mutual Exclusion)

A **mutex** is a lock that allows only one thread to hold it at a time. It’s the simplest synchronisation primitive.

- **Lock** – if the mutex is free, the thread acquires it and enters the critical section. If it’s locked, the thread blocks (waits).
- **Unlock** – releases the mutex, allowing a waiting thread to acquire it.

**Analogy**: A **bathroom key**. Only one person can have the key at a time; others wait in line. When done, they return the key.

### 3.1 Using a Mutex – Fixed Code

```c
#include <pthread.h>
#include <stdio.h>

int counter = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Counter = %d\n", counter);  // always 2000000
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

**Important**:

- Mutexes are **advisory** – they only work if all threads agree to use them.
- Locking too coarsely (large critical sections) reduces concurrency.
- Locking too finely increases overhead.

### 3.2 Mutex Attributes

In practice, mutexes can be **recursive** (allow the same thread to lock again), **error‑checking**, or **robust** (handle lock holder death). The default is non‑recursive.

---

## 4. Semaphores

A **semaphore** is a more general synchronisation primitive, introduced by Dijkstra. It’s an integer with two atomic operations:

- **P** (from Dutch _proberen_, to test) – decrement; if result < 0, block.
- **V** (from _verhogen_, to increment) – increment; if there are waiting threads, wake one.

A **binary semaphore** (value 0 or 1) acts like a mutex, but semaphores have no ownership concept – any thread can signal a semaphore. This makes them useful for signalling between threads.

A **counting semaphore** allows up to N threads to enter a critical section (e.g., a pool of identical resources).

**Analogy**: A **parking lot** with 5 spaces. The semaphore value = number of free spaces.

- P = take a space; if none, wait.
- V = free a space; notify waiting cars.

### 4.1 Semaphore Example (POSIX)

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

sem_t sem;
int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        sem_wait(&sem);
        counter++;
        sem_post(&sem);
    }
    return NULL;
}

int main() {
    sem_init(&sem, 0, 1);  // 0 for intra‑process, initial value 1
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    printf("Counter = %d\n", counter);
    sem_destroy(&sem);
    return 0;
}
```

**Binary semaphore as mutex** – works, but mutexes are generally preferred for mutual exclusion because they are often faster and have ownership tracking (deadlock detection).

### 4.2 Counting Semaphore – Producer‑Consumer Example

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>

#define BUFFER_SIZE 5
int buffer[BUFFER_SIZE];
int in = 0, out = 0;
sem_t empty, full;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* producer(void* arg) {
    int item = 0;
    while (1) {
        sem_wait(&empty);          // wait for an empty slot
        pthread_mutex_lock(&mutex);
        buffer[in] = item++;
        in = (in + 1) % BUFFER_SIZE;
        printf("Produced %d\n", buffer[in-1]);
        pthread_mutex_unlock(&mutex);
        sem_post(&full);           // signal that a slot is full
    }
    return NULL;
}

void* consumer(void* arg) {
    while (1) {
        sem_wait(&full);           // wait for a full slot
        pthread_mutex_lock(&mutex);
        int item = buffer[out];
        out = (out + 1) % BUFFER_SIZE;
        printf("Consumed %d\n", item);
        pthread_mutex_unlock(&mutex);
        sem_post(&empty);          // signal an empty slot
    }
    return NULL;
}

int main() {
    sem_init(&empty, 0, BUFFER_SIZE);
    sem_init(&full, 0, 0);
    pthread_t prod, cons;
    pthread_create(&prod, NULL, producer, NULL);
    pthread_create(&cons, NULL, consumer, NULL);
    pthread_join(prod, NULL);
    pthread_join(cons, NULL);
    sem_destroy(&empty);
    sem_destroy(&full);
    return 0;
}
```

Here, `empty` counts free slots, `full` counts occupied slots. The mutex protects the shared buffer pointers.

---

## 5. Deadlock

**Deadlock** occurs when two or more threads are blocked forever, each waiting for a resource held by another.

**Analogy**: Two cars meet at a four‑way intersection. Each wants to go straight, but they both refuse to move until the other goes. They wait forever.

### 5.1 Necessary Conditions (Coffman Conditions)

All four must hold for deadlock to occur:

1. **Mutual exclusion** – resources cannot be shared.
2. **Hold and wait** – a thread holds a resource while waiting for another.
3. **No preemption** – resources cannot be forcibly taken.
4. **Circular wait** – there exists a cycle of threads each waiting for a resource held by the next.

### 5.2 Deadlock Example (Two Mutexes)

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

pthread_mutex_t mutexA = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutexB = PTHREAD_MUTEX_INITIALIZER;

void* thread1(void* arg) {
    pthread_mutex_lock(&mutexA);
    printf("Thread1 locked A\n");
    sleep(1);   // give thread2 time to lock B
    pthread_mutex_lock(&mutexB);
    printf("Thread1 locked B\n");
    pthread_mutex_unlock(&mutexB);
    pthread_mutex_unlock(&mutexA);
    return NULL;
}

void* thread2(void* arg) {
    pthread_mutex_lock(&mutexB);
    printf("Thread2 locked B\n");
    sleep(1);
    pthread_mutex_lock(&mutexA);
    printf("Thread2 locked A\n");
    pthread_mutex_unlock(&mutexA);
    pthread_mutex_unlock(&mutexB);
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, thread1, NULL);
    pthread_create(&t2, NULL, thread2, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    return 0;
}
```

Thread1 locks A, then tries B; Thread2 locks B, then tries A. They wait forever.

### 5.3 Deadlock Prevention & Avoidance

**Prevention**: Break one of the four conditions.

- **Hold and wait** – require threads to request all resources at once (not always practical).
- **No preemption** – allow the OS to preempt resources (risky, can lead to inconsistencies).
- **Circular wait** – impose a total ordering on resources. For example, always lock mutex A before B.

**Avoidance**: Use algorithms like the **Banker’s Algorithm** (for processes with known maximum resource needs). The OS only grants requests if it can guarantee deadlock‑free state.

**Detection and Recovery**: Allow deadlock, detect it (e.g., by building a resource allocation graph), and then break it by aborting threads or preempting resources.

In practice, developers use **lock ordering** to prevent circular wait.

**Fixed code** (lock ordering):

```c
// In both threads, always lock A then B
void* thread2(void* arg) {
    pthread_mutex_lock(&mutexA);   // same order as thread1
    pthread_mutex_lock(&mutexB);
    // ...
    pthread_mutex_unlock(&mutexB);
    pthread_mutex_unlock(&mutexA);
}
```

---

## 6. Other Synchronisation Primitives

- **Condition variables**: Allow threads to wait for a condition to become true (e.g., queue not empty). Used with a mutex.
- **Read‑write locks**: Allow multiple readers or a single writer.
- **Spinlocks**: Busy‑wait loops; used in low‑level kernel code where waiting would be too expensive.

**Condition variable example** (producer‑consumer with `pthread_cond`):

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

void* producer(void* arg) {
    pthread_mutex_lock(&mutex);
    ready = 1;
    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&mutex);
    return NULL;
}

void* consumer(void* arg) {
    pthread_mutex_lock(&mutex);
    while (!ready) {
        pthread_cond_wait(&cond, &mutex);
    }
    printf("Got signal!\n");
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

---

## 7. Common Pitfalls

- **Forgot to lock/unlock** – leads to race conditions.
- **Double lock** – deadlock (unless recursive mutex).
- **Lock inversion** – different lock order causes deadlock.
- **Holding locks while doing blocking I/O** – can cause long delays.
- **Too coarse locking** – reduces parallelism.
- **Priority inversion** – a low‑priority thread holds a lock needed by a high‑priority thread. Can be mitigated with priority inheritance (provided by some mutexes, e.g., in Linux `PTHREAD_PRIO_INHERIT`).

---

## 8. Concurrency in Modern Languages

While the OS provides low‑level primitives, high‑level languages offer abstractions:

- **Python** – `threading.Lock`, `threading.Semaphore`, `queue.Queue`.
- **Go** – goroutines with channels and `sync.Mutex`.
- **Rust** – `Mutex<T>`, `RwLock<T>` with ownership semantics.
- **Java** – `synchronized`, `ReentrantLock`, `Semaphore`.

These hide the OS details but the underlying concepts remain the same.

---

## 9. Putting It All Together

Concurrency is a fundamental challenge in OS design. The OS provides **mutexes** for mutual exclusion, **semaphores** for counting resources and signalling, and the scheduler controls when threads run. Developers must use these primitives correctly to avoid:

- **Race conditions** → data corruption.
- **Deadlock** → system freeze.
- **Starvation** → some threads never run.

**Key principle**: Keep critical sections as short as possible. Use lock ordering to avoid deadlock. Prefer higher‑level abstractions (like message queues) when they fit the problem.

---

## 10. Summary

- **Mutex**: Binary lock, ownership‑based, protects a critical section.
- **Semaphore**: General integer counter; binary semaphore can act as mutex; counting semaphore manages a pool of resources.
- **Deadlock**: Four conditions must hold; prevent by breaking circular wait (lock ordering).
- **Concurrency primitives** are the tools the OS gives us to build reliable multi‑threaded programs.

understanding why a database transaction deadlocked.

_“Concurrency is not parallelism; it’s about structure. Synchronisation is the art of taming nondeterminism.”_

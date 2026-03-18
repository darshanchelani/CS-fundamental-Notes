Exception and interrupt handling is the mechanism by which a CPU responds to asynchronous events (interrupts from hardware) or synchronous events (exceptions from instructions). It's the foundation of multitasking, device drivers, and fault tolerance. As a senior engineer, understanding the deep details—how state is saved, how the CPU finds the handler, how it returns, and how priorities and nesting work—is essential for writing robust low-level code and debugging system crashes.

Let's dive into the anatomy of exception and interrupt handling, from hardware to software, with concrete examples from x86 and ARM.

---

## 1. Exceptions vs. Interrupts

- **Exceptions**: Synchronous events caused by the execution of an instruction. Examples: page fault, division by zero, system call (trap). The CPU usually saves the address of the offending instruction (or next) so that execution can resume after handling.
- **Interrupts**: Asynchronous events triggered by external hardware (timer, disk, network). They occur independently of the current instruction stream.

Both cause the CPU to transfer control to a predefined handler. The distinction matters for resumption and for understanding the state saved.

---

## 2. Overview of the Handling Flow

When an exception or interrupt occurs, the hardware performs a sequence of steps:

1. **Detect the event** and determine its type (vector number).
2. **Save minimal state** (typically program counter and processor status) to allow resumption.
3. **Fetch the handler address** from a table (Interrupt Vector Table, Interrupt Descriptor Table) indexed by vector.
4. **Switch to privileged mode** (if not already) and possibly to a separate stack.
5. **Transfer control** to the handler (usually via a far call or jump).
6. **Handler executes** (saves additional state, handles the event, may re-enable interrupts, acknowledges device).
7. **Return from handler** using a special instruction that restores saved state and resumes the interrupted code.

---

## 3. Detailed Steps in Hardware

### 3.1 Interrupt Detection and Acknowledgment

For interrupts, an external device signals the CPU via an interrupt request line. In modern systems, an **Interrupt Controller** (like the 8259 PIC or APIC) manages multiple devices, prioritizes them, and forwards the interrupt to the CPU with a vector number.

**Process**:

- Device asserts interrupt line.
- Interrupt controller determines priority and sends an interrupt signal to CPU (e.g., INTR pin).
- CPU checks if interrupts are enabled (IF flag in x86). If enabled, it initiates an interrupt acknowledge cycle.
- CPU sends an interrupt acknowledge (INTA) to the controller, which responds with the vector number (0-255).

### 3.2 Saving State

The CPU automatically saves a minimal context on the stack (or dedicated registers) to allow resumption. The exact state saved depends on architecture and the type of event.

**x86 (protected mode)** :

- For interrupts and exceptions, the CPU pushes:
  - **SS, ESP** (if privilege level changes)
  - **EFLAGS** (the flags register)
  - **CS, EIP** (code segment and instruction pointer)
  - **Error code** (for some exceptions, e.g., page fault)
- The stack pointer (ESP) before the push is from the interrupted context (or from the TSS if privilege change).
- The CPU then loads new CS and EIP from the IDT entry.

**ARM (Cortex-M)** :

- Automatically pushes xPSR, PC, LR, R12, R3-R0 onto the current stack (MSP or PSP).
- For some exceptions, it may also push additional registers if FPU is used.
- The vector number is used to fetch the handler address from the vector table.

### 3.3 Finding the Handler: The Vector Table

The **Interrupt Vector Table** (real mode) or **Interrupt Descriptor Table** (protected mode) is an array of handler addresses (and descriptors). Each entry corresponds to a vector number (0-255). On x86, each IDT entry contains:

- Base address of handler (segment and offset)
- Type (interrupt gate, trap gate, task gate)
- Privilege level
- Present flag

On interrupt/exception, the CPU multiplies the vector by entry size, indexes into the IDT (location in IDTR register), and loads CS:EIP.

### 3.4 Privilege Level Change

If the event occurs in user mode (lower privilege) and the handler is in kernel mode (higher privilege), the CPU must switch stacks to a trusted stack to avoid corrupting user stack. This is done via the **Task State Segment (TSS)** on x86, which provides pointers to kernel stacks for each privilege level. The CPU automatically pushes SS and ESP from the TSS and switches to that stack before pushing the rest of the state.

---

## 4. Handler Execution: Software's Role

The handler is software (OS or driver code). Its typical tasks:

- **Save volatile registers** (if not already saved automatically) that it will use.
- **Acknowledge the interrupt** to the device (clear the interrupt request) to prevent repeated interrupts.
- **Perform minimal work** (e.g., copy data, signal a task) to keep interrupt latency low.
- **If using a bottom-half mechanism** (e.g., tasklets, workqueues in Linux), schedule the bottom half and return from interrupt quickly.
- **Restore saved registers** and execute the return instruction.

### Example: x86 Interrupt Handler Skeleton

```asm
; Assume we are in kernel mode, interrupts disabled on entry
; The CPU has already pushed EFLAGS, CS, EIP, and error code (if any)

my_isr:
    ; Save any registers we'll use
    push eax
    push ebx
    push ecx
    push edx

    ; Acknowledge interrupt to PIC (if needed)
    mov al, 0x20        ; End of Interrupt (EOI) command
    out 0x20, al        ; Send to master PIC (port 0x20)

    ; Do actual work (e.g., read keyboard scancode)
    in al, 0x60         ; Read keyboard data port

    ; ... process ...

    ; Restore registers
    pop edx
    pop ecx
    pop ebx
    pop eax

    ; Return from interrupt
    iret                ; pops EIP, CS, EFLAGS (and possibly error code)
```

Note: `iret` automatically adjusts the stack depending on whether an error code was pushed. Some architectures have different return instructions (e.g., `rti` on ARM).

---

## 5. Nesting and Priorities

### 5.1 Priorities

Interrupt controllers assign priorities to different interrupt sources. Higher-priority interrupts can interrupt lower-priority handlers. The CPU's interrupt flag can be set to allow only interrupts above a certain threshold.

**Programmable Interrupt Controller (PIC)** : In x86 legacy systems, the 8259 PIC has 8 IRQ lines, each with programmable priority. Modern systems use the **Advanced Programmable Interrupt Controller (APIC)** which supports more interrupts and inter-processor interrupts (IPIs).

**Interrupt priority levels (IPL)** : Some architectures (like VAX) have a CPU priority level; interrupts below current priority are masked.

### 5.2 Nesting

When an interrupt handler is executing, if a higher-priority interrupt arrives, the CPU can (if interrupts are enabled at the appropriate level) preempt the current handler. This requires careful design:

- **Handler must be reentrant** or use separate stacks/data per priority level.
- **Stack must be sufficient** for nested interrupts.
- **Interrupts are typically re-enabled** early in the handler (after saving state) to allow higher-priority interrupts.

**Example flow**:

- Low-priority interrupt (IRQ1) starts, saves state, re-enables interrupts.
- High-priority interrupt (IRQ2) arrives, preempts IRQ1 handler.
- IRQ2 handler runs to completion, returns.
- IRQ1 handler resumes, eventually returns.

### 5.3 Stack Management for Nesting

Each nested interrupt pushes more state onto the same stack. The stack must be large enough to accommodate worst-case nesting depth. Kernel stacks are often several pages (4-8 KB) to handle deep nesting.

Some architectures (like ARM Cortex-M) use separate stack pointers for main and process, and the hardware automatically uses the main stack for exceptions, simplifying nesting.

---

## 6. Returning from Interrupts

The **return from interrupt** instruction (e.g., `iret` on x86, `rti` on ARM) restores the saved state:

- Pops the saved PC, status register, and possibly other registers.
- If a privilege level change occurred on entry, it also restores the user stack.
- Atomically re-enables interrupts (the saved EFLAGS/PSR may have interrupts enabled, so the CPU will respond to new interrupts after the return).

Important: The return instruction must match the type of frame saved. Some exceptions push an error code; the handler must adjust the stack before `iret` or use a different return variant.

---

## 7. Architecture Specifics

### 7.1 x86 (Protected Mode)

- **IDT**: 256 entries, each 8 bytes (gate descriptor).
- **Interrupt gate**: Disables further interrupts on entry (IF cleared).
- **Trap gate**: Leaves IF unchanged (used for system calls).
- **Task gate**: Hardware task switching (rarely used).
- **Error code**: For exceptions like page fault, pushed automatically; handler must remove it before `iret`.

**System Call Example (int 0x80)** :

- User program executes `int 0x80`.
- CPU switches to kernel mode (via TSS), pushes user SS, ESP, EFLAGS, CS, EIP.
- Vector 0x80 (trap gate) is fetched; handler runs with interrupts enabled.
- Handler uses `iret` to return, restoring user state.

### 7.2 ARM Cortex-M

- **Vector table** at address 0x00000000 (or VTOR register).
- First word: initial stack pointer (MSP).
- Second word: reset handler.
- Subsequent words: exception handlers (e.g., NMI, HardFault, SVCall, PendSV, SysTick, and IRQ 0..n).
- On exception, hardware automatically pushes xPSR, PC, LR, R12, R3-R0 onto current stack.
- It fetches handler address from vector table, and updates PC.
- Return uses `BX LR` with a special EXC_RETURN value in LR (bits indicate which stack to use, return to thread/handler mode, etc.).

---

## 8. Software Engineer's Perspective

When writing interrupt handlers (or operating system code), consider:

- **Keep handlers short**: Long handlers increase interrupt latency and can cause missed interrupts.
- **Use bottom halves**: Defer work to a scheduled task (tasklet, workqueue, DPC) that runs with interrupts enabled.
- **Reentrancy**: If interrupts can nest, ensure data structures are protected (disable interrupts or use spinlocks).
- **Stack size**: Ensure kernel stacks are large enough for worst-case nesting.
- **Device acknowledgment**: Always acknowledge the device to prevent repeated interrupts. For level-triggered interrupts, ensure the device deasserts the line.
- **Race conditions**: Shared data between handler and rest of kernel must be protected with appropriate synchronization (spinlocks that disable interrupts).

**Example: Linux top half and bottom half**:

```c
// Top half (interrupt context)
irqreturn_t my_interrupt(int irq, void *dev_id) {
    // Read device, schedule bottom half
    tasklet_schedule(&my_tasklet);
    return IRQ_HANDLED;
}

// Bottom half (runs later, with interrupts enabled)
void my_tasklet_func(unsigned long data) {
    // Do heavy processing
}
```

---

## 9. Advanced Topics

### 9.1 Interrupt Affinity

In SMP systems, you can bind specific interrupts to specific CPUs using the `/proc/irq/*/smp_affinity` interface. This improves cache locality and balances load.

### 9.2 Message Signaled Interrupts (MSI)

PCIe devices use MSI/MSI-X: they write a small message to a special address, which the CPU treats as an interrupt. This avoids dedicated interrupt lines and allows many interrupts per device.

### 9.3 NMI (Non-Maskable Interrupt)

A special interrupt that cannot be disabled by software. Used for hardware errors (watchdog timers, power failure). Handlers must be extremely careful (no locks, minimal state).

### 9.4 Spurious Interrupts

Occasionally, an interrupt may be signaled but then withdrawn (e.g., due to race conditions). The handler must detect and ignore such cases (often just return without EOI).

---

## 10. Conclusion

Exception and interrupt handling is a delicate dance between hardware and software. The hardware provides the basic mechanism: saving minimal state, vectoring to a handler, and providing a return path. The software (OS) builds on this to implement device drivers, process scheduling, and fault recovery. Understanding this dance is essential for low-level programming, performance tuning, and debugging.

## 1. What Is a Hypervisor?

A **hypervisor** (Virtual Machine Monitor) is a software layer that abstracts physical hardware, allowing multiple **guest VMs** to run concurrently. Each guest believes it has full control of the hardware, but the hypervisor mediates access.

**Types**:

- **Type 1 (bare‑metal)** – runs directly on hardware (e.g., Xen, KVM when used without a host OS, VMware ESXi).
- **Type 2 (hosted)** – runs as an application on a host OS (e.g., VirtualBox, VMware Workstation).

Modern hypervisors use **hardware virtualization extensions** to avoid the overhead of binary translation and to provide efficient, secure isolation.

**Analogy**: A hypervisor is like a **building manager** who divides a large building into apartments (VMs). Each apartment has its own locked door, and the manager ensures tenants don’t interfere with each other while sharing the building’s utilities (CPU, memory, I/O).

---

## 2. Hardware Virtualization Extensions: VT‑x and AMD‑V

Before these extensions, software hypervisors used **binary translation** to run unmodified guest OSes—very slow. Intel VT‑x and AMD‑V (SVM) introduced hardware support for virtualization.

### 2.1 Intel VT‑x (VMX)

VT‑x adds two new CPU modes:

- **VMX root mode** – the hypervisor runs here; it has full control.
- **VMX non‑root mode** – guests run here; certain operations (e.g., privileged instructions) cause a **VM exit** back to the hypervisor.

Key structures:

- **VMCS (Virtual Machine Control Structure)** – per‑vCPU data structure containing guest state, host state, and control fields. When a VM exit occurs, the CPU saves guest state into the VMCS and loads host state from it. On VM entry, it does the reverse.

**VM exits** are triggered by:

- Privileged instructions (e.g., `mov cr3`, `hlt`).
- Page faults.
- I/O operations.
- Interrupts.
- Invocation of the `vmcall` instruction (for hypercalls).

The hypervisor configures which events cause exits via the VMCS.

### 2.2 AMD‑V (SVM)

Analogous to VT‑x:

- **VMRUN** instruction enters the guest.
- **VMCB (Virtual Machine Control Block)** serves the same role as VMCS.
- **VM exits** trigger hypervisor entry.

Both technologies provide **hardware‑assisted nesting** (e.g., nested virtualization) and **tagged TLB** to reduce flush overhead.

**Analogy**: VT‑x/AMD‑V are like **security doors** between apartments. Instead of the building manager having to watch every move (binary translation), the doors automatically trap any attempt to leave the apartment without permission. The manager only gets involved when a tenant tries something they shouldn’t.

---

## 3. KVM (Kernel‑based Virtual Machine)

KVM is a Linux **kernel module** that turns the Linux kernel into a Type‑1 hypervisor. It leverages hardware virtualization extensions and exposes the `/dev/kvm` interface to userspace.

### 3.1 How KVM Works

1. **KVM kernel module** (kvm.ko) initializes and enables VT‑x/AMD‑V.
2. A userspace program (typically **QEMU**) opens `/dev/kvm` and uses **KVM API** to create VMs, vCPUs, and memory.
3. QEMU maps guest physical memory to host virtual memory via `mmap` on `/dev/kvm` or via KVM_SET_USER_MEMORY_REGION.
4. Each vCPU runs via the **KVM_RUN** ioctl. The kernel enters VMX non‑root mode and executes guest code. When a VM exit occurs, the kernel handles it (if simple) or returns control to QEMU to emulate devices.
5. QEMU emulates I/O devices (disk, network, etc.) and handles VM exits that require device emulation.

### 3.2 KVM API Example (Simplified C)

```c
#include <fcntl.h>
#include <linux/kvm.h>
#include <sys/ioctl.h>
#include <sys/mman.h>

int main() {
    int kvm_fd = open("/dev/kvm", O_RDWR);
    int vm_fd = ioctl(kvm_fd, KVM_CREATE_VM, 0);
    int vcpu_fd = ioctl(vm_fd, KVM_CREATE_VCPU, 0);
    struct kvm_run *run = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,
                               MAP_SHARED, vcpu_fd, 0);

    // Setup guest memory (simplified)
    void *guest_mem = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE,
                           MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    struct kvm_userspace_memory_region region = {
        .slot = 0,
        .guest_phys_addr = 0,
        .memory_size = 0x1000,
        .userspace_addr = (__u64)guest_mem,
    };
    ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region);

    // Put some guest code in guest_mem (e.g., a simple asm)
    // ...

    // Run the vCPU
    while (1) {
        ioctl(vcpu_fd, KVM_RUN, 0);
        switch (run->exit_reason) {
        case KVM_EXIT_HLT:
            // guest halted
            return 0;
        case KVM_EXIT_IO:
            // handle I/O
            break;
        // ... other exits
        }
    }
}
```

### 3.3 QEMU + KVM

In practice, QEMU handles the heavy lifting. To run a VM:

```bash
qemu-system-x86_64 -enable-kvm -m 2G -smp 2 -drive file=disk.qcow2,format=qcow2
```

The `-enable-kvm` flag uses KVM acceleration. QEMU acts as the device model.

### 3.4 KVM Advantages

- Tight integration with Linux (scheduler, memory management).
- Leverages Linux for resource management.
- Wide hardware support.
- Can run unmodified guests (full virtualization).

---

## 4. Xen Hypervisor

Xen is a Type‑1 hypervisor that runs directly on hardware, with a privileged **dom0** (control domain) and multiple unprivileged **domU** guests. It uses **paravirtualization** (PV) for performance, but also supports hardware‑assisted **HVM** (Hardware Virtual Machine) for unmodified guests.

### 4.1 Xen Architecture

- **Hypervisor** – the minimal layer that runs on hardware and manages CPU, memory, interrupts.
- **dom0** – a privileged guest with direct hardware access, runs the toolstack (xl, libxl) and manages domU guests.
- **domU** – unprivileged guests.

### 4.2 Xen Modes

- **PV (Paravirtualization)**: Guest kernel is modified to use **hypercalls** (instead of privileged instructions) and to provide its own device drivers. This avoids VM exits and yields near‑native performance.
- **HVM (Hardware Virtual Machine)**: Uses VT‑x/AMD‑V to run unmodified guests. A device model (qemu‑dm) in dom0 emulates hardware.
- **PVH**: Hybrid – uses HVM for hardware assistance but paravirtualized drivers.

### 4.3 Hypercalls

Paravirtualized guests invoke the hypervisor via a specific instruction (e.g., `vmcall` on Intel, `vmmcall` on AMD). The hypervisor handles the request (e.g., updating page tables, scheduling). This is more efficient than trapping every privileged instruction.

### 4.4 Example: Creating a Xen VM (PV mode)

```bash
# Create a config file /etc/xen/myvm.cfg
kernel = "/boot/vmlinuz-4.9.0"
ramdisk = "/boot/initrd.img-4.9.0"
memory = 1024
vcpus = 2
name = "myvm"
disk = ['phy:/dev/vg0/myvm,xvda,w']
vif = ['bridge=xenbr0']

# Start the VM
xl create /etc/xen/myvm.cfg
xl list
```

---

## 5. KVM vs Xen – A Quick Comparison

| Feature              | KVM                                  | Xen                                          |
| -------------------- | ------------------------------------ | -------------------------------------------- |
| **Type**             | Type‑1 (kernel module) + QEMU (user) | Type‑1 (hypervisor runs on bare metal)       |
| **Architecture**     | Integrated into Linux kernel         | Standalone hypervisor; Linux dom0 as control |
| **Default mode**     | Full virtualization (HVM) with KVM   | PV (paravirtualization) or HVM               |
| **Device emulation** | QEMU (can be in a separate process)  | QEMU‑dm in dom0 (for HVM), PV drivers in PV  |
| **Performance**      | Excellent for HVM, less overhead     | Excellent for PV; HVM similar to KVM         |
| **Management**       | libvirt, virsh, virt-manager         | xl, xm, libvirt                              |
| **Use case**         | Widely used in cloud (AWS uses KVM)  | Historically popular for enterprise (Citrix) |

Both use hardware virtualization extensions; the main difference is in architecture and paravirtualization support.

---

## 6. Hardware Virtualization Internals (Detailed)

### 6.1 VM Entry and Exit Flow

When the hypervisor decides to run a vCPU:

1. It sets up the VMCS/VMCB with the guest state (registers, control registers, etc.).
2. It executes the **VM entry** instruction (e.g., `VMLAUNCH` or `VMRESUME` on Intel; `VMRUN` on AMD).
3. The CPU loads guest state from the VMCS and switches to non‑root mode. Guest code executes.
4. When a “sensitive” event occurs (e.g., `CPUID` instruction, page fault), the CPU saves the guest state into the VMCS, loads the host state, and **exits** to the hypervisor (VM exit).
5. The hypervisor handles the exit (emulates an instruction, injects an interrupt, schedules another vCPU, etc.).
6. When ready, it uses VM entry again.

### 6.2 Nested Page Tables (NPT/EPT)

For memory virtualization, hardware provides **Extended Page Tables (EPT)** on Intel, **Nested Page Tables (NPT)** on AMD. These allow the hypervisor to maintain a second level of translation: guest physical → host physical. The CPU walks both page tables (guest virtual → guest physical → host physical) without hypervisor intervention. This eliminates the need for shadow page tables and greatly improves performance.

### 6.3 Interrupt Delivery

Hardware also supports **virtual APIC** and **posted interrupts**, allowing interrupts to be delivered directly to a vCPU without VM exit (if the hypervisor configures it).

---

## 7. Putting It All Together: A Simple KVM VM from Scratch

To truly understand, let's build a minimal VM that just runs a tiny assembly program. This uses the KVM API directly (no QEMU).

**Guest code (16‑bit real‑mode)** that prints a character and halts:

```asm
; guest.asm
[BITS 16]
start:
    mov al, 'A'
    mov ah, 0x0e
    int 0x10        ; BIOS call – this will cause a VM exit
    hlt
```

Assemble to binary: `nasm -f bin guest.asm -o guest.bin`

Now, a C program to load it via KVM (full example omitted for brevity, but the skeleton above with the memory region set to contain `guest.bin` at 0x7c00, and setting registers appropriately). This is exactly what QEMU does internally.

---

## 8. Summary

- **Hardware virtualization extensions (VT‑x/AMD‑V)** provide CPU modes (root/non‑root) and structures (VMCS/VMCB) that allow efficient, secure guest execution.
- **KVM** uses Linux as the hypervisor, with QEMU providing device emulation. It leverages VT‑x/AMD‑V via the `/dev/kvm` interface.
- **Xen** is a bare‑metal hypervisor with a control domain (dom0). It supports paravirtualization (PV) for high performance and HVM for unmodified guests.
- Both rely on the same hardware features, but differ in architecture and management models.

_“Virtualization transforms one physical machine into many – the hypervisor is the stage magician, and hardware extensions are the hidden mirrors and trapdoors.”_

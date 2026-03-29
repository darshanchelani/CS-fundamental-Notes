## 1. The Boot Journey: A High‑Level View

Think of booting as a **relay race**:

1. **Firmware** (BIOS/UEFI) – the starter pistol.
2. **Bootloader** (GRUB) – the first runner, hands off to the kernel.
3. **Kernel** – the engine, initializes hardware and mounts the root.
4. **Initramfs** – a temporary runner that brings the real root filesystem.
5. **systemd/init** – the manager that starts all services and presents a login prompt.

Each stage hands over control to the next, building up from raw hardware to a usable environment.

---

## 2. Stage 1: Firmware – BIOS vs UEFI

When you press the power button, the CPU begins executing code at a fixed memory address (reset vector). This is the firmware’s domain.

### 2.1 BIOS (Basic Input/Output System)

- Exists since the original IBM PC.
- Lives in a ROM chip on the motherboard.
- Performs **POST** (Power‑On Self‑Test): checks RAM, CPU, essential devices.
- Looks for a **bootable device** (disk, USB, CD) in a configured order.
- Loads the **first sector** (512 bytes) of that device into memory at address `0x7C00` and jumps there. This sector is the **Master Boot Record (MBR)**.

**Limitations**:

- MBR only supports 2 TB disks and 4 primary partitions.
- Boot process is rigid and runs in 16‑bit real mode.

### 2.2 UEFI (Unified Extensible Firmware Interface)

- Modern replacement for BIOS.
- Runs in 64‑bit mode (or 32‑bit) and can use large disks (GPT partition table).
- Provides a more sophisticated boot manager: it reads boot entries from NVRAM and loads **EFI executables** (files with `.efi` extension) from the EFI System Partition (ESP).
- Boot manager can be user‑configured (e.g., `efibootmgr`).

**Which one is used?** Most PCs made after 2012 use UEFI, often in “legacy” (BIOS) compatibility mode.

**Analogy**: BIOS is like a **simple key** that opens one door; UEFI is a **smart access system** that knows which doors to open based on stored profiles.

---

## 3. Stage 2: Bootloader – GRUB

The bootloader is the first software the firmware loads. Its job is to find the operating system kernel, load it into memory, and transfer control.

**GRUB (GRand Unified Bootloader)** is the most common on Linux.

### 3.1 GRUB Stages (BIOS‑MBR scenario)

- **Stage 1**: The MBR contains a tiny code that locates the next stage.
- **Stage 1.5**: Sits in the gap between MBR and first partition; contains filesystem drivers to read `/boot/grub`.
- **Stage 2**: The full GRUB core, which loads the configuration (`grub.cfg`) and presents a menu.

### 3.2 GRUB Configuration

On Linux, GRUB’s configuration is generated from `/etc/default/grub` and scripts in `/etc/grub.d/`, producing `/boot/grub/grub.cfg`.

**Example entry** (simplified):

```bash
menuentry 'Ubuntu' {
    linux   /vmlinuz-5.15.0-86-generic root=/dev/sda1 ro quiet splash
    initrd  /initrd.img-5.15.0-86-generic
}
```

- `linux` – the kernel image file.
- `initrd` – the initial RAM disk.
- `root=` – kernel command‑line parameter telling the kernel where the real root filesystem is.

### 3.3 UEFI GRUB

On UEFI systems, GRUB is installed as an EFI application (`grubx64.efi`) on the ESP. The UEFI firmware runs it directly, and GRUB then reads its configuration from the same ESP or from `/boot`.

**You can view boot entries** with:

```bash
efibootmgr -v
```

---

## 4. Stage 3: Kernel Loading and Early Boot

GRUB loads the kernel (e.g., `vmlinuz`) and the initramfs (`initrd.img`) into memory, then jumps to the kernel’s entry point.

### 4.1 Kernel Initialization (start_kernel)

The kernel (in C) starts at `start_kernel()` in `init/main.c`. It does:

- **Architecture‑specific setup**: page tables, interrupt handling, CPU detection.
- **Memory management**: initializes memory allocators.
- **Scheduler**: initializes the scheduler data structures.
- **Device drivers**: probes and initializes built‑in drivers.
- **Mounts the root filesystem**: but at this point, it only has the initramfs.

**Analogy**: The kernel is like the **engine manager** – it gets the engine running, but it still needs a steering wheel (user space) to actually drive.

### 4.2 Kernel Command Line

The bootloader passes parameters (e.g., `root=`, `quiet`, `splash`) that influence kernel behavior. You can see them in `/proc/cmdline`:

```bash
cat /proc/cmdline
```

---

## 5. Stage 4: Initramfs (Initial RAM Filesystem)

The kernel needs to mount the **real root filesystem** (e.g., `/dev/sda1`), but the driver for that disk may be a kernel module that lives on the root itself – a chicken‑and‑egg problem.

**Initramfs** solves this: it’s a small filesystem (compressed cpio archive) loaded into memory by GRUB. The kernel unpacks it and runs `/init` inside it.

### 5.1 What Initramfs Contains

- Essential kernel modules (for disk controllers, filesystems, encryption).
- A minimal user‑space with busybox or systemd tools.
- A script (`/init`) that loads the needed modules, possibly sets up LVM/encryption, and then mounts the real root and `pivot_root` to it.

**Examining initramfs**:

```bash
# View contents of current initramfs
lsinitramfs /boot/initrd.img-$(uname -r)

# Or extract it:
mkdir /tmp/initrd
cd /tmp/initrd
zcat /boot/initrd.img-$(uname -r) | cpio -idmv
```

### 5.2 The `/init` Script

Simplified version:

```bash
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
# Load necessary modules
modprobe ext4
# Mount the real root
mount /dev/sda1 /newroot
# Switch to new root
exec switch_root /newroot /sbin/init
```

After `switch_root`, the kernel runs `/sbin/init` from the real root, and the initramfs is discarded.

---

## 6. Stage 5: systemd / init – The First User‑Space Process

Once the kernel mounts the real root, it executes **the init program** – traditionally SysVinit, now almost universally **systemd** on Linux. This process gets PID 1 and is responsible for bringing up the rest of the system.

### 6.1 systemd Overview

systemd is a suite of tools that:

- Starts **units** (services, sockets, devices, mounts, etc.) in parallel, based on dependencies.
- Manages logging via `journald`.
- Handles user sessions, timers, etc.

**Core concepts**:

- **Unit files**: definitions (e.g., `nginx.service`) stored in `/etc/systemd/system/` and `/lib/systemd/system/`.
- **Targets**: groups of units (like runlevels). `multi-user.target` is the standard multi‑user non‑graphical mode; `graphical.target` adds a display manager.

**Viewing the boot process**:

```bash
systemd-analyze               # boot time summary
systemd-analyze blame         # which units took the longest
journalctl -b                 # logs from this boot
```

### 6.2 systemd Initialization Steps

1. systemd mounts filesystems (from `/etc/fstab`).
2. Starts udev to manage devices.
3. Activates swap.
4. Starts basic services (`systemd-journald`, `systemd-logind`, etc.).
5. Reaches `default.target` (often `multi-user.target` or `graphical.target`), which pulls in all necessary services.
6. Finally, a **getty** service runs on the console, presenting a login prompt.

**Analogy**: systemd is the **project manager** who knows all tasks (units), their dependencies, and executes them efficiently (in parallel).

---

## 7. Putting It All Together: A Walkthrough

Let’s trace the entire boot sequence on a typical Linux system:

1. **Power on** → CPU jumps to firmware (UEFI/BIOS).
2. **Firmware** initializes hardware, runs POST, then reads boot entry (e.g., GRUB on `/dev/sda`).
3. **GRUB** loads its core, displays a menu, and upon selection loads `vmlinuz` and `initrd.img` into memory.
4. **Kernel** starts, initializes CPU, memory, interrupts, and then runs `/init` from initramfs.
5. **Initramfs** loads necessary modules (e.g., for disk encryption), mounts the real root at `/newroot`, and `switch_root`s.
6. **Kernel** now executes `/sbin/init` (systemd) from the real root.
7. **systemd** reads its configuration, starts units in parallel, and reaches `multi-user.target`.
8. A `getty` service starts on `tty1`, presenting a login prompt.
9. **User logs in** → shell or graphical session starts.

---

## 8. Code and Command Examples

### 8.1 Inspecting Your Current Boot Configuration

```bash
# Kernel command line
cat /proc/cmdline

# List boot entries (UEFI)
efibootmgr -v

# Show bootloader configuration (GRUB)
grep -v '^#' /boot/grub/grub.cfg | head -20
```

### 8.2 Manually Exploring Initramfs

```bash
# Extract current initramfs
mkdir /tmp/initrd
cd /tmp/initrd
zcat /boot/initrd.img-$(uname -r) | cpio -idv
ls -l
# Examine the init script
less init
```

### 8.3 A Minimal Bootloader Example (for QEMU)

You can write a tiny bootloader in assembly to understand the MBR concept. Here’s a simple one that prints a message and halts:

```asm
; boot.asm – assemble with nasm -f bin boot.asm -o boot.bin
[org 0x7c00]
start:
    mov si, msg
    call print
    hlt

print:
    lodsb
    or al, al
    jz .done
    mov ah, 0x0e
    int 0x10
    jmp print
.done:
    ret

msg db 'Bootloader loaded!', 0

times 510-($-$$) db 0
dw 0xaa55   ; MBR signature
```

Run with QEMU: `qemu-system-x86_64 -drive file=boot.bin,format=raw`.

This shows the bare minimum: a boot sector that gets executed by BIOS.

### 8.4 systemd Unit Example

A simple service unit (`/etc/systemd/system/hello.service`):

```
[Unit]
Description=Hello World Service
After=network.target

[Service]
ExecStart=/usr/bin/echo "Hello, systemd!"
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl enable hello.service
sudo systemctl start hello.service
sudo systemctl status hello.service
```

---

## 9. Summary

- **BIOS/UEFI** initializes hardware and loads the bootloader.
- **Bootloader (GRUB)** loads the kernel and initramfs.
- **Kernel** initializes core subsystems and executes the initramfs’s `/init`.
- **Initramfs** prepares the environment and mounts the real root.
- **systemd/init** takes over as PID 1, starts services, and presents a login prompt.

Each stage is designed to be robust and flexible, allowing for encrypted disks, network boot, and complex configurations. The boot process is the foundation on which the entire operating system is built.

_“The boot process is the journey from bare metal to a living system – a carefully orchestrated handoff where each player knows exactly what to do next.”_

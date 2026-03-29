## 1. The Problem: Running Multiple Isolated Environments

Imagine you have a powerful physical server. You want to run several applications, each with its own dependencies, libraries, and configuration. If you run them directly on the host OS, they might conflict (e.g., one needs Python 2, another Python 3). Worse, a bug in one could crash the whole system.

We need **isolation**—each environment thinks it has its own OS and resources, but they share the physical hardware. Virtualization and containers solve this, but with different trade‑offs.

**Analogy**:

- **Virtualization (VMs)** → Each application gets its own **house** (full OS, libraries, etc.) with separate foundations, walls, and utilities. Houses are built on top of land (hypervisor) that allocates land, electricity, water.
- **Containers** → Each application gets its own **room** in a shared apartment building. Rooms share the building’s foundation, plumbing, and electricity (the host kernel), but have their own walls and decorations (dependencies, filesystems). Much lighter.

---

## 2. Virtualization (Virtual Machines)

### 2.1 What Is a Virtual Machine?

A **virtual machine (VM)** is a software emulation of a physical computer. It runs a **guest OS** (like Linux, Windows) on top of **virtual hardware** (CPU, memory, disks, network). The **hypervisor** (Virtual Machine Monitor) manages these VMs and allocates physical resources.

**Two types of hypervisors**:

- **Type 1 (bare‑metal)**: Runs directly on hardware (e.g., VMware ESXi, Microsoft Hyper‑V, KVM). Provides strong isolation and high performance.
- **Type 2 (hosted)**: Runs as an application on a host OS (e.g., VirtualBox, VMware Workstation). Easier to use for development, but adds overhead.

### 2.2 How Virtualization Works

- **CPU virtualization**: The hypervisor schedules VMs’ virtual CPUs (vCPUs) on physical CPUs, often with hardware assistance (Intel VT‑x, AMD‑V) that reduces overhead.
- **Memory virtualization**: Guest physical memory is mapped to host physical memory via nested page tables (EPT/NPT).
- **I/O virtualization**: Paravirtualized drivers (virtio) or direct device assignment (VFIO) for performance.

**Example**: Using KVM on Linux (a Type‑1 hypervisor integrated into the kernel):

```bash
# Create a disk image
qemu-img create -f qcow2 ubuntu.qcow2 20G
# Install a VM
virt-install --name ubuntu-vm --ram 2048 --disk path=ubuntu.qcow2 \
  --vcpus 2 --os-type linux --os-variant ubuntu20.04 \
  --network bridge=virbr0 --graphics none --console pty,target_type=serial
```

Each VM gets its own kernel, init system, etc. Overhead is high (several GB per VM).

---

## 3. Containers: Lightweight Isolation

### 3.1 What Is a Container?

A container is a **lightweight, isolated environment** that shares the host OS kernel but has its own:

- Filesystem (root image)
- Process tree
- Network stack (optional)
- User IDs (mapped)
- Resource limits (CPU, memory)

Containers are built from **images** (read‑only templates) and run with **namespaces** and **cgroups** (control groups) in the Linux kernel.

### 3.2 The Kernel Primitives

**Namespaces** – Isolate global resources per process:

- `PID namespace`: processes inside see their own PID space (PID 1, etc.).
- `NET namespace`: own network interfaces, routing tables.
- `MNT namespace`: own mount points (filesystem view).
- `UTS namespace`: own hostname/domain.
- `IPC namespace`: own inter‑process communication resources.
- `USER namespace`: map non‑root UIDs inside container to unprivileged UIDs on host.
- `CGROUP namespace`: isolate cgroup view.

**Control groups (cgroups)** – Limit, account, and isolate resource usage (CPU, memory, disk I/O, network).

**Example: Creating a “toy container” with `unshare`** (Linux command):

```bash
# Create a new PID and MNT namespace, with a root filesystem
sudo unshare --pid --fork --mount --root=/path/to/rootfs /bin/bash
```

Inside, you’ll see a new mount namespace, and `ps aux` shows only processes inside. But it’s still using the host kernel.

### 3.3 Building a Simple Container with Namespaces and cgroups

Let’s write a Python script that creates an isolated process with its own root filesystem, network, and PID namespace:

```python
import os
import sys
import subprocess

# Unshare namespaces: clone with CLONE_NEWNS, CLONE_NEWPID, CLONE_NEWNET
def run_container(rootfs, cmd):
    # We need to fork with unshare flags; easiest: use 'unshare' command
    subprocess.run([
        "unshare", "--fork", "--pid", "--mount", "--net",
        "--root", rootfs,
        cmd
    ])

if __name__ == "__main__":
    run_container("/tmp/rootfs", "/bin/bash")
```

But for real containers, we use **Docker** (or podman, containerd).

### 3.4 Docker – The De Facto Container Engine

Docker adds **images**, **registries**, **layered filesystems (overlayfs)**, and a user‑friendly CLI.

**Example Docker workflow**:

```bash
# Pull an image
docker pull ubuntu:22.04

# Run a container
docker run -it --rm ubuntu:22.04 bash
```

Inside, you have an isolated environment, but the host kernel is used. Check with `ps` – you only see processes inside the container.

**Dockerfile** example:

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

Build and run:

```bash
docker build -t myapp .
docker run -p 8080:8080 myapp
```

---

## 4. Virtual Machines vs. Containers: Comparison

| Feature            | Virtual Machines                       | Containers                                 |
| ------------------ | -------------------------------------- | ------------------------------------------ |
| **Isolation**      | Strong (separate kernels)              | Shared kernel, but isolated via namespaces |
| **Startup time**   | Minutes                                | Milliseconds                               |
| **Disk footprint** | GBs (full OS)                          | MBs (only app and dependencies)            |
| **Performance**    | Near‑native (with hardware assist)     | Native (minimal overhead)                  |
| **Security**       | Stronger (hypervisor isolation)        | Weaker if kernel compromised               |
| **Use case**       | Running different OS, strong isolation | Microservices, dev/prod parity             |
| **Example**        | VMware, KVM, Hyper‑V                   | Docker, Podman, LXC                        |

**Analogy**:

- VMs are like **apartment buildings** with separate apartments (each with its own lock, walls, utilities).
- Containers are like **shared houses** with separate rooms, but the house’s foundation, plumbing, and electricity are shared.

---

## 5. Advanced Container Concepts

### 5.1 Orchestration (Kubernetes)

When you run hundreds of containers across many machines, you need an orchestrator. **Kubernetes** manages:

- Scheduling containers on nodes
- Service discovery
- Load balancing
- Auto‑scaling
- Rolling updates

**Example**: `kubectl run nginx --image=nginx` creates a pod (group of containers) running nginx.

### 5.2 Rootless Containers

For security, you can run containers without root privileges. Podman (Red Hat’s alternative to Docker) runs rootless by default, using user namespaces to map container root to a non‑privileged host UID.

```bash
podman run -it --rm alpine
```

### 5.3 OCI Standards

Open Container Initiative (OCI) defines specifications for container images and runtimes. Docker images are OCI‑compliant; runtimes like runc (used by Docker) implement the OCI runtime spec.

### 5.4 Unikernels

A more extreme approach: compile an application with a minimal kernel into a single VM image. Combines security of VMs with small footprint, but lacks flexibility.

---

## 6. Under the Hood: How Docker Uses Linux Kernel Features

When you run `docker run`, Docker:

1. Pulls the image layers (using overlayfs to combine them into a single rootfs).
2. Creates a new set of namespaces (PID, NET, MNT, UTS, IPC, USER) via `clone()` with flags.
3. Sets up cgroups to limit CPU, memory, etc.
4. Configures network (e.g., a veth pair to a bridge) and mounts the rootfs.
5. Executes the entrypoint process inside the namespace.

You can inspect the namespaces of a running container with `lsns`:

```bash
sudo lsns -t pid | grep docker
```

And cgroups:

```bash
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

---

## 7. Code Example: Minimal Container Using `unshare` and `chroot`

Let’s build a minimal “container” manually to see the pieces:

```bash
# Create a root filesystem
mkdir -p /tmp/mycontainer/rootfs
cd /tmp/mycontainer/rootfs

# Use busybox as a tiny userland
wget https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
chmod +x busybox
mkdir -p bin
cp busybox bin/sh
ln -s /bin/sh bin/ls  # optional

# Set up basic filesystem structure
mkdir -p proc sys dev

# Run unshare to create new namespaces and chroot into the rootfs
sudo unshare --mount --pid --fork --net --uts --ipc \
  --mount-proc --root /tmp/mycontainer/rootfs /bin/sh
```

Inside this “container”:

- `/proc` is mounted (thanks to `--mount-proc`).
- You have a separate process tree (PID 1 is the shell).
- Network is isolated (`ip link` shows only loopback).

This is essentially what LXC or Docker does, but without the image management and convenience.

---

## 8. Security Considerations

- **VMs** provide stronger isolation because each VM has its own kernel. A kernel exploit in one VM does not affect others.
- **Containers** share the host kernel, so a kernel exploit can compromise all containers and the host. Mitigations: **seccomp** (syscall filtering), **AppArmor/SELinux**, **capabilities dropping**, **rootless mode**.
- **User namespaces** map container root to an unprivileged host UID, reducing the risk if the container is compromised.

**Example**: Docker’s default seccomp profile blocks ~44 syscalls.

---

## 9. Summary

- **Virtualization** uses hypervisors to run full OS instances, providing strong isolation but with higher overhead.
- **Containers** use kernel namespaces and cgroups to isolate processes while sharing the host kernel, offering lightweight, fast, portable environments.
- **Docker** popularized containers; **Kubernetes** orchestrates them at scale.
- Understanding the underlying OS mechanisms (namespaces, cgroups, overlayfs) helps you debug, secure, and build your own container solutions.

_“Virtualization abstracts hardware; containers abstract the OS—both are tools for building isolated, manageable, and scalable systems.”_

## 1. The Linux Storage Stack: A Layered View

From the bottom (hardware) to the top (applications):

```
Application (user space)
        ↓
Filesystem (ext4, XFS, etc.)
        ↓
Block Layer (request queue, I/O scheduler)
        ↓
Device Mapper (LVM, dm-crypt, RAID, etc.)
        ↓
Block Devices (sdX, nvmeXnY)
        ↓
Hardware (disk, SSD, NVMe)
```

Each layer adds functionality: filesystems provide structure; block layer schedules I/O; device mapper enables stacking (e.g., encryption on top of RAID on top of disks).  
**RAID** and **LVM** are implemented in the device mapper layer (or, in the case of software RAID, also via `md` (multiple device) driver, which is similar to device mapper but older).

**Analogy**: The storage stack is like a **building foundation**—you pour concrete (disks), add rebar (RAID for resilience), pour another layer (LVM for flexibility), and finally lay the floor (filesystem) where you place furniture (applications).

---

## 2. RAID (Redundant Array of Independent Disks)

RAID combines multiple physical disks into one logical unit for **performance**, **redundancy**, or both.

### 2.1 Common RAID Levels

| Level       | Description                      | Minimum Disks | Pros                                            | Cons                                             |
| ----------- | -------------------------------- | ------------- | ----------------------------------------------- | ------------------------------------------------ |
| **RAID 0**  | Striping                         | 2             | High performance, full capacity                 | No redundancy; one disk fails → all data lost    |
| **RAID 1**  | Mirroring                        | 2             | Full redundancy, good read performance          | Write penalty, 50% capacity                      |
| **RAID 5**  | Striping with distributed parity | 3             | Good read performance, efficient capacity (n‑1) | Write penalty; can only survive one disk failure |
| **RAID 6**  | Striping with double parity      | 4             | Survives two disk failures                      | Higher write penalty, more parity                |
| **RAID 10** | Mirrored stripes                 | 4             | Performance + redundancy; fast rebuilds         | 50% capacity; needs even number of disks         |

**Analogy**:

- RAID 0 is like splitting a large file across two workers (fast, but if one worker quits, you lose everything).
- RAID 1 is two workers doing the same task in parallel—if one fails, the other continues.

### 2.2 Software RAID with `mdadm`

Linux software RAID uses the **md (multiple device)** driver. You can create a RAID array with `mdadm`.

**Example: Create a RAID 1 mirror of two disks (`/dev/sdb`, `/dev/sdc`)**:

```bash
# Create RAID 1
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc

# Check status
cat /proc/mdstat
mdadm --detail /dev/md0

# Create filesystem
mkfs.ext4 /dev/md0
mount /dev/md0 /mnt/raid
```

**Example: RAID 10 (four disks)**:

```bash
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e]
```

### 2.3 Hardware RAID

Hardware RAID uses a dedicated controller (e.g., LSI, HP SmartArray). The OS sees a single logical drive (`/dev/sda`). It’s managed by the controller’s BIOS. No software stack in the OS, but you lose visibility of individual disks.

---

## 3. LVM (Logical Volume Manager)

LVM adds a layer of indirection between physical disks and filesystems, allowing:

- **Resizing** logical volumes online.
- **Snapshots** (for backups, test environments).
- **Striping, mirroring** (though RAID is often used below LVM).
- **Pooling** physical storage into a single pool.

### 3.1 LVM Components

- **Physical Volumes (PVs)** – disk partitions or whole disks (e.g., `/dev/sda1`).
- **Volume Groups (VGs)** – collection of PVs (like a storage pool).
- **Logical Volumes (LVs)** – virtual partitions carved from the VG.

### 3.2 Basic LVM Commands

```bash
# Create physical volumes
pvcreate /dev/sdb /dev/sdc

# Create volume group
vgcreate my_vg /dev/sdb /dev/sdc

# Create logical volume (10 GB)
lvcreate -L 10G -n my_lv my_vg

# Create filesystem
mkfs.ext4 /dev/my_vg/my_lv
mount /dev/my_vg/my_lv /mnt/lvm

# Extend logical volume (online)
lvextend -L +5G /dev/my_vg/my_lv
resize2fs /dev/my_vg/my_lv   # for ext4

# Snapshot
lvcreate -L 1G -s -n snap /dev/my_vg/my_lv
```

**Analogy**: LVM is like a **warehouse manager**—you can rent space in different buildings (PVs) to create a flexible storage unit (LV) that you can expand or shrink as needed, and take temporary snapshots for backup.

---

## 4. Device Mapper (dm) – The Kernel Building Block

Underneath LVM, RAID (`md`), and even `dm-crypt` (disk encryption) lies the **device mapper** kernel subsystem. It creates virtual block devices by mapping regions of underlying devices using a simple language of **targets**.

### 4.1 Device Mapper Concepts

- **Mapping table**: defines how each logical sector maps to physical sectors on underlying devices.
- **Target types**:
  - `linear` – maps a contiguous range to a linear range on another device.
  - `striped` – RAID 0.
  - `mirror` – RAID 1.
  - `crypt` – encryption.
  - `snapshot` – copy‑on‑write snapshots.
  - `raid` – more complex RAID levels (5,6,10) via the `dm-raid` target.

### 4.2 Inspecting Device Mapper with `dmsetup`

```bash
# List all device mapper devices
dmsetup ls

# Show table for a device (e.g., LVM logical volume)
dmsetup table my_vg-my_lv
# Output example: 0 20971520 linear 8:16 2048
# Means: starts at sector 0, length 20971520 sectors, linear target, major:minor 8:16 (underlying device), offset 2048 sectors.

# Create a simple linear mapping manually
echo "0 1000000 linear /dev/sdb 0" | dmsetup create my_dm
```

**Example: stacking LVM on top of RAID**:  
You can create a RAID 1 with `mdadm` (`/dev/md0`), then use that as a physical volume in LVM. The device mapper sees both layers:  
`/dev/mapper/my_vg-my_lv` (dm device) → maps to `(8,16)` which is `/dev/sdb`? Actually, LVM’s underlying device could be a RAID array (`/dev/md0`). The stack is transparent.

---

## 5. How Layers Stack: A Real Scenario

Let’s build a multi‑layer storage stack:

1. **Physical disks**: `/dev/sda`, `/dev/sdb`, `/dev/sdc`, `/dev/sdd`.
2. **RAID 10** (using `mdadm`): combine them into `/dev/md0`.
3. **LVM** on top of `/dev/md0`: create PV, VG, and LVs.
4. **Filesystem** (ext4) on an LV.
5. **Application** writes to a file.

**Commands**:

```bash
# Step 1: Create RAID 10 (four disks)
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[a-d]

# Step 2: Create LVM on /dev/md0
pvcreate /dev/md0
vgcreate data_vg /dev/md0
lvcreate -L 100G -n data_lv data_vg

# Step 3: Filesystem
mkfs.ext4 /dev/data_vg/data_lv
mount /dev/data_vg/data_lv /data
```

Now, an I/O request from the application goes:

1. **Filesystem** (ext4) → translates file offset to logical block number (within the LV).
2. **Block layer** → splits into bios, submits to `/dev/data_vg/data_lv`.
3. **Device mapper (LVM)** → maps the LV’s logical sectors to sectors on the underlying device (`/dev/md0`).
4. **Device mapper (md)** → maps RAID logical sectors to physical sectors on `/dev/sda`, `/dev/sdb`, etc., and handles parity/mirroring.
5. **Block device drivers** → send commands to hardware.

**Inspection**:

```bash
# See the device mapper stack
lsblk
# Output:
# sda                    8:0    0 100G  0 disk
# └─md0                  9:0    0 200G  0 raid10
#   └─data_vg-data_lv  253:0    0 100G  0 lvm
#     └─/data             ext4   ...

# Show dm relationships
dmsetup deps
# data_vg-data_lv: 1 dependencies (9:0) → md0
# md0: 4 dependencies (8:0,8:16,8:32,8:48)
```

---

## 6. Advanced: Device Mapper Target Examples

### 6.1 Linear Target (Concatenation)

```bash
# Map a 1GB device from two disks: first 500M from /dev/sdb, next 500M from /dev/sdc
echo "0 1024000 linear /dev/sdb 0
1024000 1024000 linear /dev/sdc 0" | dmsetup create concat
```

### 6.2 Snapshot Target

Used by LVM snapshots. A snapshot LV is a dm‑snapshot device with a **origin** (original LV) and a **cow** (copy‑on‑write store).

```bash
# LVM does this internally:
lvcreate -s -L 1G -n snap /dev/vg/origin
# Under the hood: dmsetup table shows:
# snap: 0 2097152 snapshot /dev/vg/origin /dev/vg/snap-cow P 16
```

### 6.3 dm-crypt (LUKS)

Encryption layer on top of a block device:

```bash
cryptsetup luksFormat /dev/sdb
cryptsetup open /dev/sdb cryptdisk
# Now /dev/mapper/cryptdisk is a decrypted block device
```

---

## 7. Performance Considerations

- **RAID 5/6 have write penalty**: each write requires reading parity (or full stripe) and writing data+parity.
- **LVM snapshots** use copy‑on‑write; performance degrades as snapshot fills.
- **Stacking layers** adds overhead (extra I/O processing). For high‑performance workloads, consider placing filesystem directly on RAID (or using hardware RAID).
- **I/O schedulers** at the block layer (e.g., mq‑deadline, kyber) can be tuned for rotational or SSD media.

**Tools for monitoring**:

- `iostat -x 1` – see per‑device utilization.
- `dstat` – combined stats.
- `blktrace` / `btt` – detailed I/O tracing through the stack.

---

## 8. Summary

- **RAID** provides performance and redundancy via striping/mirroring; managed by `mdadm` (software) or hardware controllers.
- **LVM** adds flexible volume management: resizing, snapshots, pooling.
- **Device Mapper** is the kernel’s low‑level framework that implements LVM, RAID (`dm-raid`), encryption, and more.
- Layers can be stacked (e.g., RAID → LVM → filesystem), each adding abstraction.
- The block layer coordinates I/O between these layers and the hardware.

_“The storage stack is like a set of Russian dolls—each layer wraps the one below, adding new capabilities while hiding complexity from the one above.”_

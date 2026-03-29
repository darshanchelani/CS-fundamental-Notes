## 1. What Problem Does a File System Solve?

Imagine you have a **blank hard disk**—just a sequence of sectors (blocks) that can be read or written. Without a file system, you’d have to remember:

- Which blocks belong to your photo?
- Which blocks belong to your text file?
- How long each file is?
- Where to put new data without overwriting existing data?

A **file system** organizes these blocks into:

- **Files** – named, contiguous (or indexed) sequences of data.
- **Directories** – mappings from names to files/subdirectories.
- **Metadata** – size, timestamps, permissions, etc.
- **Free space management** – knowing which blocks are unused.

**Analogy**: A **library** with books (files), a card catalog (directory), and a librarian (file system) who knows where each book is kept.

---

## 2. The Two‑Level View: Logical vs Physical

From an application’s perspective, the file system provides a **logical view**:

```
/home/user/notes.txt
```

From the OS’s perspective, that path translates to a **set of disk blocks** (physical view). The file system is the translator.

---

## 3. Core Components of a Simple File System

Most classic simple file systems (like the early Unix File System or a simple educational FS) are built from these on‑disk structures:

### 3.1 Boot Block (Optional)

The first sector(s) contain code to bootstrap the OS. Not strictly part of the file system but part of the volume.

### 3.2 Superblock

A single block (or a few) that describes the file system’s parameters:

- Size of the file system (total blocks)
- Block size (e.g., 4 KB)
- Number of inodes
- Location of inode table
- Location of free space map
- Magic number (to identify the FS type)

**Analogy**: The library’s **master blueprint** – total shelf space, number of shelves, where the catalog is located.

### 3.3 Inode Table

An **inode** (index node) stores all metadata about a file **except its name**:

- File type (regular, directory, symlink, etc.)
- Size (in bytes)
- Timestamps (creation, modification, access)
- Permissions (owner, group, r/w/x)
- Pointers to data blocks (direct pointers, indirect pointers)

Each inode has a unique number (inode number). The inode table is a contiguous array of inodes.

**Analogy**: A **book’s library card** – tells you the title’s details, where it’s stored, but the card itself sits in a drawer (the inode table).

### 3.4 Data Blocks

Actual file content and directory contents. For small files, the inode points directly to a few data blocks. For larger files, it uses **indirect blocks** (blocks that point to other blocks).

### 3.5 Directory Structure

A directory is a **special file** that maps file names to inode numbers.

In a simple file system, a directory might be an array of fixed‑size entries:

```
struct dir_entry {
    int inode;          // inode number (0 = free)
    char name[16];      // file name
};
```

When you look up a name, the OS scans the directory file for that name, finds the inode number, then reads the inode to access data.

### 3.6 Free Space Management

The file system must track which blocks are free. Simple approaches:

- **Free bitmap**: one bit per block (1 = free, 0 = used).
- **Free list**: linked list of free blocks (but slower).

---

## 4. A Walk‑Through: How a File Is Read

When you call `read(fd, buf, size)`:

1. The file descriptor (`fd`) is mapped to a `struct file` in the OS, which holds:
   - The inode pointer
   - Current file position (offset)
2. The OS computes which logical block number the offset falls into (e.g., offset / block_size).
3. It looks at the inode’s block pointers to find the disk block number.
4. It issues a read request to the block device driver to fetch that block.
5. Data is copied from the block into user space.

**Code snippet (simplified conceptual)**:

```c
// Pseudocode: reading from a file
int read_file(inode_t *inode, int offset, char *buf, int size) {
    int block_idx = offset / BLOCK_SIZE;
    int block_offset = offset % BLOCK_SIZE;

    // Get disk block number from inode's block pointers
    int disk_block = get_block_pointer(inode, block_idx);

    // Read the whole block from disk
    char block[BLOCK_SIZE];
    read_disk_block(disk_block, block);

    // Copy requested part to user buffer
    int bytes_to_copy = min(size, BLOCK_SIZE - block_offset);
    memcpy(buf, block + block_offset, bytes_to_copy);

    return bytes_to_copy;
}
```

---

## 5. A Toy Simple File System in Memory (Code Demo)

Let’s build a **very simple in‑memory file system** to see the concepts in action.  
We’ll implement a minimal structure: a superblock, an inode table, and a data region.

```python
import struct

BLOCK_SIZE = 1024
NUM_INODES = 32
NUM_BLOCKS = 128

# Simulate disk as a list of blocks (each block is a bytearray)
disk = [bytearray(BLOCK_SIZE) for _ in range(NUM_BLOCKS)]

class Superblock:
    def __init__(self):
        self.magic = 0x46494C45      # "FILE" magic
        self.block_size = BLOCK_SIZE
        self.num_inodes = NUM_INODES
        self.num_blocks = NUM_BLOCKS
        self.inode_table_start = 1   # block 1 (block 0 is superblock)
        self.data_start = 2           # blocks 2..NUM_BLOCKS-1

    def write_to_disk(self):
        # Pack superblock into block 0
        fmt = "IIIIII"
        data = struct.pack(fmt, self.magic, self.block_size,
                           self.num_inodes, self.num_blocks,
                           self.inode_table_start, self.data_start)
        disk[0][:len(data)] = data

    @staticmethod
    def read_from_disk():
        sb = Superblock()
        fmt = "IIIIII"
        size = struct.calcsize(fmt)
        data = bytes(disk[0][:size])
        (sb.magic, sb.block_size, sb.num_inodes,
         sb.num_blocks, sb.inode_table_start, sb.data_start) = struct.unpack(fmt, data)
        return sb

class Inode:
    FORMAT = "IIIIII"  # mode, size, direct[0..2] (3 pointers), indirect
    SIZE = struct.calcsize(FORMAT)

    def __init__(self, mode=0, size=0):
        self.mode = mode        # file type + permissions
        self.size = size
        self.direct = [0, 0, 0]  # pointers to data blocks (max 3 blocks)
        self.indirect = 0         # indirect block pointer

    def write_to_block(self, block_data):
        # Pack into bytes
        data = struct.pack(self.FORMAT, self.mode, self.size,
                           self.direct[0], self.direct[1], self.direct[2],
                           self.indirect)
        block_data[:len(data)] = data

    @staticmethod
    def read_from_block(block_data):
        inode = Inode()
        (inode.mode, inode.size,
         inode.direct[0], inode.direct[1], inode.direct[2],
         inode.indirect) = struct.unpack(Inode.FORMAT, block_data[:Inode.SIZE])
        return inode

# Initialize the file system (format)
def format_fs():
    sb = Superblock()
    sb.write_to_disk()
    # Zero out inode table (all inodes empty)
    inode_table_start = sb.inode_table_start
    inode_bytes = Inode.SIZE * NUM_INODES
    num_blocks_for_inodes = (inode_bytes + BLOCK_SIZE - 1) // BLOCK_SIZE
    for i in range(num_blocks_for_inodes):
        disk[inode_table_start + i] = bytearray(BLOCK_SIZE)

    # Initialize free block bitmap (we'll use a simple free list for demo)
    # For simplicity, we'll just keep a free set in memory.
    # In a real FS, this would be on disk.

def create_file(path, content):
    # This is a huge simplification: we'll ignore directories and just
    # create a new inode with a name stored somewhere.
    # Real FS would do name lookup, allocate inode, allocate blocks.
    pass

# This is a conceptual demo; the real value is understanding the layout.
```

This toy shows how metadata and data are separated. In a real file system, directories map names to inode numbers, and you’d have to traverse the directory structure to resolve a path.

---

## 6. Directory Implementation (Simple)

A directory is just a file containing a list of entries. For example:

```c
struct dir_entry {
    int inode;       // 0 means empty
    char name[28];   // fixed length for simplicity
};
```

When you open `/home/user/notes.txt`:

1. Read the root directory (inode #2 usually) to find "home".
2. That entry gives the inode for `/home`.
3. Read `/home` directory to find "user".
4. Read `/home/user` directory to find "notes.txt".
5. That entry gives the inode for the file.

**Code simulation (Python)**:

```python
def lookup(path, root_inode):
    parts = path.strip('/').split('/')
    current_inode = root_inode
    for name in parts:
        # read directory file's content
        dir_blocks = get_data_blocks(current_inode)
        dir_data = b''.join(disk[block] for block in dir_blocks)
        # parse dir entries
        found = False
        for entry in parse_dir_entries(dir_data):
            if entry.name == name:
                current_inode = entry.inode
                found = True
                break
        if not found:
            raise FileNotFoundError
    return current_inode
```

---

## 7. Simple File System Operations – System Calls

From a process’s perspective, file system operations are done via system calls:

- `open()` – translates path to inode, returns file descriptor.
- `read()` – uses inode’s block pointers to fetch data.
- `write()` – may allocate new blocks and update inode.
- `close()` – flushes metadata to disk.
- `mkdir()` – creates a new directory file (allocates inode and initializes directory entries).

**Analogy**: The system calls are the **library request forms**. The file system (librarian) does all the behind‑the‑scenes work to find the book and bring it to you.

---

## 8. Trade‑Offs in Simple File Systems

- **Simplicity**: Easy to implement and understand.
- **Limitations**:
  - **Fixed‑size structures** – inode table size fixed at format time; cannot grow later.
  - **Single‑level indirect** – limits maximum file size.
  - **Poor performance** for random access in large files.
  - **No crash consistency** – if power fails during a write, the FS can become corrupted (needs `fsck`).
  - **No journaling** – recovery is slow and may lose data.

These limitations led to more advanced designs: **ext2/ext3/ext4** (Linux), **NTFS**, **ZFS**, etc., which add features like:

- **Dynamic inodes**
- **Extents** (instead of indirect blocks)
- **Journaling / copy‑on‑write**
- **B‑trees for directories**
- **Snapshots and checksums**

---

## 9. Interaction with Other OS Subsystems

- **Process management**: When a process calls `open()`, the OS creates a file descriptor and adds it to the process’s file descriptor table. When a process `fork()`s, the child inherits all file descriptors (shared file offsets).
- **CPU scheduling**: A process waiting for a disk read (e.g., `read()` on a file) is moved to the waiting queue, letting the scheduler run other processes. When the I/O completes, the process becomes ready.
- **Memory management**: The file system uses **buffer cache** or **page cache** to keep frequently accessed blocks in RAM, avoiding repeated disk reads.

---

## 10. Real‑World Example: Reading a File in Linux

When you run `cat myfile.txt`, the following happens:

1. Shell calls `fork()` + `exec()` to start `cat`.
2. `cat` calls `open("myfile.txt", O_RDONLY)`.
3. The OS uses the **VFS** (Virtual File System) to dispatch to the appropriate file system (e.g., ext4).
4. ext4 looks up the path, finds the inode, and returns a file descriptor.
5. `cat` calls `read()` repeatedly.
6. The file system converts the logical offset to a disk block, finds it in the page cache or reads from disk.
7. The data is copied to user space.
8. When `cat` exits, the OS closes the file and cleans up.

---

## 11. Summary

- A **file system** turns a raw block device into a structured, named storage.
- **Simple file systems** use a **superblock**, **inode table**, **data blocks**, and **free space management**.
- **Directories** are files that map names to inode numbers.
- **System calls** like `open`, `read`, `write` provide the interface.
- **Limitations** of simple FS led to modern designs with better performance, scalability, and reliability.

Understanding simple file systems gives you the foundation to grasp how modern file systems work. When you later deal with databases, distributed storage, or even `git`, you’ll appreciate the elegance and trade‑offs of organizing data on disk.

_“A file system is the persistent memory of the operating system – it outlives processes, survives reboots, and keeps your data alive.”_

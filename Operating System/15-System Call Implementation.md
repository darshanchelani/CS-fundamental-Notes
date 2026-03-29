## 1. What Is a System Call?

A **system call** is a controlled entry point into the kernel, allowing a user process to request a privileged operation (e.g., read a file, create a process, allocate memory).

**Why not just jump to kernel code?**

- The CPU must switch from **user mode** to **kernel mode** (privilege level change).
- The kernel must verify that the request is safe (arguments, permissions).
- The kernel must return results without exposing internal state.

System calls provide a **well‑defined, safe interface** between user applications and the kernel.

**Analogy**: A system call is like **calling the front desk** in a secured building. You don’t have the master key; you give your request through a small window, and the staff (kernel) performs the task on your behalf.

---

## 2. The Journey of a System Call

Let’s trace a simple `write(1, "hello\n", 6)` call on Linux/x86_64. The steps:

1. **User‑space wrapper** (libc’s `write` function)
2. **Assemble arguments and trigger the trap**
3. **CPU switches to kernel mode, saves context**
4. **Kernel dispatches to the correct handler via syscall table**
5. **Handler runs with arguments, does the work**
6. **Return value is placed, CPU returns to user mode**
7. **User‑space wrapper returns result to program**

---

## 3. Hardware Support: From User to Kernel

The CPU provides dedicated instructions to enter and exit kernel mode safely.

### 3.1 Legacy: `int 0x80` (x86)

Used in 32‑bit Linux. The user program triggers software interrupt 0x80; the kernel’s interrupt handler reads the syscall number and arguments from registers.

### 3.2 Modern: `syscall` / `sysret` (x86_64)

The `syscall` instruction:

- Switches to kernel mode (privilege level 0).
- Saves the return address in `rcx` and the old flags in `r11`.
- Jumps to a predetermined kernel entry point (stored in the `IA32_LSTAR` MSR).

`sysret` reverses the process: returns to user mode, restores flags, and jumps to the saved address.

**AMD64 calling convention**:

- Syscall number in `rax`.
- Arguments in `rdi`, `rsi`, `rdx`, `r10`, `r8`, `r9` (note: `rcx` and `r11` are clobbered).
- Return value in `rax`.
- On error, `rax` contains a negative value (`-errno`).

---

## 4. Kernel Side: The Syscall Table

The kernel maintains a **syscall table**—an array of function pointers indexed by the syscall number. On Linux, it’s defined in `arch/x86/entry/syscall_64.c`:

```c
asmlinkage const sys_call_ptr_t sys_call_table[] = {
    [0] = sys_read,
    [1] = sys_write,
    [2] = sys_open,
    ...
    [57] = sys_fork,
    ...
};
```

Each entry is a kernel function with a specific signature, e.g.:

```c
asmlinkage long sys_write(unsigned int fd, const char __user *buf, size_t count);
```

**`asmlinkage`** tells the compiler to expect arguments on the stack (or in registers, depending on ABI). The `__user` annotation reminds developers that the pointer is from user space and must be accessed with special copy functions.

### 4.1 Syscall Entry Point

When the CPU executes `syscall`, it jumps to an entry point defined in assembly. For x86_64, it’s `entry_SYSCALL_64` in `arch/x86/entry/entry_64.S`.

This assembly code:

- Saves registers to the kernel stack.
- Switches to the kernel stack.
- Loads the syscall table pointer.
- Calls `do_syscall_64`, which looks up the syscall number in the table and calls the corresponding function.

**Simplified pseudocode**:

```c
long do_syscall_64(unsigned long nr, struct pt_regs *regs) {
    if (nr >= NR_syscalls)
        return -ENOSYS;
    return sys_call_table[nr](regs->rdi, regs->rsi, regs->rdx,
                              regs->r10, regs->r8, regs->r9);
}
```

`struct pt_regs` holds the saved user registers (including the arguments).

---

## 5. Argument Passing and Validation

The kernel receives arguments directly from the user’s registers. But these arguments are **unsafe**:

- They are pointers into user space; the kernel must **not** dereference them directly, because the user could have passed a kernel pointer, or the page might be swapped out, or the user could modify the memory concurrently.
- The kernel must **copy** data to/from user space using dedicated functions: `copy_from_user`, `copy_to_user`, `get_user`, `put_user`.

**Example: simplified `sys_write`**:

```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count) {
    struct file *file;
    ssize_t ret;
    char *kernel_buf;

    // 1. Validate file descriptor
    file = fget(fd);
    if (!file)
        return -EBADF;

    // 2. Allocate kernel buffer (if needed)
    kernel_buf = kmalloc(count, GFP_KERNEL);
    if (!kernel_buf) {
        fput(file);
        return -ENOMEM;
    }

    // 3. Copy data from user space
    if (copy_from_user(kernel_buf, buf, count)) {
        kfree(kernel_buf);
        fput(file);
        return -EFAULT;   // bad address
    }

    // 4. Perform the operation (e.g., vfs_write)
    ret = vfs_write(file, kernel_buf, count, &file->f_pos);

    // 5. Cleanup and return
    kfree(kernel_buf);
    fput(file);
    return ret;
}
```

The `SYSCALL_DEFINE3` macro expands to `asmlinkage long sys_write(...)` with proper argument type checking.

---

## 6. Returning to User Space

After the syscall handler finishes, the return value is placed in `rax`. The kernel then invokes the `sysret` instruction, which:

- Restores the saved user‑mode state (including `rip` and `rflags`).
- Switches to user mode (privilege level 3).
- Jumps to the instruction after the original `syscall` in the user program.

The user‑space wrapper receives the return value (or error code) and sets `errno` accordingly if the value is negative.

---

## 7. Example: A Minimal Assembly System Call

Let’s write a tiny program that invokes `write` directly without libc, using assembly.

**syscall.asm** (nasm syntax, x86_64):

```asm
section .data
    msg db 'Hello, syscall!', 10   ; '10' is newline
    len equ $ - msg

section .text
    global _start
_start:
    ; write(1, msg, len)
    mov rax, 1          ; syscall number for write
    mov rdi, 1          ; fd = stdout
    mov rsi, msg        ; buffer
    mov rdx, len        ; count
    syscall

    ; exit(0)
    mov rax, 60         ; syscall number for exit
    xor rdi, rdi        ; status = 0
    syscall
```

Assemble and run:

```bash
nasm -f elf64 syscall.asm -o syscall.o
ld syscall.o -o syscall
./syscall
```

**Output**: `Hello, syscall!`

You can trace the syscalls with `strace ./syscall`. You’ll see `write(1, ...)` and `exit(0)`.

---

## 8. Observing Syscalls with strace

`strace` is a powerful tool that uses `ptrace` to intercept and display every syscall a program makes.

```bash
strace ls
```

You’ll see:

```
execve("/bin/ls", ["ls"], 0x7ffd...) = 0
brk(NULL)                               = 0x55d...
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT
...
write(1, "file1  file2\n", 14)          = 14
...
exit_group(0)                           = ?
```

This is the actual communication between user program and kernel.

---

## 9. Adding a New System Call (for Learning)

To understand the full flow, you can add a trivial syscall to the Linux kernel (on a test VM). The steps (simplified):

1. **Add the syscall number** in `include/uapi/asm-generic/unistd.h`:

   ```c
   #define __NR_hello 451
   __SYSCALL(__NR_hello, sys_hello)
   ```

2. **Implement the function** in `kernel/sys.c`:

   ```c
   SYSCALL_DEFINE0(hello) {
       printk("Hello from syscall!\n");
       return 0;
   }
   ```

3. **Add the entry** to the syscall table (architecture‑dependent). For x86_64, in `arch/x86/entry/syscall_64.c`:

   ```c
   [451] = sys_hello,
   ```

4. **Rebuild the kernel**, boot, and test with a user program:

   ```c
   #include <unistd.h>
   #include <sys/syscall.h>

   int main() {
       syscall(451);
       return 0;
   }
   ```

The kernel will print “Hello from syscall!” to the kernel log (view with `dmesg`).

---

## 10. Security and Safety Mechanisms

- **Copying user data** with `copy_from_user` ensures that the kernel never blindly dereferences user pointers. The function checks that the entire range is in user space and handles page faults.
- **Validating arguments**: The kernel checks file descriptors, buffer lengths, permissions, etc., before acting.
- **Syscall filtering (seccomp)**: A process can install a filter (eBPF) that restricts which syscalls it can call. Used by container runtimes to reduce attack surface.
- **Control flow integrity**: Modern kernels use features like `CONFIG_RETPOLINE` and kernel address space layout randomization (KASLR) to mitigate exploits that abuse syscall entry points.

---

## 11. Summary

- **System calls** are the **only way** for user‑mode processes to access kernel‑controlled resources.
- **Hardware** provides dedicated instructions (`syscall`/`sysret`) to switch privilege levels efficiently.
- The kernel maintains a **syscall table** mapping numbers to handler functions.
- **Argument passing** uses CPU registers; arguments are **validated and copied** between user and kernel space.
- **Return** to user mode uses `sysret`, with the return value in `rax`.
- Tools like `strace` show syscalls in action; you can even **add your own syscall** to the kernel.

_“The system call interface is the contract between applications and the operating system—a carefully guarded bridge between two worlds.”_

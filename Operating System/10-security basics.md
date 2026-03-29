## 1. Why Access Control Matters

An operating system manages shared resources: files, devices, memory, processes, network ports. Without access control, any process could read any file, kill any process, or reconfigure the system.

**Analogy**: A building with many rooms (resources). Access control is the **security system**: badges (credentials), locks (permissions), and security guards (OS kernel) that enforce who can enter which room, and what they can do inside.

The core components:

- **Authentication** – proving identity (e.g., login, password, key).
- **Authorization** – determining what an authenticated user/process is allowed to do.
- **Auditing** – logging what happened.

We’ll focus on authorization—the **access control** layer.

---

## 2. Access Control Models

### 2.1 Discretionary Access Control (DAC)

- The owner of a resource decides who can access it.
- Example: Unix file permissions (`chmod`, `chown`). The file owner can grant or revoke permissions at will.
- **Pros**: Flexible, easy to understand.
- **Cons**: Users may accidentally expose sensitive data; malware running with user privileges can modify permissions.

### 2.2 Mandatory Access Control (MAC)

- A system‑wide policy (defined by the OS or administrator) overrides user decisions.
- Users cannot bypass the policy, even if they own the resource.
- Examples: SELinux (Linux), AppArmor, SMACK.
- **Pros**: Strong security, containment of malware.
- **Cons**: Complexity, requires careful policy configuration.

### 2.3 Role‑Based Access Control (RBAC)

- Permissions are assigned to **roles**, and users are assigned to roles.
- Common in enterprise systems and databases.
- Example: `admin`, `developer`, `auditor` roles. A user in the “developer” role can modify code but not deploy to production.

Modern OSes combine these models. Unix DAC is the base, with MAC frameworks layered on top.

---

## 3. Unix File Permissions – The Classic DAC

### 3.1 The Model

Every file and directory has:

- **Owner** (a user)
- **Group** (a group)
- **Permissions** for three categories: **owner**, **group**, **others**

Permissions are three bits: **read (r)**, **write (w)**, **execute (x)**.

For directories, `x` means “traverse” (enter the directory), `r` means “list contents”, `w` means “create/delete entries”.

### 3.2 Viewing and Changing Permissions

```bash
$ ls -l secret.txt
-rw-r--r-- 1 alice staff 1024 Mar 28 10:00 secret.txt
```

Interpretation:

- `-` (regular file), `rw-` (owner: read+write), `r--` (group: read), `r--` (others: read).

Change permissions with `chmod`:

```bash
chmod 600 secret.txt   # owner: rw-; group/others: ---
chmod u+x script.sh    # add execute for owner
```

Change owner/group with `chown` and `chgrp`:

```bash
chown bob:staff secret.txt
```

### 3.3 Setuid, Setgid, Sticky Bit

- **setuid** (s on owner execute): When a user runs the executable, the process runs with the file owner’s privileges. Used for programs like `passwd` (needs root to modify `/etc/shadow`).
- **setgid** (s on group execute): Process runs with group of the file; on directories, new files inherit the directory’s group.
- **sticky bit** (t on others): On directories, only the file owner can delete files inside (e.g., `/tmp`).

**Example**: Set the setuid bit:

```bash
chmod u+s myprogram
```

**Code snippet – checking and setting permissions in C**:

```c
#include <sys/stat.h>
#include <stdio.h>

int main() {
    struct stat st;
    stat("file.txt", &st);
    if (st.st_mode & S_IRUSR) printf("Owner can read\n");
    // set group write bit
    chmod("file.txt", st.st_mode | S_IWGRP);
    return 0;
}
```

### 3.4 Access Control Lists (ACLs)

Unix permissions are limited to owner, group, others. **ACLs** extend this to arbitrary users and groups.

Linux uses `getfacl` and `setfacl`:

```bash
setfacl -m u:bob:rw file.txt   # give bob read/write
getfacl file.txt
```

ACLs are supported in many modern file systems (ext4, XFS, ZFS).

---

## 4. Capabilities – Breaking “Root is God”

Traditional Unix: process with UID 0 (root) has unlimited power. **Capabilities** split root privileges into fine‑grained units. A process can have only the capabilities it needs.

Example capabilities: `CAP_NET_BIND_SERVICE` (bind to privileged ports), `CAP_SYS_ADMIN` (system administration), `CAP_DAC_OVERRIDE` (bypass file permissions).

You can see capabilities of a process with `getpcaps`:

```bash
$ getpcaps 1234
Capabilities for `1234': = cap_net_bind_service+ep
```

To drop capabilities when running a program:

```bash
capsh --drop=CAP_SYS_ADMIN -- -c /path/to/program
```

**Example**: In C, use `cap_set_proc()` to drop privileges after starting.

This is a form of **principle of least privilege**: give only the permissions absolutely needed.

---

## 5. Mandatory Access Control in Practice

### 5.1 SELinux (Security‑Enhanced Linux)

SELinux enforces a MAC policy defined by the administrator. Every resource (file, process, socket) gets a **security context** (user:role:type:level).

- Type Enforcement (TE): main mechanism. Rules define which domains (types of processes) can access which types of objects.
- Example context: `system_u:object_r:httpd_sys_content_t:s0` (a web server file).
- If a process in the `httpd_t` domain tries to write to a file with type `etc_t` (system configuration), SELinux denies it even if the file permissions would allow it.

**Viewing SELinux contexts**:

```bash
ls -Z /var/www/html
ps -Z
```

**Troubleshooting**:

```bash
audit2allow -a   # analyze denied events and suggest policy
```

### 5.2 AppArmor

Another MAC framework, more profile‑based. It restricts programs by defining what files they can access, network capabilities, etc.

Profiles are stored in `/etc/apparmor.d/`. Example profile snippet for `/usr/sbin/mysqld`:

```
/usr/sbin/mysqld {
    /var/lib/mysql/** rw,
    /etc/mysql/* r,
    capability net_bind_service,
}
```

AppArmor is considered simpler than SELinux and is default on some distributions (Ubuntu).

---

## 6. Authentication vs. Authorization

**Authentication**: verifying identity (e.g., login with password, SSH key).  
**Authorization**: once authenticated, what resources can the user access?

In Unix, authentication is handled by `login`, `pam` (Pluggable Authentication Modules), or `sshd`.  
Authorization is enforced by the kernel via the file system permissions, capabilities, and MAC.

**Example**: When you run `cat /etc/shadow`, the kernel checks:

1. Your UID (from the process) – if root, bypass.
2. Otherwise, check file permissions (owner: root, group: shadow, others: none). If you’re not root and not in the shadow group, deny.
3. If SELinux is active, also check the security context.

---

## 7. Secure Design Principles

- **Least Privilege**: Give a process only the permissions it needs, and no more.
- **Separation of Privilege**: Require multiple independent conditions to perform sensitive actions (e.g., two‑factor authentication).
- **Fail‑Safe Defaults**: Deny access unless explicitly granted.
- **Economy of Mechanism**: Keep security code simple to avoid bugs.
- **Complete Mediation**: Every access must be checked; no caching of permissions that can be bypassed.

---

## 8. Common Security Pitfalls

- **Overly permissive file permissions** (e.g., world‑writable config files).
- **setuid root binaries with bugs** – can lead to privilege escalation.
- **Ignoring capabilities** – running a web server as root when it only needs to bind to port 80.
- **Bypassing MAC** – disabling SELinux/AppArmor entirely instead of fixing policies.
- **Not using ACLs** – when fine‑grained access is needed, resorting to making files world‑readable.

---

## 9. Code Example: Dropping Privileges Securely

A typical pattern for a daemon: start as root, bind to a privileged port, then drop to an unprivileged user.

```c
#include <unistd.h>
#include <sys/types.h>
#include <pwd.h>
#include <stdio.h>

void drop_privileges(const char *user) {
    struct passwd *pw = getpwnam(user);
    if (!pw) {
        perror("getpwnam");
        exit(1);
    }
    // set group and user IDs
    if (setgid(pw->pw_gid) != 0 || setuid(pw->pw_uid) != 0) {
        perror("setuid/setgid");
        exit(1);
    }
    // drop supplementary groups
    if (initgroups(user, pw->pw_gid) != 0) {
        perror("initgroups");
        exit(1);
    }
    // optionally drop capabilities (not shown)
}

int main() {
    // ... bind socket while root ...
    drop_privileges("nobody");
    // ... continue as nobody ...
    return 0;
}
```

---

## 10. Summary

- **Access control** is the core of OS security, governing how processes interact with resources.
- **DAC** (like Unix permissions) gives control to resource owners; **MAC** (SELinux, AppArmor) enforces system‑wide policies.
- **Capabilities** allow fine‑grained delegation of root privileges.
- **ACLs** extend beyond the classic user/group/others model.
- Security design principles (least privilege, complete mediation) guide robust implementations.

_“Access control is the OS’s immune system—it must distinguish friend from foe with relentless precision.”_

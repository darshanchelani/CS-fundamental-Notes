## 1. Why IPC?

Processes are isolated by design (remember user/kernel mode, virtual memory). But applications often need to cooperate:

- A shell pipeline: `ls | grep txt` – `ls` writes, `grep` reads.
- A web server handing requests to a separate logging process.
- A client and server communicating over a network.

IPC provides the **channels** for this communication.

**Analogy**: Processes are like houses in a neighbourhood. IPC is the mail system (pipes, sockets) that lets them send letters, packages, and even talk over the fence.

---

## 2. Pipes – The Original Unix IPC

Pipes are **unidirectional**, **byte‑stream** channels connecting two processes. They are created in memory and live as long as at least one end is open.

### 2.1 Anonymous Pipes

Used between related processes (e.g., parent and child after `fork`).  
In C, use `pipe(int fd[2])` – `fd[0]` for reading, `fd[1]` for writing.

**Example: parent writes, child reads**

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main() {
    int pipefd[2];
    pid_t pid;
    char buffer[256];

    if (pipe(pipefd) == -1) {
        perror("pipe");
        return 1;
    }

    pid = fork();
    if (pid == 0) {  // child
        close(pipefd[1]); // close write end
        read(pipefd[0], buffer, sizeof(buffer));
        printf("Child received: %s\n", buffer);
        close(pipefd[0]);
        return 0;
    } else {         // parent
        close(pipefd[0]); // close read end
        char *msg = "Hello from parent!";
        write(pipefd[1], msg, strlen(msg) + 1);
        close(pipefd[1]);
        wait(NULL);
    }
    return 0;
}
```

**Python equivalent** – using `os.pipe()` and `os.fork()`:

```python
import os

r, w = os.pipe()
pid = os.fork()

if pid == 0:  # child
    os.close(w)
    data = os.read(r, 100)
    print("Child received:", data.decode())
    os.close(r)
else:         # parent
    os.close(r)
    os.write(w, b"Hello from parent!")
    os.close(w)
    os.wait()
```

### 2.2 Named Pipes (FIFOs)

Anonymous pipes have no name and only work between related processes. **Named pipes** (FIFOs) exist as files in the filesystem, allowing unrelated processes to communicate.

**Creating a named pipe**:

```bash
mkfifo mypipe
```

**Writer process**:

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd = open("mypipe", O_WRONLY);
    write(fd, "Hello, FIFO!", 13);
    close(fd);
    return 0;
}
```

**Reader process**:

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    char buf[256];
    int fd = open("mypipe", O_RDONLY);
    int n = read(fd, buf, sizeof(buf));
    write(1, buf, n);  // output to stdout
    close(fd);
    return 0;
}
```

Run reader first (it will block until writer opens), then writer.

**Analogy**: Anonymous pipe = a **private phone line** between two rooms in the same building. Named pipe = a **mailbox** with an address; anyone can drop a letter, and anyone can pick it up.

---

## 3. Sockets – Network and Local IPC

Sockets are the **universal IPC** – they work between processes on the same machine (Unix domain sockets) and across networks (Internet sockets). They are **bidirectional** and support both **stream** (TCP) and **datagram** (UDP) semantics.

### 3.1 Unix Domain Sockets (Local)

Use the filesystem as a rendezvous point. Fast, no network overhead.

**Server** (stream):

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define SOCKET_PATH "/tmp/mysocket"

int main() {
    int server_fd, client_fd;
    struct sockaddr_un addr;
    char buffer[256];

    // create socket
    server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (server_fd < 0) { perror("socket"); return 1; }

    // bind to path
    unlink(SOCKET_PATH);  // remove if exists
    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (bind(server_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("bind"); return 1;
    }

    listen(server_fd, 5);
    client_fd = accept(server_fd, NULL, NULL);
    if (client_fd < 0) { perror("accept"); return 1; }

    read(client_fd, buffer, sizeof(buffer));
    printf("Received: %s\n", buffer);
    write(client_fd, "Hello from server", 18);

    close(client_fd);
    close(server_fd);
    unlink(SOCKET_PATH);
    return 0;
}
```

**Client**:

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define SOCKET_PATH "/tmp/mysocket"

int main() {
    int sock;
    struct sockaddr_un addr;
    char buffer[256];

    sock = socket(AF_UNIX, SOCK_STREAM, 0);
    if (sock < 0) { perror("socket"); return 1; }

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sock, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
        perror("connect"); return 1;
    }

    write(sock, "Hello from client", 18);
    read(sock, buffer, sizeof(buffer));
    printf("Server replied: %s\n", buffer);

    close(sock);
    return 0;
}
```

**Python version** (simpler):

```python
# server
import socket
import os

sock_path = "/tmp/mysocket"

if os.path.exists(sock_path):
    os.unlink(sock_path)

server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
server.bind(sock_path)
server.listen(1)

conn, _ = server.accept()
data = conn.recv(1024)
print("Received:", data.decode())
conn.send(b"Hello from server")
conn.close()
server.close()

# client
client = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
client.connect(sock_path)
client.send(b"Hello from client")
resp = client.recv(1024)
print("Server says:", resp.decode())
client.close()
```

### 3.2 Network Sockets (TCP/IP)

Same API, but with `AF_INET` and IP addresses/ports.

**TCP echo server** (C):

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {.sin_family = AF_INET,
                               .sin_port = htons(8080),
                               .sin_addr.s_addr = INADDR_ANY};

    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 5);
    int client_fd = accept(server_fd, NULL, NULL);
    char buf[1024];
    int n = read(client_fd, buf, sizeof(buf));
    write(client_fd, buf, n);  // echo back
    close(client_fd);
    close(server_fd);
    return 0;
}
```

Client connects to `localhost:8080`.

---

## 4. Other IPC Mechanisms

- **Message queues** – POSIX message queues (`mq_open`, `mq_send`, `mq_receive`). Provide message boundaries and priorities.
- **Shared memory** – The fastest IPC, because processes access the same physical memory region. Must be used with synchronisation (semaphores, mutexes).
- **Signals** – Not for data transfer, but for event notification (e.g., `SIGUSR1`).
- **D‑Bus** – High‑level IPC used in desktop environments (e.g., between applications and system services).

---

## 5. Comparison and Use Cases

| Mechanism         | Speed                | Data Boundaries | Use Case                                   |
| ----------------- | -------------------- | --------------- | ------------------------------------------ |
| Anonymous Pipe    | Medium               | Stream          | Between parent/child, simple filters       |
| Named Pipe (FIFO) | Medium               | Stream          | Unrelated processes on same machine        |
| Unix Socket       | High                 | Stream/Datagram | Local, high‑performance IPC, bidirectional |
| TCP Socket        | Lower (over network) | Stream          | Network communication, reliable, ordered   |
| UDP Socket        | Variable             | Datagram        | Low‑latency, loss‑tolerant (e.g., video)   |
| Shared Memory     | Very high            | Byte stream     | Large data exchange, needs synchronisation |
| Message Queue     | Medium               | Message         | Structured messages, priorities            |

---

## 6. Common Pitfalls

- **Deadlock** – With pipes, if both ends are blocked waiting for each other.
- **Buffer limits** – Pipes have a fixed capacity; writes may block until data is read.
- **Address already in use** – Binding a socket to a port that’s already occupied.
- **End‑of‑file handling** – In pipes, when writer closes, read returns 0 (EOF). In sockets, it’s a connection reset.
- **Data serialisation** – You often need to send structured data; agree on a format (JSON, protobuf, or simple fixed‑size structs).

---

## 7. Summary

- **Pipes** are simple, unidirectional, and great for parent‑child communication or shell pipelines.
- **Named pipes** extend this to unrelated processes using the filesystem.
- **Sockets** are the universal IPC: Unix domain for local, TCP/UDP for networking. They are bidirectional and support both streaming and datagram models.
- The OS provides many IPC mechanisms; choosing the right one depends on the need for speed, reliability, message boundaries, and whether communication is local or remote.

Understanding IPC is essential for building anything from command‑line utilities to distributed systems. When you use a database connection, a web API, or even a shell pipe, you’re relying on IPC.

_“IPC is the nervous system of a multi‑process world—it carries the signals that turn isolated units into a coherent system.”_

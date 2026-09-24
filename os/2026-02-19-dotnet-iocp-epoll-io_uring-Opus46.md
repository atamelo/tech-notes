# Windows I/O Completion Ports (IOCP) - In-Depth Explanation

---

## 1. The Problem Space

### 1.1 The C10K Problem (and Beyond)

The foundational problem is: **how does a single machine efficiently handle tens of thousands of concurrent I/O operations** - network sockets, file reads, pipe communications - without dedicating a thread to each one?

I/O is slow. A disk read takes milliseconds; a network round-trip can take hundreds of milliseconds. During that time, whatever issued the I/O is _waiting_. The question is how you structure your program around that waiting.

### 1.2 The Naïve Models and Why They Fail

**Model A: One-Thread-Per-Connection (Blocking I/O)**

```
for each incoming connection:
    spawn a new thread
    thread calls recv() in a loop, blocking until data arrives
```

This is simple, but it collapses at scale:

- **Memory pressure.** Each thread has a stack (default 1 MB on Windows). 10,000 connections = ~10 GB of committed stack space, most of it idle.
- **Context-switch overhead.** The OS scheduler must cycle through thousands of threads, most of which are blocked. The cost of saving/restoring register state, flushing TLB entries, and polluting CPU caches becomes dominant.
- **Scheduler inefficiency.** The Windows scheduler is O(1) for dispatch but the sheer volume of thread state transitions (blocked → ready → running → blocked) creates contention on internal kernel locks (the dispatcher lock, scheduler database lock).

**Model B: Single-Thread Select/Poll Loop**

```
loop:
    select(all_sockets, timeout)
    for each ready socket:
        handle it
```

This avoids the thread explosion, but:

- **`select()` is O(n).** Every call copies the entire fd_set into the kernel and back, and the kernel scans every descriptor linearly.
- **Single-threaded bottleneck.** You can only use one CPU core. Handling one request blocks all others. If a handler accidentally does something slow (a DNS lookup, a log write), everything stalls.
- **No parallelism.** Modern machines have 8, 16, 64+ cores. A single-thread model wastes all but one.

**Model C: Thread Pool with Polling**

You could have N threads all calling `select()` or `WaitForMultipleObjects()`:

- **Thundering herd.** All threads wake up when one event fires; only one can actually handle it. The rest burn CPU for nothing.
- **`WaitForMultipleObjects` limit.** Hard cap of 64 handles per call on Windows, requiring awkward partitioning.
- **No built-in load balancing.** You must manually distribute sockets across threads.

### 1.3 What We Actually Want

The ideal solution has these properties:

1. **Asynchronous I/O initiation.** "Start this read; don't block me; tell me when it's done."
2. **Unified completion notification.** A single place where _all_ completions arrive, regardless of which socket/file/pipe they belong to.
3. **Automatic, bounded thread pool.** A small, fixed set of worker threads that the OS itself keeps optimally busy - not too few (wasting CPU) and not too many (wasting memory and causing thrashing).
4. **LIFO thread wake-up.** When a completion arrives, wake the _most recently blocked_ thread, because its stack and cache lines are still hot.
5. **Concurrency throttling.** If a worker thread happens to block (e.g., on a lock), the OS should transparently wake another worker so the CPU doesn't sit idle, but only up to a configured maximum to avoid oversubscription.

IOCP is the Windows kernel primitive that delivers all five of these.

---

## 2. The Required Pieces

### 2.1 Overlapped I/O (the Async Primitive)

Before IOCP can deliver completions, you need a way to _start_ an I/O operation without blocking. On Windows, this is **Overlapped I/O**.

Every async Windows I/O function - `ReadFile`, `WriteFile`, `WSARecv`, `WSASend`, `ConnectEx`, `AcceptEx`, `TransmitFile`, etc. - accepts a pointer to an `OVERLAPPED` structure:

```c
typedef struct _OVERLAPPED {
    ULONG_PTR Internal;       // used by OS: I/O status
    ULONG_PTR InternalHigh;   // used by OS: bytes transferred
    union {
        struct {
            DWORD Offset;     // file position low
            DWORD OffsetHigh; // file position high
        };
        PVOID Pointer;
    };
    HANDLE hEvent;            // optional event (not used with IOCP)
} OVERLAPPED;
```

When you call `ReadFile(hFile, buf, len, NULL, &overlapped)`:

- The kernel queues the I/O request to the appropriate driver stack.
- The call returns **immediately** with `FALSE`, and `GetLastError()` returns `ERROR_IO_PENDING`.
- The buffer you provided (`buf`) must remain valid and untouched until the I/O completes.
- When the driver finishes, the kernel _posts a completion_ somewhere - and that "somewhere" is the IOCP.

**Key insight:** The `OVERLAPPED` structure is _your_ per-operation state token. You almost always embed it inside a larger structure:

```c
typedef struct {
    OVERLAPPED  ov;           // must be first (or use CONTAINING_RECORD)
    WSABUF      wsabuf;
    char        buffer[4096];
    enum { OP_READ, OP_WRITE, OP_ACCEPT } opType;
    SOCKET      socket;
} PER_IO_CONTEXT;
```

When the completion is dequeued, you get back the `OVERLAPPED*`, and you use `CONTAINING_RECORD` (or pointer arithmetic, since it's the first member) to recover your full context. This pattern is the backbone of every IOCP-based server.

### 2.2 The I/O Completion Port Object

The IOCP itself is a **kernel object** - an opaque handle created via `CreateIoCompletionPort`. Conceptually, it contains:

```
┌─────────────────────────────────────────────┐
│           I/O COMPLETION PORT               │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  Completion Queue (FIFO)            │    │
│  │  [completion3] [completion2] [c1]───┼──► dequeue
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  Waiting Thread Queue (LIFO)        │    │
│  │  thread_C → thread_B → thread_A     │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  Released Thread List               │    │
│  │  (threads currently processing)     │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  MaxConcurrency = N                         │
│  (typically = number of CPU cores)          │
└─────────────────────────────────────────────┘
```

- **Completion Queue (FIFO).** Completed I/O packets land here. Each packet carries: the completion key, the bytes transferred, the `OVERLAPPED*`, and an error code.
- **Waiting Thread Queue (LIFO).** Threads blocked in `GetQueuedCompletionStatus` are pushed here. LIFO order means the most-recently-blocked thread is woken first (cache-hot).
- **Released Thread List.** Threads currently running (processing a completion). The kernel tracks how many there are.
- **MaxConcurrency.** The maximum number of threads the IOCP will allow to run simultaneously. Typically set to the number of CPU cores. If a released thread blocks on something else (a mutex, a sleep, a synchronous I/O call), the kernel notices and releases another waiting thread to keep `MaxConcurrency` threads active.

### 2.3 Handle Association

An IOCP on its own does nothing. You must **associate** file/socket handles with it. This is also done through `CreateIoCompletionPort`, but used in its "associate" mode:

```c
CreateIoCompletionPort(
    hSocket,            // the handle to associate
    hIOCP,              // existing IOCP
    completionKey,      // arbitrary per-handle value (usually a pointer)
    0                   // ignored when associating
);
```

After this call, _any_ overlapped I/O operation on `hSocket` that completes will have its completion packet delivered to `hIOCP`. The `completionKey` is your per-handle context - typically a pointer to a `PER_SOCKET_CONTEXT` structure.

A single IOCP can (and should) have thousands of handles associated with it.

### 2.4 Sockets in Overlapped Mode

For network I/O, sockets must be created with the `WSA_FLAG_OVERLAPPED` flag (which is actually the default for `WSASocket` and `socket`). But certain advanced operations require specific setup:

- **`AcceptEx`**: Pre-posts an accept operation. The listening socket must be associated with the IOCP. You pre-allocate a buffer that will receive the local/remote addresses and optionally the first chunk of data.
- **`ConnectEx`**: The socket must be bound first (even to `INADDR_ANY:0`). You get a function pointer via `WSAIoctl` with `GUID WSAID_CONNECTEX`.
- **`DisconnectEx`**: Allows socket reuse without closing/reopening.

These "Ex" functions are retrieved at runtime via GUIDs because they're Winsock extension functions provided by the transport provider, not direct Win32 exports.

### 2.5 The Worker Thread Pool

IOCP doesn't create threads for you. You create a pool of threads, and each one runs a simple loop:

```c
DWORD WINAPI WorkerThread(LPVOID param) {
    HANDLE hIOCP = (HANDLE)param;
    DWORD  bytesTransferred;
    ULONG_PTR completionKey;
    OVERLAPPED *pOv;

    while (TRUE) {
        BOOL ok = GetQueuedCompletionStatus(
            hIOCP,
            &bytesTransferred,
            &completionKey,
            &pOv,
            INFINITE
        );
        // ... handle completion ...
    }
}
```

The recommended thread count is **2 × NumberOfProcessors** (Microsoft's guidance), with `MaxConcurrency` set to `NumberOfProcessors`. The extra threads exist so that if a worker temporarily blocks (e.g., acquiring a lock), the kernel can release a standby thread to keep all cores busy.

---

## 3. How the Pieces Come Together

### 3.1 The Core Flow

```
                    ┌──────────────┐
   User code        │  Worker Pool │
   issues async ──► │  Thread 1    │◄──┐
   I/O (ReadFile,   │  Thread 2    │   │  GetQueuedCompletionStatus()
   WSARecv, etc.)   │  ...         │   │  blocks until a completion
                    │  Thread N    │   │  is available
                    └──────┬───────┘   │
                           │           │
              ┌────────────▼───────────┤
              │     IOCP Kernel Object │
              │  ┌──────────────────┐  │
  Driver      │  │ Completion Queue │  │
  completes ──┼─►│ [pkt][pkt][pkt]  │──┘
  I/O         │  └──────────────────┘  │
              │  MaxConcurrency = 4    │
              └────────────────────────┘
```

Step by step:

1. **Setup.** Create the IOCP. Create worker threads. Each calls `GetQueuedCompletionStatus` and blocks (enters the waiting thread queue, LIFO).

2. **Associate handles.** As sockets are created (e.g., via `AcceptEx`), associate each with the IOCP along with a per-handle context pointer as the completion key.

3. **Post I/O.** Call `WSARecv(socket, ..., &overlapped)`. The Winsock layer passes this to the kernel, which passes it to the network driver. The call returns immediately with `WSA_IO_PENDING`.

4. **Data arrives.** The NIC interrupts the CPU. The driver's DPC (Deferred Procedure Call) copies data from the NIC buffer into your user-mode buffer (or maps it via DMA). The kernel _posts a completion packet_ to the IOCP's completion queue.

5. **Thread wake-up.** The kernel checks: is the number of _released_ threads less than `MaxConcurrency`? If yes, pop the top of the waiting thread queue (LIFO) and wake it. That thread's `GetQueuedCompletionStatus` call returns with the completion key, bytes transferred, and `OVERLAPPED*`.

6. **Processing.** The worker recovers its per-I/O context from the `OVERLAPPED*`, its per-socket context from the completion key, and handles the event (parses HTTP, sends a response, etc.).

7. **Re-arm.** The worker posts a new `WSARecv` on the same socket to continue reading, then loops back to `GetQueuedCompletionStatus`.

### 3.2 The Concurrency Governor

This is the most subtle and powerful aspect. Suppose `MaxConcurrency = 4` (quad-core machine):

- 4 worker threads are processing completions (released).
- 4 more are waiting in `GetQueuedCompletionStatus`.
- Worker thread #2 acquires a database lock and blocks.
- The kernel detects that released count dropped to 3 - below `MaxConcurrency`.
- It wakes one of the waiting threads. Now 4 threads are active again.
- Worker #2 gets the lock, resumes. Now 5 are active - temporarily above `MaxConcurrency`.
- The kernel allows this momentary overshoot but won't wake _more_ threads until the count drops below `MaxConcurrency` again.

This means: **you get automatic, adaptive parallelism with no user-mode coordination**, bounded by real CPU count.

### 3.3 LIFO Wake-Up and Cache Efficiency

The LIFO ordering of the waiting thread queue is a deliberate performance optimization. If thread A just finished handling a completion and called `GetQueuedCompletionStatus` 1 microsecond ago, its stack, local variables, and cache lines are still hot. If a new completion arrives, waking thread A (the most recent) is cheaper than waking thread D (which blocked 500ms ago and whose cache lines have long been evicted). This alone can yield measurable throughput improvements at high load.

### 3.4 Manual Completions via PostQueuedCompletionStatus

You can inject _synthetic_ completion packets into the IOCP:

```c
PostQueuedCompletionStatus(hIOCP, 0, SHUTDOWN_KEY, NULL);
```

This is how you:

- Signal worker threads to shut down gracefully.
- Wake workers for non-I/O events (timers, inter-thread messages).
- Integrate custom event sources into the same dispatch loop.

---

## 4. Concrete API Usage

### 4.1 Creating the IOCP

```c
// Create a new IOCP with concurrency = number of cores
SYSTEM_INFO si;
GetSystemInfo(&si);

HANDLE hIOCP = CreateIoCompletionPort(
    INVALID_HANDLE_VALUE,   // no handle to associate yet
    NULL,                   // create new (not associate)
    0,                      // completion key (unused here)
    si.dwNumberOfProcessors // MaxConcurrency
);
```

### 4.2 Associating a Socket

```c
typedef struct {
    SOCKET socket;
    SOCKADDR_IN addr;
    // ... per-connection state ...
} PER_SOCKET_CONTEXT;

PER_SOCKET_CONTEXT *ctx = malloc(sizeof(*ctx));
ctx->socket = acceptedSocket;

CreateIoCompletionPort(
    (HANDLE)ctx->socket,
    hIOCP,
    (ULONG_PTR)ctx,   // completion key = pointer to context
    0                 // ignored
);
```

### 4.3 Posting an Async Read

```c
PER_IO_CONTEXT *ioCtx = malloc(sizeof(*ioCtx));
memset(&ioCtx->ov, 0, sizeof(OVERLAPPED)); // MUST zero before each use
ioCtx->opType = OP_READ;
ioCtx->socket = ctx->socket;
ioCtx->wsabuf.buf = ioCtx->buffer;
ioCtx->wsabuf.len = sizeof(ioCtx->buffer);

DWORD flags = 0;
int rc = WSARecv(
    ctx->socket,
    &ioCtx->wsabuf, 1,
    NULL,            // bytes received - ignored for overlapped
    &flags,
    &ioCtx->ov,
    NULL             // no completion routine - using IOCP
);

if (rc == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
    // real error - clean up
}
// Otherwise: I/O is pending. Completion will arrive on hIOCP.
```

### 4.4 The Worker Loop

```c
DWORD WINAPI WorkerThread(LPVOID param) {
    HANDLE hIOCP = (HANDLE)param;

    for (;;) {
        DWORD bytes;
        ULONG_PTR key;
        OVERLAPPED *pOv;

        BOOL ok = GetQueuedCompletionStatus(hIOCP, &bytes, &key, &pOv, INFINITE);

        if (key == SHUTDOWN_KEY) break; // graceful exit

        PER_SOCKET_CONTEXT *sockCtx = (PER_SOCKET_CONTEXT *)key;
        PER_IO_CONTEXT *ioCtx = CONTAINING_RECORD(pOv, PER_IO_CONTEXT, ov);

        if (!ok || bytes == 0) {
            // Connection closed or error
            closesocket(sockCtx->socket);
            free(ioCtx);
            free(sockCtx);
            continue;
        }

        switch (ioCtx->opType) {
        case OP_READ:
            // Process received data in ioCtx->buffer[0..bytes-1]
            ProcessData(sockCtx, ioCtx->buffer, bytes);

            // Post a response (async write)
            ioCtx->opType = OP_WRITE;
            memset(&ioCtx->ov, 0, sizeof(OVERLAPPED));
            // fill ioCtx->wsabuf with response data ...
            WSASend(sockCtx->socket, &ioCtx->wsabuf, 1, NULL, 0, &ioCtx->ov, NULL);
            break;

        case OP_WRITE:
            // Write completed. Re-arm the read.
            ioCtx->opType = OP_READ;
            memset(&ioCtx->ov, 0, sizeof(OVERLAPPED));
            ioCtx->wsabuf.buf = ioCtx->buffer;
            ioCtx->wsabuf.len = sizeof(ioCtx->buffer);
            DWORD flags = 0;
            WSARecv(sockCtx->socket, &ioCtx->wsabuf, 1, NULL, &flags, &ioCtx->ov, NULL);
            break;
        }
    }
    return 0;
}
```

### 4.5 AcceptEx for Scalable Accepts

Instead of blocking on `accept()`, you pre-post accept operations:

```c
// Get function pointer (once, at startup)
GUID guidAcceptEx = WSAID_ACCEPTEX;
LPFN_ACCEPTEX lpfnAcceptEx;
DWORD dwBytes;
WSAIoctl(listenSocket, SIO_GET_EXTENSION_FUNCTION_POINTER,
         &guidAcceptEx, sizeof(guidAcceptEx),
         &lpfnAcceptEx, sizeof(lpfnAcceptEx),
         &dwBytes, NULL, NULL);

// Pre-post an accept
SOCKET acceptSocket = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
PER_IO_CONTEXT *ioCtx = malloc(sizeof(*ioCtx));
memset(&ioCtx->ov, 0, sizeof(OVERLAPPED));
ioCtx->opType = OP_ACCEPT;
ioCtx->socket = acceptSocket;

lpfnAcceptEx(
    listenSocket,
    acceptSocket,
    ioCtx->buffer,                     // receives addresses + optional first data
    0,                                 // 0 = don't wait for data, just accept
    sizeof(SOCKADDR_IN) + 16,
    sizeof(SOCKADDR_IN) + 16,
    &dwBytes,
    &ioCtx->ov
);
// Completion arrives on the IOCP when a client connects.
```

When the accept completes, the worker must call `setsockopt(acceptSocket, SOL_SOCKET, SO_UPDATE_ACCEPT_CONTEXT, ...)` to fully inherit the listening socket's properties, then associate the new socket with the IOCP and post a read.

### 4.6 Graceful Shutdown

```c
// Signal all workers to exit
for (int i = 0; i < numWorkers; i++) {
    PostQueuedCompletionStatus(hIOCP, 0, SHUTDOWN_KEY, NULL);
}
WaitForMultipleObjects(numWorkers, hThreads, TRUE, INFINITE);
CloseHandle(hIOCP);
```

### 4.7 GetQueuedCompletionStatusEx (Batched Dequeue)

Windows Vista+ added a batched version that dequeues multiple completions in a single syscall:

```c
OVERLAPPED_ENTRY entries[16];
ULONG removed;
GetQueuedCompletionStatusEx(hIOCP, entries, 16, &removed, INFINITE, FALSE);
for (ULONG i = 0; i < removed; i++) {
    // entries[i].lpCompletionKey, entries[i].lpOverlapped, etc.
}
```

This amortizes the cost of kernel transitions when under high load.

---

## 5. Complete Flow: Echo Server Lifecycle

Let's trace the full lifecycle of an echo server handling one client, from process start to graceful shutdown.

### Phase 1: Initialization

```
Main Thread
    │
    ├─ WSAStartup()
    ├─ Create IOCP: CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 4)
    │   → Kernel creates IOCP object with MaxConcurrency=4
    │
    ├─ Create listening socket (overlapped mode)
    ├─ bind() to port 8080
    ├─ listen()
    ├─ Associate listener with IOCP:
    │   CreateIoCompletionPort(listenSocket, hIOCP, (ULONG_PTR)listenCtx, 0)
    │
    ├─ Pre-post 10 AcceptEx operations (accept pool)
    │   Each creates a pre-allocated socket + PER_IO_CONTEXT with opType=OP_ACCEPT
    │   All 10 are now pending inside the kernel's accept queue
    │
    ├─ Create 8 worker threads (2 × 4 cores)
    │   Each thread enters WorkerThread(), calls GetQueuedCompletionStatus(), blocks.
    │   IOCP Waiting Queue (LIFO): [T8, T7, T6, T5, T4, T3, T2, T1]
    │
    └─ Main thread waits (e.g., WaitForSingleObject on a shutdown event)
```

**State of the IOCP:**

- Completion Queue: empty
- Waiting Queue: 8 threads (LIFO)
- Released: 0
- Pending I/O: 10 AcceptEx operations on the listener

### Phase 2: Client Connects

```
Event: TCP SYN arrives from client 192.168.1.50:49152
    │
    ├─ Windows TCP/IP stack completes 3-way handshake
    ├─ One of the 10 pending AcceptEx operations completes
    ├─ Kernel posts completion to IOCP:
    │   { key = listenCtx, bytes = 0, pOv = &acceptIoCtx->ov }
    │
    ├─ IOCP checks: released threads (0) < MaxConcurrency (4)? YES
    ├─ Pop LIFO: wake Thread T8
    │   Waiting Queue: [T7, T6, T5, T4, T3, T2, T1]
    │   Released: [T8]
    │
    └─ T8's GetQueuedCompletionStatus() returns
```

**Thread T8 processes the accept:**

```
T8:
    ├─ Recover PER_IO_CONTEXT from pOv → opType == OP_ACCEPT
    ├─ Extract client socket from ioCtx->socket
    ├─ setsockopt(SO_UPDATE_ACCEPT_CONTEXT) to inherit listener properties
    ├─ Allocate PER_SOCKET_CONTEXT for this client
    ├─ Associate client socket with IOCP:
    │   CreateIoCompletionPort(clientSocket, hIOCP, (ULONG_PTR)clientCtx, 0)
    │
    ├─ Post a WSARecv on the client socket:
    │   Allocate new PER_IO_CONTEXT, opType=OP_READ
    │   WSARecv(clientSocket, ..., &readIoCtx->ov)
    │   → Returns WSA_IO_PENDING. Read is now pending in the kernel.
    │
    ├─ Re-post AcceptEx to replenish the accept pool:
    │   Reset the old ioCtx, create new pre-accept socket
    │   AcceptEx(listenSocket, newSocket, ..., &ioCtx->ov)
    │
    └─ Loop back to GetQueuedCompletionStatus() → blocks
        Waiting Queue: [T8, T7, T6, T5, T4, T3, T2, T1]  (T8 pushed on top, LIFO)
        Released: []
```

### Phase 3: Client Sends Data

```
Event: Client sends "Hello, server!\n"
    │
    ├─ NIC receives packet → DMA to kernel buffer → TCP reassembly
    ├─ Kernel copies data to the user-mode buffer (readIoCtx->buffer)
    │   pointed to by the pending WSARecv's WSABUF
    ├─ Posts completion: { key = clientCtx, bytes = 15, pOv = &readIoCtx->ov }
    │
    ├─ IOCP: released (0) < MaxConcurrency (4)? YES → wake T8 (LIFO top)
    │   Released: [T8]
    │
    └─ T8's GQCS returns: ok=TRUE, bytes=15, key=clientCtx, pOv=readIoCtx
```

**Thread T8 processes the read:**

```
T8:
    ├─ Recover PER_IO_CONTEXT → opType == OP_READ
    ├─ readIoCtx->buffer now contains "Hello, server!\n" (15 bytes)
    │
    ├─ Echo: prepare to send the same data back
    │   Reuse ioCtx: opType = OP_WRITE
    │   wsabuf.buf = ioCtx->buffer (already has the data)
    │   wsabuf.len = 15
    │   memset(&ioCtx->ov, 0, sizeof(OVERLAPPED))
    │   WSASend(clientSocket, &wsabuf, 1, NULL, 0, &ioCtx->ov, NULL)
    │   → WSA_IO_PENDING
    │
    └─ Loop back to GQCS → blocks
        Waiting Queue: [T8, ...]
```

### Phase 4: Write Completes

```
Event: TCP stack finishes sending data, ACK received from client
    │
    ├─ Kernel posts completion: { key = clientCtx, bytes = 15, pOv = &writeIoCtx->ov }
    ├─ Wake T8 again (LIFO, cache-hot)
    │
    └─ T8 processes:
        ├─ opType == OP_WRITE, bytes == 15 → write succeeded
        ├─ Re-arm the read:
        │   ioCtx->opType = OP_READ
        │   memset(&ioCtx->ov, 0, sizeof(OVERLAPPED))
        │   WSARecv(clientSocket, ..., &ioCtx->ov)
        │   → WSA_IO_PENDING
        │
        └─ Back to GQCS → blocks
```

The cycle repeats: read → process → write → read → ...

### Phase 5: Client Disconnects

```
Event: Client sends FIN (graceful close)
    │
    ├─ The pending WSARecv completes with bytes = 0
    ├─ Kernel posts completion: { key = clientCtx, bytes = 0, pOv = &readIoCtx->ov }
    │
    └─ T8 wakes:
        ├─ bytes == 0 → graceful disconnect
        ├─ closesocket(clientSocket)
        ├─ free(readIoCtx)    // PER_IO_CONTEXT
        ├─ free(clientCtx)    // PER_SOCKET_CONTEXT
        └─ Back to GQCS
```

### Phase 6: Shutdown

```
Main Thread (receives shutdown signal):
    │
    ├─ for i = 0..7:
    │   PostQueuedCompletionStatus(hIOCP, 0, SHUTDOWN_KEY, NULL)
    │   → 8 synthetic completions enter the queue
    │
    ├─ Each worker wakes, sees key == SHUTDOWN_KEY, breaks loop, returns
    │
    ├─ WaitForMultipleObjects(8, hThreads, TRUE, INFINITE)
    ├─ Close all remaining sockets
    ├─ CloseHandle(hIOCP)
    └─ WSACleanup()
```

### Summary of Object Lifetimes

| Object                             | Created                | Destroyed                               | Owned By              |
| ---------------------------------- | ---------------------- | --------------------------------------- | --------------------- |
| IOCP handle                        | Process start          | Shutdown                                | Main thread           |
| Worker threads                     | Process start          | Shutdown (via SHUTDOWN_KEY)             | Main thread           |
| Listen socket                      | Process start          | Shutdown                                | Main thread           |
| PER_SOCKET_CONTEXT                 | AcceptEx completes     | Client disconnects or error             | Worker thread         |
| Pre-accept socket                  | AcceptEx posted        | Becomes client socket on accept         | Transitions ownership |
| PER_IO_CONTEXT                     | Before each async I/O  | After completion processed, or on error | Worker thread         |
| OVERLAPPED (inside PER_IO_CONTEXT) | Zeroed before each I/O | With its parent PER_IO_CONTEXT          | Worker thread         |

---

## 6. How the .NET Runtime Uses IOCP Internally

### 6.1 One IOCP Per Process - The Global I/O Completion Port

The .NET runtime (CoreCLR) creates **exactly one IOCP for the entire process**. This happens during `ThreadPool` initialization, deep in the native runtime startup path. In CoreCLR source (roughly `src/coreclr/vm/win32threadpool.cpp`), the initialization looks conceptually like:

```cpp
// Simplified from CoreCLR internals
HANDLE g_hCompletionPort = CreateIoCompletionPort(
    INVALID_HANDLE_VALUE,
    NULL,
    0,
    0   // MaxConcurrency = 0 → the OS uses the number of processors
);
```

Note that `MaxConcurrency = 0` - this is the Windows convention meaning "use the number of logical processors." The runtime trusts the OS's concurrency governor entirely.

This single IOCP serves as the convergence point for **all** asynchronous I/O across the entire .NET process: every `FileStream.ReadAsync`, every `Socket.ReceiveAsync`, every `NamedPipeClientStream.ReadAsync`, every `HttpClient` request - all of their OS-level completions funnel through this one kernel object. There is no per-socket IOCP, no per-module IOCP - just one.

### 6.2 The Two ThreadPools: Worker vs. I/O

The .NET `ThreadPool` is actually **two distinct pools** behind a single API surface:

```
┌───────────────────────────────────────────────────────────────┐
│                    .NET ThreadPool                            │
│                                                               │
│  ┌────────────────────────┐    ┌────────────────────────────┐ │
│  │   Worker Thread Pool   │    │   I/O Completion Pool      │ │
│  │                        │    │                            │ │
│  │  Driven by:            │    │  Driven by:                │ │
│  │  - Global work queue   │    │  - The single IOCP         │ │
│  │  - Local work queues   │    │  - GetQueuedCompletion-    │ │
│  │  - Work stealing       │    │    Status() loop           │ │
│  │                        │    │                            │ │
│  │  Threads do:           │    │  Threads do:               │ │
│  │  - Task.Run callbacks  │    │  - Receive OS completions  │ │
│  │  - QueueUserWorkItem   │    │  - Invoke IOCompletion-    │ │
│  │  - async continuations │    │    Callback                │ │
│  │  - Timer callbacks     │    │  - Typically hand off to   │ │
│  │                        │    │    worker pool             │ │
│  │  Sizing: Hill Climbing │    │  Sizing: demand-driven,    │ │
│  │  algorithm             │    │  simpler heuristic         │ │
│  └────────────────────────┘    └────────────────────────────┘ │
│                                                               │
│  ThreadPool.SetMinThreads(workerMin, ioMin)                   │
│  ThreadPool.SetMaxThreads(workerMax, ioMax)                   │
│  ThreadPool.GetAvailableThreads(out workers, out io)          │
└───────────────────────────────────────────────────────────────┘
```

The **worker threads** handle `Task.Run`, `QueueUserWorkItem`, `async/await` continuations (the code _after_ the `await`), and timer callbacks. Their sizing is managed by a sophisticated hill-climbing algorithm that experiments with adding/removing threads to maximize throughput.

The **I/O completion threads** are fundamentally different. Each one sits in a tight loop calling `GetQueuedCompletionStatus` on the process-wide IOCP. When an OS-level I/O completion arrives, one of these threads wakes up, invokes the registered managed callback, and then loops back to `GQCS`. Their count is managed by a simpler demand-driven heuristic: the runtime adds I/O threads when completions are arriving faster than they're being processed, and lets them retire after an idle timeout.

You can observe both pools via `ThreadPool.GetAvailableThreads(out int workerThreads, out int ioThreads)` - these are two separate numbers.

### 6.3 The I/O Thread Loop (Native Side)

Each I/O completion thread in the CLR runs a native function that looks conceptually like:

```cpp
// Simplified from CoreCLR's ThreadpoolMgr::CompletionPortThreadStart
DWORD CompletionPortThreadStart(LPVOID lpArgs) {
    for (;;) {
        DWORD bytesTransferred;
        ULONG_PTR completionKey;
        OVERLAPPED *pOverlapped;

        BOOL success = GetQueuedCompletionStatus(
            g_hCompletionPort,
            &bytesTransferred,
            &completionKey,
            &pOverlapped,
            timeout  // not INFINITE - allows the thread to retire if idle
        );

        if (pOverlapped == NULL) {
            // Timeout or IOCP closed - consider thread retirement
            if (ShouldRetireThread()) break;
            continue;
        }

        DWORD errorCode = success ? ERROR_SUCCESS : GetLastError();

        // Transition into managed code and dispatch
        DispatchManagedCallback(pOverlapped, errorCode, bytesTransferred);
    }
    return 0;
}
```

The `DispatchManagedCallback` step is where the native OVERLAPPED pointer is resolved back to a managed `NativeOverlapped*`, which in turn points to a managed `IOCompletionCallback` delegate. More on this below.

### 6.4 Binding Handles: ThreadPoolBoundHandle

The bridge between a Win32 handle and the CLR's global IOCP is `ThreadPoolBoundHandle` (in `System.Threading`):

```csharp
// This is what .NET's own FileStream, Socket, etc. call internally
SafeFileHandle fileHandle = ...; // from CreateFile, WSASocket, etc.
ThreadPoolBoundHandle boundHandle = ThreadPoolBoundHandle.BindHandle(fileHandle);
```

Under the hood, `BindHandle` does exactly one thing on Windows:

```csharp
// Pseudo-code of what happens inside BindHandle
public static ThreadPoolBoundHandle BindHandle(SafeHandle handle) {
    // Calls into native CLR:
    // CreateIoCompletionPort(handle, g_hCompletionPort, key, 0);
    ThreadPool.BindHandle(handle);
    return new ThreadPoolBoundHandle(handle);
}
```

This is the same `CreateIoCompletionPort` "associate" call from the Win32 world. After this call, any overlapped I/O on that handle will deliver completions to the CLR's global IOCP. The handle is now "bound."

**Every .NET type that does async I/O calls BindHandle during construction:**

- `FileStream` (when `useAsync: true` or `isAsync: true`) binds its file handle
- `Socket` binds its socket handle the first time you call an async operation
- `NamedPipeClientStream` / `NamedPipeServerStream` bind their pipe handles
- `HttpClient` → `SocketsHttpHandler` → `Socket` → bound

If you create a `FileStream` with `useAsync: false`, the handle is **not** bound to the IOCP, and "async" operations are faked by dispatching synchronous reads to the worker thread pool - a common performance pitfall.

### 6.5 The Managed Overlapped Machinery

Win32 IOCP deals in raw `OVERLAPPED*` pointers. Managed code deals in objects, delegates, and the garbage collector. The CLR provides a layer to bridge these:

```
┌──────────────────────────────────────────────────────┐
│  Managed World                                       │
│                                                      │
│  PreAllocatedOverlapped                              │
│    - IOCompletionCallback _callback                  │
│    - object _state                                   │
│    - Pinned in memory (GC can't move it)             │
│    │                                                 │
│    ▼                                                 │
│  NativeOverlapped*  (allocated from pinned buffer)   │
│    ┌──────────────────────────┐                      │
│    │ OVERLAPPED fields        │ ◄── what the OS sees │
│    │ (Internal, InternalHigh, │                      │
│    │  Offset, hEvent)         │                      │
│    ├──────────────────────────┤                      │
│    │ CLR bookkeeping:         │                      │
│    │  → back-pointer to       │                      │
│    │    managed Overlapped    │                      │
│    │  → GC handle to state    │                      │
│    │  → callback pointer      │                      │
│    └──────────────────────────┘                      │
└──────────────────────────────────────────────────────┘
```

The lifecycle is:

1. **Allocate.** `boundHandle.AllocateNativeOverlapped(callback, state, bufferToPinn)` returns a `NativeOverlapped*`. This pins the buffer in memory (so the GC won't move it while the OS is DMA-ing into it) and allocates the native structure from a pre-pinned heap region.

2. **Post I/O.** Pass the `NativeOverlapped*` as the `lpOverlapped` parameter to the Win32 async call (`ReadFile`, `WSARecv`, etc.). From the OS's perspective, it's just an `OVERLAPPED*`.

3. **Completion arrives.** An I/O completion thread's `GQCS` returns with this `OVERLAPPED*`. The CLR's native dispatch code follows the back-pointer to find the managed `IOCompletionCallback` delegate and invokes it, passing the error code, bytes transferred, and the `NativeOverlapped*`.

4. **Free.** The callback (or code it triggers) calls `boundHandle.FreeNativeOverlapped(pOverlapped)`, which unpins the buffer and releases the native memory.

**Critical detail:** The `NativeOverlapped` must be allocated via `ThreadPoolBoundHandle.AllocateNativeOverlapped` - not via `Overlapped.Pack` directly if you want IOCP dispatch. The bound handle ensures the CLR routes the completion correctly.

### 6.6 Inside Socket.ReceiveAsync - End-to-End

Let's trace what happens when you call `await socket.ReceiveAsync(buffer)` on Windows:

```
User code                        .NET Runtime                      OS / Kernel
─────────                        ────────────                      ──────────
await socket.ReceiveAsync(buf)
    │
    ├─► Socket checks: is handle
    │   bound to IOCP? If not,
    │   call ThreadPoolBoundHandle
    │   .BindHandle(safeHandle)
    │   ──────────────────────────► CreateIoCompletionPort(
    │                                 socket, g_hIOCP, ...)
    │
    ├─► Allocate (or reuse cached)
    │   PreAllocatedOverlapped
    │   with IOCompletionCallback
    │   = SocketAsyncEngine callback
    │   Pin the receive buffer.
    │
    ├─► Call WSARecv(socket,         ──────────────────────────►
    │     &wsabuf, 1, NULL,            Kernel queues IRP to
    │     &flags,                       TCP/IP driver stack
    │     pNativeOverlapped,           Returns WSA_IO_PENDING
    │     NULL)                      ◄──────────────────────────
    │
    ├─► WSARecv returned PENDING
    │   Return incomplete ValueTask<int>
    │   to caller. Caller's state
    │   machine suspends at await.
    │
    │                                                    ┌──────────────┐
    │   ═══ time passes, data arrives on NIC ═══         │ NIC → DMA →  │
    │                                                    │ kernel buffer│
    │                                                    │ → user buffer│
    │                                                    │ (the pinned  │
    │                                                    │  managed buf)│
    │                                                    └──────┬───────┘
    │                                                           │
    │                                Kernel posts completion    │
    │                                to g_hIOCP ◄───────────────┘
    │
    │   I/O completion thread
    │   (blocked in GQCS) wakes:
    │                                GQCS returns:
    │                                  pOverlapped, bytes, ok
    │                                    │
    │                                    ▼
    │                                CLR native dispatch:
    │                                resolve NativeOverlapped*
    │                                → managed IOCompletionCallback
    │                                    │
    │                                    ▼
    │                                Callback fires ON THE
    │                                I/O COMPLETION THREAD:
    │                                  - Sets bytes received
    │                                  - Completes the ValueTask
    │                                  - If the continuation is
    │                                    short, it MAY inline on
    │                                    this I/O thread
    │                                  - Otherwise, schedules the
    │                                    continuation to the
    │                                    WORKER thread pool
    │
    │   Worker thread picks up ◄─── ThreadPool.QueueUserWorkItem(
    │   the continuation               continuation)
    │                                OR
    │                                Inlined on I/O thread if
    │                                TaskContinuationOptions
    │                                .ExecuteSynchronously
    │
    ▼
User's code after `await`
resumes with `int bytesRead`.
```

### 6.7 The I/O Thread → Worker Thread Handoff

This is a key architectural decision. When a completion arrives, the callback runs **on the I/O completion thread**. But the code after `await` (the continuation) typically runs on a **worker thread**. Why?

The I/O thread's job is to **dequeue completions as fast as possible**. If a continuation does anything non-trivial - accessing a database, computing something, even acquiring a contested lock - it blocks the I/O thread. A blocked I/O thread means the IOCP's concurrency governor must wake another I/O thread, and if all I/O threads are stuck running user continuations, completions pile up in the IOCP queue and throughput collapses.

So the standard pattern in .NET's socket/file/pipe internals is:

1. I/O thread wakes, invokes the `IOCompletionCallback`.
2. Callback does minimal work: store the result, mark the `ValueTask`/`Task` as completed.
3. The `Task` machinery schedules the continuation via `ThreadPool.UnsafeQueueUserWorkItem` → goes to the **worker** pool.
4. I/O thread immediately loops back to `GQCS`.

The exception: if `.ConfigureAwait(false)` is used (which is default in library code) and the continuation is very cheap, the runtime _may_ inline it on the I/O thread as an optimization. This is controlled by internal heuristics and `TaskContinuationOptions.ExecuteSynchronously`.

### 6.8 How .NET Async Types Map to IOCP Operations

| .NET API                                       | Underlying Win32                      | IOCP Role                 |
| ---------------------------------------------- | ------------------------------------- | ------------------------- |
| `FileStream.ReadAsync`                         | `ReadFile` with `OVERLAPPED`          | Completion via IOCP       |
| `FileStream.WriteAsync`                        | `WriteFile` with `OVERLAPPED`         | Completion via IOCP       |
| `Socket.ReceiveAsync`                          | `WSARecv`                             | Completion via IOCP       |
| `Socket.SendAsync`                             | `WSASend`                             | Completion via IOCP       |
| `Socket.AcceptAsync`                           | `AcceptEx`                            | Completion via IOCP       |
| `Socket.ConnectAsync`                          | `ConnectEx`                           | Completion via IOCP       |
| `NamedPipeServerStream.WaitForConnectionAsync` | `ConnectNamedPipe` with `OVERLAPPED`  | Completion via IOCP       |
| `HttpClient.GetAsync`                          | (via `SocketsHttpHandler` → `Socket`) | Completion via IOCP       |
| `Task.Run(...)`                                | N/A                                   | Worker pool, **not** IOCP |
| `Task.Delay(...)`                              | Timer queue                           | Worker pool callback      |
| `SemaphoreSlim.WaitAsync`                      | N/A                                   | Worker pool continuation  |

The crucial insight: only operations involving OS handles produce IOCP completions. Pure computational async (`Task.Run`) and coordination primitives (`SemaphoreSlim`, `Channel<T>`) go through the worker pool directly.

### 6.9 Thread Count Management

```csharp
// You can observe and configure both pools:
ThreadPool.GetMinThreads(out int minWorkers, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorkers, out int maxIO);

// defaults (as of .NET 8 on a 8-core machine):
// minWorkers = 8    (= Environment.ProcessorCount)
// minIO      = 1    (only 1 I/O thread initially!)
// maxWorkers = 32767
// maxIO      = 1000
```

**Worker pool sizing:** Uses a hill-climbing algorithm. Starts at `ProcessorCount` threads, then experiments: adds a thread, measures throughput; if throughput increased, adds another; if it decreased, removes one. This converges on an optimal count for the current workload. Injection rate is throttled (one new thread per 500ms if starved).

**I/O pool sizing:** Much simpler. Starts with just 1 thread. When an I/O completion arrives and no I/O thread is available to process it promptly, the runtime creates a new one. I/O threads that sit idle for a period (around 10–20 seconds in practice) retire themselves. The pool grows and shrinks purely based on I/O completion pressure.

Under heavy I/O load (e.g., a web server handling thousands of requests), you might see 4–8 I/O threads and 16–32 worker threads on an 8-core machine. The I/O thread count stays low because each I/O thread can process thousands of completions per second - the callback work is minimal.

### 6.10 Interaction with async/await State Machine

When the C# compiler transforms an `async` method, it generates a state machine struct. Here's how it interacts with IOCP:

```csharp
// Your code:
async Task HandleClientAsync(Socket socket) {
    var buffer = new byte[4096];
    int n = await socket.ReceiveAsync(buffer);  // ← suspension point
    Process(buffer, n);                          // ← continuation
}

// Compiler generates (simplified):
struct HandleClientAsyncStateMachine : IAsyncStateMachine {
    int state;
    AsyncTaskMethodBuilder builder;
    Socket socket;
    byte[] buffer;
    ValueTask<int> awaiter;

    void MoveNext() {
        switch (state) {
        case 0:
            buffer = new byte[4096];
            awaiter = socket.ReceiveAsync(buffer);
            if (!awaiter.IsCompleted) {
                state = 1;
                // Register continuation: when awaiter completes,
                // call MoveNext() again, on the ThreadPool.
                builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                return;  // ← suspend. Thread returns to pool.
            }
            goto case 1;
        case 1:
            int n = awaiter.Result;
            Process(buffer, n);
            builder.SetResult();
            return;
        }
    }
}
```

The flow through the thread pools:

```
Worker Thread A                 I/O Thread              Worker Thread B
───────────────                 ─────────               ───────────────
MoveNext() case 0
  socket.ReceiveAsync(buf)
    → WSARecv → PENDING
  AwaitUnsafeOnCompleted
    → register continuation
  return (thread A goes
  back to the worker pool)
                                GQCS wakes
                                Callback:
                                  mark ValueTask complete
                                  schedule continuation
                                  → QueueUserWorkItem
                                back to GQCS
                                                        Picks up work item
                                                        MoveNext() case 1
                                                          Process(buffer, n)
                                                        Done.
```

Note: Thread A and Thread B may or may not be the same physical thread. The worker pool doesn't guarantee affinity - and that's by design.

### 6.11 The `FILE_FLAG_OVERLAPPED` Pitfall

A common source of bugs: if you P/Invoke to `CreateFile` or work with raw handles and forget `FILE_FLAG_OVERLAPPED`, the handle operates in synchronous mode. When you then try to bind it to the IOCP via `ThreadPoolBoundHandle.BindHandle`, the I/O will **not** be truly async. The OS will complete it synchronously before returning, and the IOCP will receive a completion that was already done - turning your "async" code into blocking code that merely has extra overhead.

The .NET `FileStream(path, options)` with `options.IsAsync = true` (or the old `useAsync: true`) ensures `FILE_FLAG_OVERLAPPED` is passed. This is why `new FileStream(path, FileMode.Open)` (sync by default) followed by `ReadAsync` is slower than a properly opened async stream - it's faking async on the worker pool rather than using true overlapped I/O through the IOCP.

### 6.12 Summary: The Full .NET IOCP Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          .NET Process                                   │
│                                                                         │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐               │
│  │  FileStream    │ │  Socket        │ │ NamedPipe      │  ... etc      │
│  │  (async=true)  │ │                │ │                │               │
│  └──────┬─────────┘ └──────┬─────────┘ └──────┬─────────┘               │
│         │                  │                  │                         │
│         └─────────┬────────┴────────┬─────────┘                         │
│                   ▼                 ▼                                   │
│         ThreadPoolBoundHandle.BindHandle()                              │
│            = CreateIoCompletionPort(handle, g_hIOCP, ...)               │
│                   │                                                     │
│                   ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │           Single Process-Wide IOCP (kernel object)           │       │
│  │                                                              │       │
│  │  ┌──────────────────────┐   ┌─────────────────────────────┐  │       │
│  │  │  Completion Queue    │   │  I/O Completion Threads     │  │       │
│  │  │  (all completions    │──►│  Thread IO-1: GQCS loop     │  │       │
│  │  │   from all handles)  │   │  Thread IO-2: GQCS loop     │  │       │
│  │  └──────────────────────┘   │  Thread IO-3: GQCS loop     │  │       │
│  │                             │  (count: 1..~1000,          │  │       │
│  │                             │   grows/shrinks on demand)  │  │       │
│  │                             └──────────────┬──────────────┘  │       │
│  └────────────────────────────────────────────┼─────────────────┘       │
│                                               │                         │
│                IOCompletionCallback fires     │                         │
│                on I/O thread. Minimal work:   │                         │
│                set result, schedule           │                         │
│                continuation.                  │                         │
│                                               ▼                         │
│  ┌──────────────────────────────────────────────────────────────┐       │
│  │                Worker Thread Pool                            │       │
│  │                                                              │       │
│  │  Thread W-1  Thread W-2  Thread W-3  ...  Thread W-N         │       │
│  │                                                              │       │
│  │  Processes: Task.Run, async continuations, QueueUserWorkItem │       │
│  │  Sizing: Hill Climbing algorithm                             │       │
│  └──────────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **One IOCP per process.** Every async I/O handle in the .NET process is bound to the same kernel object. This is efficient: one queue, one set of I/O threads, one concurrency governor.

2. **Two-pool design.** I/O threads dequeue completions; worker threads run your code. This separation ensures that user code (which may block) never starves the completion-dispatch path.

3. **Binding is explicit.** Handles must be associated via `ThreadPoolBoundHandle.BindHandle`. The framework types (`FileStream`, `Socket`, etc.) do this automatically, but only when opened in async/overlapped mode.

4. **NativeOverlapped is the bridge.** It extends the Win32 `OVERLAPPED` with managed back-pointers, enabling the CLR to route a raw kernel completion to a managed delegate.

5. **I/O threads do minimal work.** The callback on the I/O thread should be cheap - set a result, schedule a continuation. Heavy work belongs on the worker pool.

6. **async/await composes naturally.** The compiler's state machine suspends at `await`, freeing both the worker thread and eventually the I/O thread to serve other completions. True async I/O means zero threads are blocked waiting for data.

---

## 7. .NET Async I/O on Linux: epoll, io_uring, and the Readiness Model

### 7.1 The Fundamental Architectural Difference: Completion vs. Readiness

Windows IOCP is a **completion-based** model:

> "Start this read into this buffer. The OS will tell me when it's **done** and the buffer is full."

Linux's primary async I/O mechanism (epoll) is a **readiness-based** model:

> "Tell me when this file descriptor is **ready** to be read without blocking. I'll do the actual read myself."

This is not a cosmetic difference - it changes the entire internal architecture:

```
Windows IOCP (Completion)              Linux epoll (Readiness)
──────────────────────────             ─────────────────────────
1. Provide buffer                      1. Register interest in fd
2. Call WSARecv (async)                2. epoll_wait() blocks
3. OS fills buffer via DMA             3. Kernel says "fd 42 is readable"
4. OS posts completion to IOCP         4. YOU call read(fd, buf, len)
5. Your thread gets full buffer        5. read() returns immediately (non-blocking)
                                       6. You process the data

Threads blocked during I/O: 0          Threads blocked during I/O: 0
Buffers pinned during I/O: YES         Buffers pinned during I/O: NO
Kernel does the copy: YES              User-space does the read: YES
```

Both achieve the same goal - no thread is blocked waiting for data - but they distribute responsibilities differently. .NET must abstract over this difference to present a unified `async/await` API.

### 7.2 epoll in Brief

epoll is a Linux kernel facility for scalable I/O event notification. It consists of three syscalls:

```c
// Create an epoll instance (returns an fd)
int epfd = epoll_create1(0);

// Register/modify interest in a file descriptor
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;  // readable, edge-triggered
ev.data.ptr = my_context;        // arbitrary user data
epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &ev);

// Wait for events (blocks until something is ready)
struct epoll_event events[64];
int n = epoll_wait(epfd, events, 64, timeout_ms);
for (int i = 0; i < n; i++) {
    void *ctx = events[i].data.ptr;
    uint32_t flags = events[i].events;  // EPOLLIN, EPOLLOUT, EPOLLERR, ...
    // fd associated with ctx is now ready - go read/write it
}
```

Key properties relevant to .NET:

- **O(1) for wait** - `epoll_wait` returns only the _ready_ fds, not all registered ones.
- **Edge-triggered mode (EPOLLET)** - notifies only on state _transitions_ (not-ready → ready), not on continuous readiness. More efficient but requires draining the fd completely on each notification.
- **`data.ptr`** - each registration carries a user pointer, which .NET uses to store a back-reference to the managed socket context.
- **Scales to millions of fds** - same O(1) characteristics as IOCP.

### 7.3 The .NET Linux Architecture: SocketAsyncEngine

On Linux, the equivalent of the Windows IOCP plumbing lives in `SocketAsyncEngine` (in `System.Net.Sockets`). The architecture:

```
┌───────────────────────────────────────────────────────────────────┐
│                       .NET Process on Linux                       │
│                                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                      │
│  │ Socket A  │  │ Socket B  │  │ Socket C  │  ... thousands       │
│  │ (async)   │  │ (async)   │  │ (async)   │                      │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                      │
│        │              │              │                            │
│        └───────┬──────┴───────┬──────┘                            │
│                ▼              ▼                                   │
│  ┌─────────────────────────────────────────────────┐              │
│  │        SocketAsyncEngine(s)                     │              │
│  │                                                 │              │
│  │  ┌─────────────────────────────────────────┐    │              │
│  │  │  epoll instance(s)                      │    │              │
│  │  │  (1 per engine, typically 1-N engines   │    │              │
│  │  │   where N ≈ processor count, capped)    │    │              │
│  │  └──────────────┬──────────────────────────┘    │              │
│  │                 │                               │              │
│  │  ┌──────────────▼──────────────────────────┐    │              │
│  │  │  Dedicated epoll thread (per engine)    │    │              │
│  │  │  while (true) {                         │    │              │
│  │  │    n = epoll_wait(epfd, events, ...);   │    │              │
│  │  │    for each ready event:                │    │              │
│  │  │      queue callback to ThreadPool       │    │              │
│  │  │  }                                      │    │              │
│  │  └─────────────────────────────────────────┘    │              │
│  └─────────────────────────────────────────────────┘              │
│                   │                                               │
│                   ▼  schedules work items                         │
│  ┌─────────────────────────────────────────────────────────┐      │
│  │              Worker Thread Pool                         │      │
│  │  Thread W-1  Thread W-2  ...  Thread W-N                │      │
│  │                                                         │      │
│  │  1. Performs the ACTUAL non-blocking read()/write()     │      │
│  │  2. Completes the ValueTask/Task                        │      │
│  │  3. Runs the async continuation                         │      │
│  └─────────────────────────────────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────┘
```

Key differences from the Windows architecture:

| Aspect                        | Windows                                 | Linux                                                                        |
| ----------------------------- | --------------------------------------- | ---------------------------------------------------------------------------- |
| Notification model            | Completion (data already in buffer)     | Readiness (fd is ready, you must read)                                       |
| Kernel object                 | 1 IOCP per process                      | 1+ epoll instances (SocketAsyncEngine)                                       |
| Dedicated threads             | I/O completion threads (GQCS loop)      | epoll threads (epoll_wait loop)                                              |
| Where the actual read happens | Kernel (before completion)              | Worker thread (after readiness notification)                                 |
| Buffer pinning during I/O     | Required (kernel writes to your buffer) | Not required (you read into buffer when ready)                               |
| Separate I/O thread pool      | Yes (distinct from worker pool)         | No - epoll threads are not the CLR "I/O pool"; worker pool does the real I/O |

### 7.4 The Flow: `await socket.ReceiveAsync(buffer)` on Linux

```
User code                     SocketAsyncEngine            epoll thread          Worker Pool
─────────                     ─────────────────            ────────────          ───────────
await socket.ReceiveAsync(buf)
    │
    ├─► Set socket to non-blocking
    │   (if not already)
    │
    ├─► Attempt read() immediately
    │   (optimistic fast-path)
    │   ├─ If data available:
    │   │   read succeeds synchronously!
    │   │   Return completed ValueTask.
    │   │   No epoll involvement at all.
    │   │
    │   └─ If EAGAIN/EWOULDBLOCK:
    │       No data yet. Register with epoll:
    │       epoll_ctl(epfd, EPOLL_CTL_MOD,
    │         fd, EPOLLIN | EPOLLET)
    │       Store pending operation state.
    │       Return incomplete ValueTask.
    │       Caller's state machine suspends.
    │
    │                                          epoll_wait() returns:
    │                                          "fd 42 is readable"
    │                                              │
    │                                              ▼
    │                                          Resolve fd → managed
    │                                          SocketAsyncContext
    │                                              │
    │                                              ▼
    │                                          Queue callback to
    │                                          worker ThreadPool
    │                                          (NOT read here - the
    │                                          epoll thread must stay
    │                                          lean, just like the
    │                                          IOCP I/O thread)
    │                                                              │
    │                                                              ▼
    │                                                        Worker thread:
    │                                                        1. read(fd, buf, len)
    │                                                           → succeeds (non-blocking)
    │                                                        2. Complete the ValueTask
    │                                                        3. Run continuation
    │                                                           (your code after await)
```

**The optimistic fast-path** is critical for performance. Many `ReceiveAsync` calls return data synchronously because the kernel's TCP receive buffer already has data. In this case, the call never touches epoll at all - it's just a non-blocking `read()` syscall that succeeds immediately, and the `ValueTask<int>` is returned already completed. The `await` doesn't even suspend. This is why `ValueTask` (instead of `Task`) matters: it avoids a heap allocation on the synchronous-completion path.

### 7.5 Why the Actual read() Happens on the Worker Thread

This is a subtle but important point. On Windows, by the time the IOCP completion fires, the data is _already in your buffer_ - the kernel did the copy as part of completing the overlapped I/O. On Linux with epoll, the completion only tells you the fd is _ready_. Someone still has to call `read()`.

The epoll thread could do the `read()` itself, but .NET deliberately doesn't:

1. **`read()` can block** in edge cases (spurious wake-ups, fd closed between notification and read).
2. **It keeps the epoll thread doing only one thing** - calling `epoll_wait` and dispatching. This mirrors the Windows design where I/O completion threads do minimal work.
3. **It naturally load-balances** across worker threads.

So the epoll thread's role is analogous to the IOCP I/O completion thread: receive notification, do the absolute minimum, hand off to the worker pool.

### 7.6 File I/O on Linux: The Uncomfortable Truth

Network I/O maps cleanly to epoll. **File I/O does not.**

`epoll` does not work with regular files. Regular file descriptors are _always_ reported as ready by epoll - because from the kernel's perspective, a disk read will always "succeed" (it just might take a while for the data to arrive from disk). This means:

```c
// This is useless for disk files:
epoll_ctl(epfd, EPOLL_CTL_ADD, regular_file_fd, &ev);
// → epoll_wait will ALWAYS immediately return "ready"
// → but the subsequent read() may still block for milliseconds waiting for disk
```

So how does `FileStream.ReadAsync()` work on Linux? **It fakes async by dispatching synchronous I/O to the worker thread pool:**

```
await fileStream.ReadAsync(buffer)
    │
    └─► ThreadPool.QueueUserWorkItem(() => {
            // This runs on a worker thread:
            int n = pread(fd, buffer, count, offset);  // BLOCKS until disk I/O completes
            // Complete the Task with the result
        });
```

This is the same "fake async" behavior as opening a `FileStream` without `useAsync: true` on Windows. A worker thread _is_ blocked. It's not truly async in the way network I/O is.

Linux's historical solutions for async file I/O:

- **POSIX AIO (`aio_read`)** - poorly implemented on Linux, uses internal thread pool anyway.
- **Linux native AIO (`io_submit`)** - only works with `O_DIRECT` (bypassing page cache), too restrictive.
- **`io_uring`** - the modern solution (see 7.8).

### 7.7 Pipe / Unix Domain Socket I/O

Unlike regular files, pipes and Unix domain sockets **do** work with epoll. They behave like network sockets: they can be non-blocking, and epoll can report readiness. So `NamedPipeClientStream.ReadAsync()` on Linux goes through the same epoll mechanism as TCP sockets.

### 7.8 io_uring: Linux Gets a Completion Model

`io_uring`, introduced in Linux 5.1 (2019), finally brings a **completion-based** model to Linux - conceptually similar to IOCP:

```
┌────────────────────────────────────────────────────┐
│                   io_uring                         │
│                                                    │
│  Submission Queue (SQ)         Completion Queue    │
│  ┌────────────────────┐       (CQ)                 │
│  │ [SQE] [SQE] [SQE]  │       ┌────────────────┐   │
│  │  read   write  ..  │       │ [CQE] [CQE]    │   │
│  └────────────────────┘       └────────────────┘   │
│                                                    │
│  User-space and kernel share these ring buffers    │
│  via mmap - submissions and completions can        │
│  happen WITHOUT syscalls (io_uring_enter optional) │
└────────────────────────────────────────────────────┘
```

Properties:

- **Completion-based**, like IOCP: submit a read with a buffer, get a completion when the data is in the buffer.
- **Works with regular files** - true async disk I/O without `O_DIRECT`.
- **Shared memory ring buffers** - can submit and reap completions without syscalls.
- **Batching** - submit many operations in one `io_uring_enter` call.
- **Linked operations** - chain operations (read then write) in the kernel.

.NET has been progressively adding io_uring support:

- **Experimental in .NET 5–7** via community libraries (e.g., `IoUring.Transport` for Kestrel).
- **.NET 8+** - `System.Net.Sockets` can use io_uring internally when available, gated by runtime configuration. The `SocketAsyncEngine` can be backed by io_uring instead of epoll.
- When io_uring is used, the architecture converges with the Windows model: submit async reads with buffers, get completions, no need for the worker thread to call `read()`.

```
With io_uring (conceptual):

await socket.ReceiveAsync(buf)
    │
    ├─► Submit SQE: { op=READ, fd=socket, buf=buffer, len=4096 }
    │   to the io_uring submission ring
    │
    │   ═══ kernel does the read asynchronously ═══
    │
    ├─► io_uring completion thread reaps CQE from completion ring
    │   → buffer is already filled (like Windows IOCP!)
    │   → schedule continuation to worker pool
    │
    └─► Worker thread runs continuation (code after await)
```

### 7.9 Summary: Linux vs. Windows Async I/O in .NET

```
┌──────────────────┬─────────────────────────┬────────────────────────────┐
│                  │       Windows           │         Linux              │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Network async    │ Overlapped I/O + IOCP   │ Non-blocking + epoll       │
│ mechanism        │ (completion-based)      │ (readiness-based)          │
│                  │                         │ or io_uring (completion)   │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ File async       │ Overlapped I/O + IOCP   │ Thread pool fake-async     │
│ mechanism        │ (true async)            │ (blocking pread on worker) │
│                  │                         │ or io_uring (true async)   │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Event loop       │ I/O completion threads  │ epoll threads              │
│ threads          │ calling GQCS            │ calling epoll_wait         │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Who reads data   │ Kernel (before          │ Worker thread (after       │
│ from socket      │ completion fires)       │ readiness notification)    │
│                  │                         │ or kernel (with io_uring)  │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Buffer pinning   │ Required during I/O     │ Not needed (epoll)         │
│                  │ (GC can't move buffer)  │ Required (io_uring)        │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Optimistic       │ Not typical - post      │ Yes - try read() first,    │
│ synchronous path │ overlapped, wait for    │ only register with epoll   │
│                  │ completion              │ if EAGAIN                  │
├──────────────────┼─────────────────────────┼────────────────────────────┤
│ Pipe/Unix socket │ N/A (named pipes use    │ epoll (same as TCP)        │
│                  │ overlapped I/O + IOCP)  │                            │
├──────────────────┼─────────────────────────┴────────────────────────────┤
│ .NET API surface │           Identical: async/await, Task/ValueTask     │
│ to user code     │           Same code runs on both platforms           │
└──────────────────┴──────────────────────────────────────────────────────┘
```

The elegance of .NET's design is that all of this is invisible to application code. Your `await socket.ReceiveAsync(buffer)` compiles to the same IL, and the runtime swaps in the right OS machinery beneath it. The `SocketAsyncEngine` on Linux and the IOCP binding on Windows are both internal implementation details behind the same `Socket` class.

---

### Key Invariants to Maintain

1. **Each outstanding I/O operation needs its own OVERLAPPED.** Reusing one causes undefined behavior and data corruption.
2. **Always zero the OVERLAPPED before reuse.** The `Internal` and `InternalHigh` fields are written by the kernel.
3. **Keep buffers alive.** The buffer pointed to by `WSABUF` must not be freed or moved until the completion arrives.
4. **Handle the `ok == FALSE && pOv != NULL` case.** This means the I/O completed with an error (e.g., connection reset). You still have a valid `pOv` and must clean up.
5. **Handle the `ok == FALSE && pOv == NULL` case.** This means `GQCS` itself failed (e.g., timeout). There's no I/O context to process.
6. **Always replenish AcceptEx.** After each accept completes, post a new one so the pool doesn't drain.

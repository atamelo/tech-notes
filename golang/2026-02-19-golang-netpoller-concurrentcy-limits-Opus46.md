# Go Netpoller & Scheduler Internals (Claude Opus 4.6)

*A conversation with Claude Opus 4.6: how many netpollers a Go process has and how much I/O they handle, why there's no dedicated netpoller thread, the latency cost of that design, the `findrunnable` path, and how goroutines end up in the global run queue.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; When a golang goroutine makes an async network call, it gets temporarily migrated from the currently assigned P to the netpoller (and returns back to the original P's LRQ once the i/o is done), right?
<br><br>
The question: how many netpollers do exist in a single go process and how many async i/o requests can a single netpoller handle?
</td></tr>
</table>

Good question, but let me first correct a subtle point in your mental model.

### The goroutine doesn't "migrate" to the netpoller

What actually happens:

- A goroutine on some M/P attempts a network I/O operation (e.g., read on a TCP conn).
- The runtime checks if the fd is ready. If not, it parks (goparks) the goroutine — it's removed from the P's local run queue entirely and its state becomes \_Gwaiting.
- The fd interest is registered with the netpoller (via `runtime.pollDesc`).
- The M and P are now free to pick up and run other goroutines — nothing is blocked.
- When the netpoller later detects that the fd is ready, it marks the goroutine as \_Grunnable and injects it back into a P's run queue (not necessarily the original one).

So the goroutine isn't "on" the netpoller — it's parked in limbo, and the netpoller is just the mechanism that wakes it up.

### To your actual questions

**How many netpollers per process?** — Exactly one. It's a process-global singleton initialized once (`runtime.netpollinit`), backed by the platform's I/O multiplexer:

- epoll on Linux
- kqueue on FreeBSD/macOS
- IOCP on Windows

There's a single epoll fd (or kqueue fd) for the entire Go process.

**How many async I/O requests can it handle?** — There is no Go-imposed limit. The runtime itself doesn't cap the number of fds registered with the netpoller. The practical limits come from the OS:

- On Linux, epoll itself has no hard per-instance limit. The bottleneck is the system-wide fd limit (`/proc/sys/fs/file-max`) and the per-process limit (`ulimit -n`). Both are tunable — on modern systems you can comfortably push into the hundreds of thousands or millions.
- `epoll_wait` is O(ready fds), not O(total fds), so even with a huge number of registered fds, you only pay for the ones that are actually active — which is why this single-netpoller design scales well.

So in practice, a single Go process can handle hundreds of thousands of concurrent network connections through that one netpoller without issue. The real bottlenecks tend to be memory (each goroutine's stack, connection buffers) and OS-level fd limits, not the netpoller itself.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; so, you're saying that the async i/o interaction b/w a G and the netpoller outlined in this article: <a href="https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html" style="color: #9ecbff;">https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html</a>
<br>
is not correct?
</td></tr>
</table>

OK, now that I've read it — no, I wouldn't say the article is wrong. But it's a deliberate simplification, and Bill Kennedy is upfront about that. He literally says at the top: the post focuses on "semantic level" behavior and "the little mechanical details are not important."

The core narrative — goroutine does network I/O, gets taken off the M/P so the M/P stays productive, netpoller handles the async I/O, goroutine comes back when ready — is correct and a useful mental model.

That said, a few specific phrasings are imprecise if you take them literally:

### 1. "Goroutine-1 is moved to the network poller"

This is the main simplification. The goroutine doesn't live "on" the netpoller the way it lives on an M. What actually happens is the goroutine is parked (`gopark`, state becomes \_Gwaiting) and a reference to it is stored in the `pollDesc` struct associated with that fd. The netpoller doesn't execute the goroutine — it just monitors the fd and knows which goroutine to wake up. "Moved to the netpoller" is a mental shorthand, not a literal description.

### 2. "The network poller has an OS Thread and it is handling an efficient event loop"

This is the most misleading line. The netpoller does not have a dedicated OS thread running an event loop. Instead, it gets polled in several ways:

- The sysmon thread (a background monitoring thread) periodically calls `netpoll()` to check for ready fds.
- Any M that's looking for work during `schedule()` → `findrunnable()` may call `netpoll()`.
- When the runtime is idle, it can block in `netpoll(block=true)`.

So it's more of a "poll on demand" model than a dedicated event-loop thread. The epoll/kqueue fd exists, but no single thread sits in a loop servicing it.

### 3. "Goroutine-1 is moved back into the LRQ for the P"

Minor point: it doesn't necessarily go back to the same P's LRQ. When `netpoll()` finds ready goroutines, they get injected into whatever P happens to be running the code that called `netpoll()`, or into the global run queue. There's no affinity to the original P.

**Bottom line:** The article gives you the right intuition for why the design is efficient (M stays busy, no extra threads needed for network I/O). If you're reasoning about performance and scheduler behavior at an application level, it's a solid model. Just don't take "moved to the netpoller" as a literal mechanical description of the runtime internals.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; The model where there is a dedicated OS thread that the netpoller exclusively uses makes a lot of sense and looks realistic. I'd like you to double-check and make 100% sure that the alternative model which you've just described is the correct one. Otherwise - correct yourself.
</td></tr>
</table>

Fair challenge. Let me go verify against the actual Go runtime source code.

OK, I've now verified against the actual Go runtime source. My original description stands — I don't need to correct myself. Here's the evidence from the source code:

### There is no dedicated netpoller thread

The source code is definitive on this. `netpoll()` is a function, not a goroutine or thread. It's called from multiple call sites by whichever M happens to need it:

1. **sysmon** (the background monitoring thread) — calls `netpoll(0)` (non-blocking) periodically when it notices no poll has happened recently. You can see the pattern: it checks `sched.lastpoll`, and if enough time has passed, does a non-blocking poll and injects any ready goroutines via `injectglist`. But sysmon is a general-purpose watchdog — it also handles timer checks, goroutine preemption, forced GC, etc. It's not a netpoller event loop.

2. **findrunnable** (any idle M looking for work) — this is actually the primary consumer. The scheduling order from the Gerrit code review lays it out explicitly:

```text
6.4 non-blocking netpoll(0) if netpollWaiters > 0 and no other M polling.
...
6.8 blocking netpoll(delay) until next earliest timer or IO ready. if any other M already polling, no blocking netpoll.
```

So when an M has exhausted all other options (local queue, global queue, work stealing), it can block in `netpoll(delay)` — but only one M at a time can do this, coordinated by `sched.pollingNet`.

3. **startTheWorldWithSema** — does a non-blocking `netpoll(0)` when resuming after a stop-the-world pause (like GC).

### The key coordination mechanism

The `sched.pollingNet` atomic variable acts as a mutex ensuring at most one M is blocked in netpoll at any time. This is the line you can see in `proc.go`:

```go
if netpollinited() && netpollAnyWaiters() && sched.lastpoll.Load() != 0 && sched.pollingNet.Swap(1) == 0 {
```

This is fundamentally different from a dedicated event-loop thread. It's an opportunistic model: whoever is idle takes the job.

### Why the article's wording is misleading

The line from the Ardan Labs article — "The network poller has an OS Thread and it is handling an efficient event loop" — suggests a dedicated thread sitting in a loop servicing epoll events, like you'd see in libuv (Node.js) or Netty. That's not what happens. The epoll fd exists (one per process, as `epfd` in `netpoll_epoll.go`), but no single thread owns it. Various Ms call `epoll_wait` on it as part of their scheduling duties.

If anything, the design is more clever than a dedicated event-loop thread, because it means no thread is wasted just waiting for I/O when there's other work to do.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; But that makes the upper boundary of i/o latency unbounded. Where a dedicated thread just for epolling wouldn't have that shortcoming.
</td></tr>
</table>

You're raising a valid point, and the Go team is aware of this tradeoff. Let me be precise about where the latency risk actually lives.

### When it works well (the common case)

If there's at least one idle M — which is typical in I/O-heavy workloads — that M blocks in `netpoll(delay)` inside `findrunnable` (step 6.8). In this scenario it's effectively acting as a dedicated poller thread. The wake-up latency is just the OS's `epoll_wait` notification latency, which is in the low microseconds. So for the typical network server, the behavior is practically equivalent to having a dedicated thread.

### When it falls apart

The bad case is when all Ps are saturated with CPU-bound work and no M is idle. Then nobody is calling netpoll, and the only backstop is sysmon. And sysmon has a progressive backoff — it starts at 20μs but backs off up to 10ms between iterations. So yes, in the worst case, an I/O-ready fd sits unnoticed for up to 10ms. That's your unbounded (well, 10ms-bounded) latency.

A dedicated poller thread wouldn't have this problem — it would be blocked in `epoll_wait` at all times, waking within microseconds of any fd becoming ready, regardless of how busy the rest of the process is.

### The Go team knows this

The gVisor issue (#29734) I found earlier is exactly about this. The gVisor team replaced the runtime netpoller with dedicated per-fd polling threads and measured TCP round-trip latency dropping from ~182μs to ~112μs. Their workload involved rapid goroutine blocking/unblocking where findrunnable overhead (including the scheduling dance around who gets to call netpoll) added measurable latency. In the discussion, Dmitry Vyukov (the scheduler's author) acknowledged the gap but argued it should be fixable within the existing design rather than requiring a dedicated thread.

### Why they still chose this design

The tradeoff is deliberate. A dedicated poller thread means:

- One OS thread permanently consumed even when there's zero network I/O
- When it wakes goroutines, those goroutines still need to be scheduled onto an M/P — so you add a cross-thread handoff that the opportunistic model avoids (in findrunnable, the M that calls netpoll can immediately run the woken goroutine itself, zero handoff)
- More contention on run queues from the dedicated thread injecting goroutines

For the vast majority of Go programs (web servers, microservices, API backends), there are always idle Ms, so the opportunistic model gives near-optimal latency and avoids the dedicated thread overhead. The 10ms worst case only materializes under heavy CPU saturation with no idle Ms — and at that point, the program is already bottlenecked on compute, so a few ms of I/O notification delay is arguably not the primary concern.

That said, you're right that for latency-sensitive systems where microsecond-level I/O responsiveness must be guaranteed even under CPU pressure, the Go scheduler's design is a real limitation. That's exactly why systems like gVisor opted out of it.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; When you say this "If there's at least one idle M — which is typical in I/O-heavy workloads — that M blocks in <code>netpoll(delay)</code> inside <code>findrunnable</code> (step 6.8)." - what exactly do you mean, can you expand on that point?
</td></tr>
</table>

Sure. Let me walk through the full findrunnable path for an M that has run out of work, step by step.

### Setup: an M runs out of goroutines

Say M3 is attached to P2 and has just finished executing a goroutine. It calls `schedule()`, which calls `findrunnable()` to get the next goroutine to run. Here's what happens inside findrunnable:

**Steps 1–5:** Check local run queue → check global run queue → non-blocking `netpoll(0)` → try to steal from other Ps' local queues. All come back empty.

**Step 6 (transition):** M3 transitions from "spinning" to "non-spinning" state. This is significant — it decrements `sched.nmspinning`.

**Step 7 (recheck):** One more pass over all queues to avoid a race where work arrived between earlier checks. Still nothing.

**Step 8 (the critical moment):** M3 has exhausted every option. Now the key question: is any other M already blocked in netpoll?

It checks `sched.pollingNet`. If no one else is polling:

- M3 releases P2 back to the idle P list — this is important, the P is now available for other Ms to pick up if new work appears
- M3, now P-less, calls `netpoll(delay)` where delay is based on the nearest pending timer
- This translates directly to `epoll_wait(epfd, events, maxEvents, timeout)` at the syscall level
- The OS thread blocks in the kernel — it consumes no CPU, it's just sitting in a wait queue inside the kernel's epoll implementation

At this point, M3 is effectively a dedicated poller thread — an OS thread doing nothing but waiting for I/O readiness notifications. The difference from a truly dedicated thread is that this role is transient and opportunistic: M3 didn't get assigned this job, it just happened to be the one with nothing better to do.

### When I/O becomes ready

An fd becomes ready. The kernel wakes M3 from `epoll_wait`. M3 now has a `gList` — a list of goroutines whose I/O is ready. Then:

- M3 needs a P to run goroutines — it tries to acquire one from the idle P list
- If it gets a P, it can directly start executing one of the ready goroutines — no cross-thread handoff, no run queue insertion. This is the latency advantage I mentioned: the thread that noticed the I/O readiness immediately runs the goroutine
- The remaining ready goroutines (if any) get put on the global run queue via `injectglist`, where other Ms can pick them up

### Why "at least one idle M" matters so much

If all Ms are busy running CPU-bound goroutines on their Ps, then nobody ever enters findrunnable, nobody reaches step 6.8, and nobody is sitting in `epoll_wait`. The only thing checking for I/O readiness is sysmon's periodic non-blocking `netpoll(0)`, with its 20μs → 10ms backoff.

But in a typical I/O-heavy server (say, handling HTTP requests), goroutines are constantly parking on network reads and waking up. This means Ms frequently cycle through findrunnable, and at any given moment it's very likely that at least one M has found no work and has settled into the blocking `netpoll(delay)` call. So you get microsecond-level I/O notification almost for free, without dedicating a thread to it.

The design essentially says: "the thread that would otherwise be sleeping anyway might as well be our epoll waiter." It's elegant, but as you correctly identified, it has no guarantee — it's "usually someone is there" rather than "someone is always there."

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; What are the ways that a goroutine can end up in the GRQ?
</td></tr>
</table>

Good question — there are several distinct paths. Let me trace each one through the runtime.

### 1. Local run queue overflow

Each P's LRQ is a fixed-size ring buffer of 256 slots. When a new goroutine is created (`go func()`) or a goroutine becomes runnable and `runqput` finds the LRQ full, it calls `runqputslow`, which moves half of the LRQ (128 goroutines) plus the new one to the GRQ in a single batch. This amortizes the cost of the global lock on `sched.lock`.

This is probably the most common path in programs that spawn goroutines rapidly.

### 2. runtime.Gosched() — voluntary yield

When a goroutine explicitly calls `runtime.Gosched()`, the path is `gosched_m` → `goschedImpl` → `globrunqput`. It goes directly to the GRQ, not back to the local queue. This makes sense — the goroutine is saying "I don't need to run urgently, let someone else go," so putting it on the GRQ gives it the lowest priority.

### 3. Async preemption (Go 1.14+)

When sysmon detects a goroutine has been running on a P for over 10ms, it triggers signal-based preemption. The preempted goroutine ends up calling `goschedImpl` → `globrunqput` — same GRQ path as `Gosched()`. Before Go 1.14, this only worked at function call preambles (cooperative preemption via stack growth checks), but the destination was the same.

### 4. Syscall return when the P is gone

When a goroutine returns from a blocking syscall (the synchronous kind where the M/P were detached, as in the Ardan Labs article's figures 6–8), it calls `exitsyscall`. It first tries to reacquire its old P (`oldp`). If that P was taken by another M, it tries to grab any idle P. If no P is available at all, it calls `exitsyscall0` → `globrunqput` — the goroutine is runnable but has nowhere to run, so it goes to the GRQ to wait until a P becomes free.

### 5. Netpoll result injection

When netpoll returns a batch of ready goroutines (from any of its call sites — findrunnable, sysmon, startTheWorldWithSema), they're handed to `injectglist`. This function tries to distribute goroutines to idle Ps' local queues first, but any remainder goes to the GRQ via `globrunqputbatch`. In the common case where sysmon finds 50 ready goroutines but only 2 Ps are idle, the extra 48 end up on the GRQ.

### 6. GC-related transitions

After a stop-the-world pause (e.g., GC mark phase), `startTheWorldWithSema` does a non-blocking `netpoll(0)` and feeds results through `injectglist` (same as #5). Also, GC background workers and finalizer goroutines can end up on the GRQ when they're kicked off by the runtime.

### How goroutines leave the GRQ

For completeness — the drain side: in `schedule()`, before checking the LRQ, there's this check:

```go
// Check the global runnable queue once in a while to ensure fairness.
// Otherwise two goroutines can completely occupy the local runqueue
// by constantly respawning each other.
if schedtick%61 == 0 {
    // try to get from GRQ
}
```

So every 61st scheduling tick, a P pulls from the GRQ instead of the LRQ. And of course during work stealing, if an idle P finds nothing in other Ps' LRQs, it checks the GRQ too (findrunnable step 6.3).

The 1/61 ratio is a tuned heuristic — high enough to prevent GRQ starvation, low enough to preserve locality (goroutines on the LRQ tend to have warm caches for that P's M/core).

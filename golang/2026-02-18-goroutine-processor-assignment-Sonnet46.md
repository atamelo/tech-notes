# Goroutines, Ps & Ms: Scheduling and Preemption (Claude Sonnet 4.6)

*A conversation with Claude Sonnet 4.6: how Gs, Ps and Ms really relate, how SIGURG-based async preemption works (safe points, `gsignal`, `sigaltstack`), how Go's approach differs from .NET's thread injection, and why GOMAXPROCS is fixed.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Do I get it right that:
<br><br>
1. In the golang runtime a goroutine (G) is "hard-assigned" to a particular processor (P) (P can have many such goroutines assigned to it) and that processor (P) is, in its turn, hard-assigned to one particular OS thread (M).
<br>
2. All goroutines assigned to a P are scheduled to the same thread M taking turns.
<br>
3. A goroutine can't easily migrate from its P to some other P.
<br>
4. P can't be easily assigned a different M.
<br>
?
</td></tr>
</table>

Your intuition is partially right but the key details are off. Here's how it actually works:

**What you got right:** A P can only run on one M at a time, and all goroutines on a given P do run on the same OS thread at any given moment.

**What's different from your model:**

**On point 1 & 3 — Goroutines migrate freely.** There is no "hard assignment" of a G to a P. The Go scheduler actively steals goroutines between Ps. Each P has a local run queue, but when a P runs out of work it will steal half the goroutines from another P's run queue. Additionally, there's a global run queue that all Ps can pull from. Migration is a core feature, not an edge case.

**On point 4 — P and M are also not hard-assigned to each other.** This is the most important thing to understand. The P/M binding is dynamic and is the whole point of having Ps as a separate concept. When an M blocks on a syscall, it releases its P. Another idle M (or a newly created one) picks up that P and keeps running goroutines. When the blocking M returns from the syscall, it tries to acquire a P again — if none is free, it parks its goroutine on the global queue and puts itself to sleep. So the number of active Ps stays at GOMAXPROCS even if many threads are blocked in syscalls.

**The mental model that helps:** Think of P as a "permit to run Go code." There are exactly GOMAXPROCS permits. An M must hold a permit to execute goroutines, and must surrender it whenever it can't make progress (e.g. syscall, blocking cgo). Goroutines live in queues attached to permits, not to threads.

So the real picture is:

- **G** → loosely associated with a P's run queue, but routinely migrates
- **P** → holds a run queue and local state; binds to whatever M is currently running it
- **M** → grabs whichever P is available; may block and release its P at any time

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Your "permit" model is too childish. Imo P is better viewed as a queue/pool of G that can run on a given M (that this P is currently assigned to). And the code (a runtime thunk) constantly running on M tries to pull items (Gs) from that queue to execute.
</td></tr>
</table>

That's a fair and more accurate description of the mechanics. The M does essentially run a scheduler loop (`schedule()`) that continuously pulls Gs from the P's local run queue — so modeling P as a queue/pool that M drains is much closer to what's actually happening in the runtime code.

The one thing worth adding to your model is that P carries more state than just the run queue. It also holds the `mcache` (per-P memory allocation cache), pooled defer structs, GC-related work buffers, and a few other caches. This is actually a big part of why P exists as a separate concept from M — keeping that state per-P rather than per-M means that when an M blocks and hands its P off to another M, the new M inherits all that warm cache state and can keep allocating and running GC work efficiently without contention. If all that state lived on M, you'd either lose it on a block or need locks to share it.

So your model is right — P is essentially a run queue plus local caches, and M is the engine draining it — but the caches are load-bearing, not incidental.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; So, for a 4 core CPU, does that mean that if my program creates a 100k goroutines (G) all of those will be executed by only 4 Ms ?
</td></tr>
</table>

Not quite — it depends on what those goroutines are doing.

**If they're purely CPU-bound:** Yes, essentially 4 Ms do all the work. With `GOMAXPROCS=4` you get 4 Ps, and since no M ever blocks, there's no reason to create more than 4 Ms. Your 100k Gs sit in the P run queues and get drained by those 4 Ms.

**If they do blocking syscalls (file I/O, network, etc.):** The number of Ms can grow well beyond 4. When an M blocks in a syscall, it releases its P, and the runtime spins up a new (or parks/unparks an existing) M to take over that P. So you can end up with many more Ms than Ps — potentially hundreds or thousands if many goroutines are blocked simultaneously. The Go runtime caps Ms at 10,000 by default (`runtime/debug.SetMaxThreads`).

**Network I/O is a special case though:** Go uses a netpoller (epoll/kqueue) so network-blocked goroutines don't actually block their M at all. They're parked and a dedicated poller goroutine wakes them up when their fd is ready. So purely network-bound goroutines won't cause M proliferation.

**The short summary:** the number of concurrently running goroutines is always capped at GOMAXPROCS, but the number of Ms can grow if goroutines block on syscalls. The 100k Gs are mostly just data sitting in queues, waiting their turn.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Tell me what is the role of SIGURG in the model we discussed here.
</td></tr>
</table>

It's the mechanism for asynchronous preemption, introduced in Go 1.14.

**The problem it solves:** In the model we discussed, M runs a scheduler loop that pulls Gs from P's queue. But once M starts executing a G, when does it stop and switch to the next one? Originally Go used cooperative preemption — a G would only yield at function call points, where the compiler inserts checks. A tight CPU-bound loop with no function calls could monopolize an M/P indefinitely, starving other Gs in that P's queue.

**What SIGURG does:** A special runtime goroutine called `sysmon` runs outside the normal P/M model and monitors all running Gs. If it notices a G has been occupying its P for more than ~10ms, it sends SIGURG to the M running that G. The signal handler interrupts execution, saves the G's register state, puts the G back in the run queue, and returns control to M's scheduler loop — which then picks up the next G from P's queue. Effectively a forced context switch.

**Why SIGURG specifically:** It was chosen because it's a signal that almost no real-world code uses for its own purposes, so hijacking it for the runtime's internal signaling is safe in practice.

So in terms of your model — SIGURG is the mechanism that ensures M doesn't get stuck draining just one item from P's queue forever. It's the preemption hook that makes the scheduling loop actually fair under CPU-bound workloads.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; 1. How does the SIGURG handler make sure that it's safe to interrupt a given G currently running on M? 2. Where does the code handling SIGURG signal lives and executes (i.e. what thread)? 3. When executing, what stack does the SIGURG handler use?
</td></tr>
</table>

Good set of questions, they're all connected.

### 1. How does the handler know it's safe to interrupt?

It often doesn't preempt immediately — it checks first. The handler inspects the interrupted G's program counter and verifies it's at a "safe point" — a point where the compiler-generated stack/pointer maps are valid and the GC can correctly identify all roots on the stack. If the G is inside a runtime-internal critical section, holding a lock, in a write barrier, or otherwise in non-preemptible code, the handler simply returns and lets sysmon try again later. The compiler actually generates safe-point metadata for essentially every instruction in Go 1.14+ (not just function call sites), which is what made async preemption possible in the first place.

### 2. Which thread executes the signal handler?

sysmon uses `tgkill` (on Linux) to send SIGURG to a specific OS thread — the M that is running the target G. So the handler executes on that same M, not on sysmon's thread. Sysmon just pulls the trigger; the interrupted M handles it itself.

### 3. Which stack does the handler use?

Each M has an alternate signal stack set up via `sigaltstack`, backed by a special internal goroutine called `gsignal` that lives on that M. The signal handler runs on this gsignal stack. This is important because at the moment of interruption the G's own stack might be in an arbitrary state, and you can't safely push signal handler frames onto it. The gsignal stack is a small, dedicated, always-valid landing pad for exactly this purpose.

**The flow then is:** SIGURG hits M → handler runs on gsignal stack → inspects interrupted G's PC for safety → if safe, saves G's full register state, marks G as runnable, puts it back in P's queue → returns to M's scheduler loop to pick up next G.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; You missed an important point in the picture above: who sets the stack pointer to point to the gsignal stack and when exactly does that happen?
</td></tr>
</table>

The key point you're highlighting: it's the kernel that switches the stack pointer, not Go runtime code.

When the Go runtime creates a new M, it calls `sigaltstack` to register the gsignal buffer as the alternate signal stack for that OS thread. That's just registration — nothing happens yet.

Later, when SIGURG is delivered to that thread, the kernel itself — as part of its signal delivery machinery — sees that an alternate signal stack is registered for this thread, switches SP to point to it, and then jumps to the signal handler. All of this happens atomically from userspace's perspective, before a single instruction of the handler has executed.

This is crucial because there's no opportunity for any Go runtime code to do the switch "just in time" — the signal can arrive between any two instructions. The only safe design is to have the stack pre-registered with the kernel so the kernel handles the switch itself. That's exactly what `sigaltstack` is for.

So the full picture is:

- **M init time:** runtime calls `sigaltstack`, handing the kernel a pre-allocated gsignal stack for this thread
- **Signal delivery time:** kernel switches SP to that stack and invokes the handler — no userspace involvement in the switch itself
- **Handler runs:** now safely on gsignal stack, can inspect and manipulate the interrupted G's state without touching its stack at all

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Now, afaui, one of the key distinctions of the golang goroutines runtime from how .net does task to thread multiplexing is that in .net threadpool can inject new threads whenever it thinks starvation can occur (i.e. when some task is hogging a thread) - so essentially .net "offloads" starvation management to the OS shoulders. While in golang runtime it's not that easy to just inject a new thread M and have the P that's running a hogging goroutine G migrate to it (btw, why?).. And so the golang runtime does this "hog watching" and preempts a goroutine by "detaching" it from the M if needed. So, in contrast to .net golang doesn't rely on the OS for preemption. Would that be a fair assessment of the matter?
</td></tr>
</table>

The assessment is broadly fair but a couple of things are worth sharpening.

**On why you can't just inject a new M and have P migrate to it:** The problem is that a CPU-hogging G never causes its M to block, and P handoff only happens naturally when M blocks (syscall, etc.). The currently running M holds the P and has no reason to give it up — there's no cooperative yield point. You can create a new M, but there's no P for it to grab since the hogging M is sitting on it. The only way to free that P is to forcibly interrupt the G that's holding it hostage — which is exactly what SIGURG does. So it's not that P migration is hard per se, it's that you first need to stop the G before any migration becomes possible.

**A small correction to the framing:** SIGURG doesn't really "detach G from M" in the sense of migrating P to a new M. After preemption, G goes back into P's run queue and that same M continues running its scheduler loop — now picking up the next G from that same P. The P stays on the same M. It's purely about fairness among Gs sharing one P, not about M proliferation.

**On the .NET comparison:** It's fair but worth noting that .NET's thread injection isn't really the OS doing it — the CLR threadpool's hill-climbing algorithm decides when to inject. What .NET does rely on the OS for is actual preemption of threads via normal time-slicing. Since tasks yield at await points cooperatively, and the OS handles thread preemption at the CPU level, the CLR doesn't need its own async preemption signal mechanism. Go chose not to rely on OS time-slicing for goroutine fairness — a goroutine hogging an M would still get its full OS time-slice, starving other goroutines on that P — so it had to build SIGURG-based preemption itself.

So yes: Go owns the preemption mechanism fully, .NET leans on OS scheduling more heavily. That's a real and meaningful architectural difference.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; when you said above "The currently running M holds the P and has no reason to give it up — there's no cooperative yield point. You can create a new M, but there's no P for it to grab since the hogging M is sitting on it."
<br>
But that newly created M could have a P assigned to it that could simply "steal" awaiting Gs from the (other) P whose M has been hogged by a hog (G).
</td></tr>
</table>

You're right, and that's actually already happening — work stealing by other Ps takes care of exactly that. If M1 is hogged by a G, the other 3 Ps (on a 4-core machine) will steal Gs from P1's run queue and execute them. So the queued Gs on that P aren't completely starved.

The real, narrower problem that SIGURG solves is different: it's about that one hogging G itself never yielding. The Gs in the queue can be stolen and are fine. But the hogging G is monopolizing one full P/M indefinitely — meaning you're effectively running at GOMAXPROCS-1 capacity for the duration of the hog. SIGURG forces that G to yield so its P becomes useful again for the full rotation of Gs.

So to sharpen the picture:

- Work stealing handles the queued Gs just fine without any preemption
- SIGURG is specifically about reclaiming the P from the hogging G itself — forcing it back into the queue so it takes turns like everyone else

Which means my earlier framing of "Gs on that P are starved" was imprecise. Thanks for pushing on that.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; But when you say "But the hogging G is monopolizing one full P/M indefinitely — meaning you're effectively running at <code>GOMAXPROCS-1</code> capacity for the duration of the hog. SIGURG forces that G to yield so its P becomes useful again for the full rotation of Gs."
<br><br>
I'd like to point out that if the go runtime just added another (new) M, the whole process would be running at full GOMAXPROCS again. The hog could then keep on doing its work on the monopolized M and after the work is done, that M could be just terminated..
<br><br>
It looks like .NET essentially went for this option in their design of the Thread Pool - they don't try to preempt hogs, they just inject new threads to the threadpool
</td></tr>
</table>

You're making a valid point, but it hits exactly the architectural constraint that makes it impossible in Go's model: the number of Ps is fixed at GOMAXPROCS.

A new M without a P cannot run goroutines — M needs a P to do any useful work. And all Ps are already allocated. So unlike .NET where "inject a new thread" immediately gives you more capacity because the OS schedules threads directly, in Go you'd need to inject both a new M and a new P. But increasing the number of Ps beyond GOMAXPROCS defeats the entire purpose of having GOMAXPROCS in the first place — it's there to limit parallelism to the number of actual CPU cores to avoid over-scheduling.

.NET's design is essentially "let the OS figure out the right level of parallelism via thread injection and its own scheduler." Go's design says "we know the right level of parallelism — it's GOMAXPROCS — and we'll enforce fairness within that constraint ourselves via preemption."

So the two philosophies are:

- **.NET:** fixed fairness contract is not Go's problem, inject threads, let OS sort it out
- **Go:** we own the scheduling contract end-to-end, GOMAXPROCS is sacred, enforce fairness via SIGURG

The tradeoff is that Go gets more predictable and controlled parallelism, but has to carry the complexity of async preemption machinery to uphold its own guarantees.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; But do you agree that if the GOMAXPROCS wasn't fixed, M/P pair injection would help the hogging problem without having to implement the preemption at the golang runtime level?
</td></tr>
</table>

Yes, absolutely. If you were willing to let GOMAXPROCS float dynamically, you could just inject a new M/P pair when a hog is detected, let the hog run to completion on its captured M/P, then tear that pair down — exactly what .NET effectively does with threads. No SIGURG, no safe-point metadata, no gsignal stack, no async preemption machinery at all.

The entire complexity of Go's preemption story is essentially the price of keeping GOMAXPROCS fixed. The fixedness is a deliberate choice — predictable parallelism, no over-subscription, cache-friendly scheduling — but it means you can't escape the problem by throwing more M/P pairs at it. So you have to solve fairness the hard way: by actually interrupting the hog.

It's a classic tradeoff: Go chose a stricter resource model and then had to build significant machinery to enforce fairness within it. .NET chose a more elastic resource model and got to sidestep that machinery at the cost of giving up tight control over parallelism.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Can you give me a more in-depth look into the design choice of Go to have a fixed GOMAXPROCS? I.e. your explanation + maybe some online design docs on that topics.. Specifically, why the golang designers thought that the "fixed" model was better for golang?
</td></tr>
</table>

Good, I have enough. Here's a synthesized picture, mixing what the primary design documents say with the underlying reasoning.

The primary source to read is Vyukov's original design doc (the one that introduced P into the runtime in Go 1.1):
https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit

It's worth reading in full. The problems it was solving were with the old scheduler (pre-1.1), which had no P concept and a single global lock on the run queue. The P concept and fixed GOMAXPROCS were the solution.

**Why fixed GOMAXPROCS? The core reasons:**

1. **Over-subscription and cache thrashing.** If you run more OS threads than physical cores, the OS scheduler starts context-switching between them. Each switch potentially evicts the CPU cache state (L1/L2) of the previous thread. Go's whole point with P is that each P maintains a local run queue, and goroutines preferentially enqueue locally to avoid lock contention and improve cache locality. *Goperf*

   That locality argument collapses if you let M count float above core count — the per-P caches (mcache, GC work buffers) stop being "warm" on a real core. Fixed GOMAXPROCS = fixed number of Ps = each P tends to live on one physical core = warm caches.

2. **The mcache cost.** In the old scheduler, every M held an mcache (the per-thread memory allocation pool) of up to 2MB. If the number of idle Ms became too large it could cause significant performance loss due to excessive resource consumption and poor data locality. *Columbia University*

   By moving mcache to P, you have exactly GOMAXPROCS of them — bounded, predictable, always in use.

3. **Mutex contention on the run queue.** The old scheduler had excessive reliance on the global Sched lock — in order to modify the queues of Ms and Gs, or any other global Sched field, this single lock had to be held, creating problems for high throughput servers and parallel computational programs. *Columbia University*

   Fixing GOMAXPROCS and having per-P local queues eliminates most of that contention — each P owns its queue lock-free relative to other Ps.

4. **Predictable parallelism as a feature, not a bug.** Go's design philosophy is that the programmer sets GOMAXPROCS to match real hardware parallelism. This gives you a knob with clear semantics: "at most N goroutines run truly in parallel." The fact that the number of contexts is fixed means that only GOMAXPROCS goroutines are running Go code at any point, and you can tune the process to the individual computer, such that a 4-core PC runs Go code on 4 threads. *Morsmachine*

   Letting it float dynamically (like .NET's hill-climbing) trades that predictability for elasticity.

**The tradeoff Go accepted:**

By choosing the fixed model, Go took on the obligation of solving fairness within that fixed pool itself — which is exactly the chain of complexity we discussed: cooperative preemption wasn't enough → sysmon needed to watch for hogs → SIGURG-based async preemption was needed → which required compiler-generated safe-point metadata everywhere → which required the gsignal stack machinery.

.NET's floating thread pool sidesteps all of that at the cost of giving up tight control over core utilization and cache behavior. Go considered that tradeoff not worth it, especially for a language targeting servers where predictable latency and memory behavior under load matters more than elasticity.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; So, essentially in the golang model setting GOMAXPROCS sets an upper limit on the number of P, which in its turn means, at most GOMAXPROCS OS threads (Ms) will exist in a given OS process, right?
</td></tr>
</table>

Not quite — and we actually touched on this earlier in our conversation. GOMAXPROCS caps the number of Ps, not Ms.

The number of Ms can and does exceed GOMAXPROCS. Every time a goroutine makes a blocking syscall, its M releases its P and blocks. The runtime then spins up (or unparks) a new M to grab that P and keep things running. So if you have 1000 goroutines all blocking on syscalls simultaneously, you can have 1000+ Ms alive in the process, even though GOMAXPROCS might be 4.

What GOMAXPROCS truly caps is:

- The number of Ps — always exactly GOMAXPROCS
- The number of Ms actively running Go code at any moment — at most GOMAXPROCS, since each requires a P to do so

Ms blocked in syscalls don't need a P and don't count against that limit. They're just parked OS threads waiting for the kernel to return, outside the Go scheduler's purview entirely.

So the correct statement is: GOMAXPROCS caps parallelism of Go code execution, not the total number of OS threads.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Yup, that's what I meant. thanks for this correction
</td></tr>
</table>

No problem! It's an easy conflation to make since in the common case (pure CPU-bound work, no blocking syscalls) the two numbers happen to coincide — you get exactly GOMAXPROCS Ms doing all the work. The distinction only becomes visible once blocking syscalls enter the picture.

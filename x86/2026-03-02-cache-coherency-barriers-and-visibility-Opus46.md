Q: A question on x86 CPU memory model and cache conherency mechanisms and their interconnection (if any).
Assume a multi-threaded program executes on a x86 family processor with 2+ cores. The program constits of 2 threads running in parallel - on two diffrerent CPU cores. The first thread is writing to a shared variable (say, A) in an endless loop while the second thread is reading the value of that same shared variable A in an endless loop.
Now, as each core has its own private L1 cache, is it possible that one of the threads does a wirte to a shared variable and the other thread will simply be unaware of that write and will keep reading the 'old' value from its local cache? Or some cache-coherency protocol will kick in upon each write by the first thread to the shared variable A and so the second thread's local CPU cache (L1) will be invalidated and the subsequent read will be done from the main memory?
Note: no any synchronization primitives are used by the threads, no memory barriers ether.

A: x86 Cache Coherency and the Store Buffer Subtlety
The Short Answer

On x86, the cache coherency protocol will ensure the second thread eventually sees the updated value. The reader will not be stuck reading a stale cached copy forever. However, the story has an important nuance involving store buffers.

Cache Coherency: MESI Protocol

x86 processors implement a hardware cache coherency protocol (a variant of MESI — Modified, Exclusive, Shared, Invalid). This operates transparently and is not optional. Here's what happens at the cache level when Thread 1 writes to variable A:

The writing core must first acquire exclusive ownership of the cache line containing A. If the reading core's L1 holds a copy, the protocol forces that copy into the Invalid state before the write can commit to cache. When the reading core next tries to read A, it gets a cache miss (since its line is now invalid) and must fetch the updated line — either from the writing core's cache or from a lower level of the hierarchy (L2/L3/memory), depending on the snoop result.

So at the cache level, coherency is airtight. There is no scenario where two caches hold contradictory committed values for the same line. This is a hardware guarantee.

The Real Subtlety: Store Buffers

The complication isn't the caches — it's the store buffer. When a core executes a store, the value goes into a core-private store buffer before it's committed to the L1 cache (and thus before it becomes visible to the coherency protocol). The store sits there until the core obtains exclusive ownership of the cache line, at which point it drains into L1 and becomes globally visible.

What does this mean in practice?

For other cores reading the variable: there is a small window (typically on the order of nanoseconds, maybe tens of nanoseconds) where the write is in the store buffer and not yet visible. But the store buffer must drain — it is not an indefinite hiding place. Once it drains, the MESI protocol makes the new value visible everywhere.

For the writing core itself: it reads its own stores immediately from the store buffer (store-buffer forwarding), so it always sees its own most recent write.

What x86's TSO Model Guarantees

x86 implements Total Store Order (TSO), which is among the strongest memory models in hardware. The key guarantees relevant here:

Stores from a single core become visible to all other cores in program order. A load on any core will never see an "older" value if a "newer" value has already been observed — no going backwards. The only reordering TSO permits is that a load may be satisfied before an older store (from the same core) becomes globally visible. This is the classic store-buffer forwarding effect.

So for Your Specific Scenario

With Thread 1 writing in a loop and Thread 2 reading in a loop, and no barriers:

Thread 2 will see the updates. It cannot be stuck permanently on a stale value, because the store buffer drains continuously and the MESI protocol invalidates stale cache lines. There may be a tiny lag (nanoseconds) between when Thread 1 executes the store instruction and when Thread 2 can observe it.

Then Why Do We Need volatile, Atomics, or Barriers?

The hardware guarantees visibility, but the compiler doesn't know about your other thread. Without volatile, std::atomic, or equivalent:

The compiler may hoist the read of A out of the loop entirely, loading it into a register once and never re-reading from memory. The compiler may reorder or eliminate stores it considers redundant. Optimizations like constant propagation or dead store elimination can make the variable effectively invisible across threads.

So the danger in your scenario without synchronization primitives is not a hardware problem — it's a compiler optimization problem. Using std::atomic<int> with memory_order_relaxed is the minimal correct solution: it prevents compiler reordering/caching in registers while still letting the hardware do its thing efficiently (no extra fence instructions are emitted on x86 for relaxed loads/stores of naturally-aligned values).

Summary
Layer Risk of missing an update? Why?
L1/L2 caches No MESI protocol guarantees coherency
Store buffer Briefly (nanoseconds) Drains quickly; not indefinite
Compiler Yes — this is the real danger May cache value in a register, reorder, or eliminate accesses

The bottom line: the hardware is more helpful than people often assume. The real enemy of visibility across threads on x86 is the optimizing compiler, not the caches.

Q: But the memory barriers are not just to prevent compilers doing "wrong" things.. doesn't the use of barriers also hint the CPU on how/when to make the write visible to other cores?

A: You're raising a valid and important point — I oversimplified. Let me correct that.

Barriers Do Have Real Hardware Effects on x86

Memory barrier instructions like MFENCE, SFENCE, LFENCE, and instructions with a LOCK prefix are real hardware operations, not just compiler hints. Specifically, MFENCE forces the core to stall until its store buffer has fully drained — meaning all prior stores have become globally visible through the cache coherency protocol before the core proceeds to execute any subsequent loads or stores.

So yes, a barrier does affect when other cores can see a write, in a meaningful sense.

But the Nuance Matters: Ordering vs. Visibility

Here's the key distinction I should have made more carefully:

The store buffer always drains on its own — it's a small, finite queue and the hardware continuously retires entries as it obtains cache line ownership. A barrier doesn't make it drain faster. What a barrier does is stall the issuing core until the drain is complete. The purpose is to enforce ordering, not to "push" data out.

Why does this matter? Consider the classic double-variable pattern:

// Thread 1 // Thread 2
data = 42; if (flag == 1)
flag = 1; read data; // expect 42

Without a barrier, x86's TSO actually handles this particular case correctly — stores are committed in program order, so if Thread 2 sees flag == 1, the store to data was already globally visible. No fence needed between the two stores (store-store order is guaranteed by TSO).

But now consider this variation:

// Thread 1 // Thread 2
A = 1; B = 1;
read B; read A;

Can both threads read 0? On x86, yes — without a fence. Each core's store is sitting in its local store buffer when the subsequent load executes. The load sneaks ahead of the store becoming globally visible. This is the one reordering x86 permits: a younger load can be satisfied before an older store drains. An MFENCE between the store and load on each core prevents this by forcing the store to drain before the load executes.

This is exactly why seq_cst stores on x86 compile to MOV + MFENCE (or XCHG with its implicit lock), while release stores compile to a plain MOV — TSO already provides release semantics for free, but sequential consistency requires preventing store-load reordering, which requires a real hardware fence.

Revised Picture
What barriers do Compiler level Hardware level
Prevent reordering of accesses ✅ Yes ✅ Yes (specifically store-load reordering on x86)
Prevent caching in registers ✅ Yes N/A
Force store buffer drain N/A ✅ Yes (MFENCE, LOCK-prefixed instructions)
"Speed up" visibility No No — they stall the core until the drain completes, they don't accelerate it
Coming Back to Your Original Scenario

For your specific case (one writer, one reader, single variable), the hardware effect of a barrier is less critical — TSO and MESI already ensure the reader will see updates, and store-load reordering across a single variable isn't the problem. The dominant risk is still the compiler optimizing away the re-reads. But you're absolutely right that in the general multi-variable case, barriers serve a real and essential hardware purpose beyond just disciplining the compiler. I shouldn't have reduced them to a compiler-only concern.

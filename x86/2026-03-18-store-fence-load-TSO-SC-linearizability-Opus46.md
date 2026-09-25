# Store → Fence → Load on x86: TSO, SC & Linearizability (Claude Opus 4.6)

*A conversation with Claude Opus 4.6: how a fence between a store and a load turns x86-TSO into sequential consistency, what `MFENCE` actually does to the store buffer, whether a fence after every store gives linearizability, and why compound operations still need atomic RMWs, locks or lock-free designs.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; x86-TSO memory model is known to be a bit weaker than the 'sequential consistency' model. Specifically, because it allows <code>store->load</code> reorders on a single core execution flow.
<br>
Question: do I get it right that by adding a fence like this: <code>store->fence->load</code>, that reordering is no longer allowed and so, by doing so one can get sequentially consistent memory model?
</td></tr>
</table>

Yes, you've got it exactly right. x86-TSO is "almost" sequentially consistent - the _only_ relaxation it permits over SC is that a store can be reordered with a younger load (to a different address) on the same core. This is the store-buffer effect: the store sits in the store buffer while the load executes and retires ahead of it.

All other ordering guarantees are already maintained under x86-TSO: loads aren't reordered with loads, stores aren't reordered with stores, and loads aren't reordered with older stores.

So by inserting a full fence (e.g., `MFENCE`) between a store and a subsequent load, you drain the store buffer before the load can execute, which eliminates that single relaxation. Since that was the _only_ deviation from SC, closing it gives you a sequentially consistent execution.

In practice this is why `MFENCE` (or a locked instruction like `LOCK XCHG`, which has full-fence semantics) shows up in the implementations of SC atomics on x86. For example, a common strategy for implementing C++ `memory_order_seq_cst` on x86 is to emit an `MFENCE` after every `seq_cst` store (or before every `seq_cst` load - you only need one fence per store-load pair, and compilers pick a convention). The loads and stores themselves already give you acquire/release semantics for free on x86, so the fence is purely to close that one `store→load` reordering gap.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Do I get it right, that the exact mechanics of how <code>mfence</code> helps to establish SC on x86 (being placed like this: <code>store->mfence->load</code>) is that <code>mfence</code> makes sure that the result of the store becomes globally visible (i.e. to all other cores) before the given execution flow can continue?
</td></tr>
</table>

Yes, that's the right mental model. The mechanism is specifically about the **store buffer**.

When a core executes a store, the value doesn't go directly into the cache (and thus doesn't become globally visible immediately). Instead it enters the **store buffer** - a per-core queue that holds retired-but-not-yet-committed stores. While a value sits in the store buffer, only the local core can see it (via store-to-load forwarding). Other cores can't observe it yet because it hasn't entered the coherent cache hierarchy.

Without a fence, a younger load on the same core can execute and retire while older stores are still sitting in that buffer. That's the `store→load` reordering.

What `MFENCE` does is **drain the store buffer** before allowing any subsequent loads to execute. Once the buffer is drained, those stores have been written into the L1 cache, which means the cache coherence protocol (MESIF on Intel) makes them globally visible to all other cores. Only then does execution continue past the fence.

So to restate the sequence precisely:

1. The store enters the store buffer.
2. `MFENCE` stalls until all pending entries in the store buffer are committed to the coherent cache.
3. Now the store is globally visible - every other core will observe it (or a later value, but never a stale one preceding it).
4. Only then does the subsequent load execute.

This is exactly what closes the SC gap: you've guaranteed a total order between that store and the following load that all cores will agree on, which was the one thing TSO didn't give you for free.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Now, taking it up a notch: linearizability is stronger than sequential consistency in that it requires real-time ordering/visibility of executed operations (according to Jepsen).
<br><br>
Would introducing an <code>mfence</code> after every store (i.e. not just in <code>store->load</code> pairs) on x86 give me linearizability?
</td></tr>
</table>

This is a really good question, and the answer is more nuanced than it might seem at first.

**The short answer:** On a single cache-coherent machine, SC for individual memory operations already _effectively_ gives you linearizability. So the extra fences (after every store rather than just `store→load` pairs) don't buy you much beyond what SC already provides at the hardware level.

**Why SC ≈ linearizability on cache-coherent hardware:**

The key property linearizability adds over SC is real-time ordering: if operation A completes before operation B begins (in wall-clock time), then A must appear before B in the global order. On a cache-coherent multiprocessor, once a store enters the L1 cache (and thus the coherence protocol), it's _instantaneously_ visible to all cores - there's no propagation delay across cores. So the linearization point of each operation naturally falls within its real-time interval, which is exactly what linearizability demands.

You could argue that `MFENCE` after _every_ store gives a slightly cleaner mapping: "store instruction completes" now implies "value is globally visible," so the real-time interval of the store operation, as seen by the program, always contains the linearization point. Without the fence, a store can retire (appear "complete" to the program) while the value is still private in the store buffer. But in practice, the only way another core could _know_ your store completed is by observing memory - and at that point, ordering is already established.

**Where the real gap lies:**

The SC-vs-linearizability distinction becomes meaningful at a level _above_ individual loads and stores, which is the context Jepsen typically operates in. Linearizability is a property of _operations on objects_ - things like "enqueue," "CAS," "read a register," "commit a transaction." These are typically compound operations involving multiple memory accesses. The challenges are:

- **Compound operations** need atomicity, not just ordering. A concurrent queue's `enqueue` might involve updating a pointer and writing a value - those need to appear atomic to other threads. Fences alone don't give you that; you need locks or atomic RMW instructions (`LOCK CMPXCHG`, etc.).

- **Distributed systems** (Jepsen's main domain) don't have cache coherence at all. There's no shared bus that makes a write instantly visible to all nodes. Messages have real propagation delays, clocks drift, nodes crash. Linearizability there requires consensus protocols (Raft, Paxos), not hardware fences.

So the punchline: your intuition about fences closing the SC gap is correct, and on cache-coherent hardware that gap to linearizability is essentially zero for individual memory operations. But linearizability as a practical concern lives at a higher abstraction level where the hard problems are atomicity of compound operations and, in distributed settings, the absence of shared physical time.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Couple of points:
<br><br>
1. You wrote above: "You could argue that <code>MFENCE</code> after every store gives a slightly cleaner mapping: "store instruction completes" now implies "value is globally visible," so the real-time interval of the store operation, as seen by the program, always contains the linearization point. Without the fence, a store can retire (appear "complete" to the program) while the value is still private in the store buffer. But in practice, the only way another core could know your store completed is by observing memory - and at that point, ordering is already established.". And I disagree with this part specifically: "But in practice, the only way another core could know your store completed is by observing memory - and at that point, ordering is already established". As the "real-time" part of the operation visibility/completion guarantee that linearizability give implies exactly that all cores must be able to observe the result of a store at the same time. Which would not be the case if <code>mfence</code> didn't follow literally every store - as while sitting in the store buffer the result of the store would only be visible to the core that executed that store, but not to the other cores. In other words, the operation (store) has completed, but other cores are unable (yet) to observe its results. I agree that in practice it might not matter sometimes, but it appears crucial for the formal compliance with the linearizability requirements.
<br><br>
2. You wrote above: "Linearizability is a property of operations on objects - things like "enqueue," "CAS," "read a register," "commit a transaction." These are typically compound operations involving multiple memory accesses.". Your point is clear here, but I'm still wondering from where does it follow that the SC consistency is any different in this regard - i.e. that it's not a property of operations on objects? After all, in the Jepsen's hierarchy both linearizability and SC belong to the same hierarchy branch (<a href="https://jepsen.io/consistency/models" style="color: #9ecbff;">https://jepsen.io/consistency/models</a>)
</td></tr>
</table>

You're right on both counts, and I think your reasoning is sharper than mine was. Let me address each.

**On point 1**, you're correct, and I was being too hand-wavy. The formal argument is clean: linearizability requires that the linearization point of every operation falls within its real-time interval [invocation, response]. For a store, if we take "response" to mean the instruction has retired, but the value is still sitting in the store buffer and not yet in the coherent cache, then the actual moment of global effect (the only valid linearization point) falls _after_ the response. That's a direct violation of the definition - it doesn't matter whether some other core "could know" or not.

With `MFENCE` after every store, you force the store buffer to drain before the instruction sequence continues, which guarantees the value is in the coherent cache (globally visible) before the store "completes" from the program's perspective. Now the linearization point falls squarely within the operation's real-time interval. So yes, formally, `MFENCE` after every store is necessary for linearizability of individual memory operations, not just a "slightly cleaner mapping." I understated it.

**On point 2**, you're also right, and my framing was misleading. Both SC and linearizability are consistency models over histories of operations on objects - they sit on the same branch in Jepsen's hierarchy for exactly this reason. The _only_ formal distinction between them is the real-time constraint. SC requires a total order consistent with each process's program order. Linearizability requires the same, _plus_ that total order must respect real-time ordering of non-overlapping operations.

What I was clumsily trying to gesture at is a practical observation: in real systems, the SC-vs-linearizability gap (the real-time requirement) tends to become _more consequential_ as operations become coarser-grained and involve more latency - especially in distributed systems where "completed but not yet globally visible" can mean milliseconds or more, not nanoseconds in a store buffer. But that's an observation about where the distinction _bites hardest in practice_, not a definitional difference between the two models. Formally, they operate at exactly the same level of abstraction.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; So, to paraphrase your point:
<br>
In terms of simple (single-value) stores and loads, adding an <code>mfence</code> after each and every store on x86 will give linearizability for single value stores and loads across all cores.
<br>
In practice, though, just single-value stores and loads are often not enough. I.e. a program typically contains more coarse-grained/compound multi-value operations (== operations on objects). You may call it "higher-level" operations from the level of abstraction standpoint compared to simple single-value stores and loads. And to provide linearizability for those multi-value operations (on objects) just having <code>mfence</code> after every (simple, single-value) store is not enough as the correct invariant for such an app object typically spans multiple fields which must be updated atomically.
<br>
This is where "higher-level" instructions like '<code>lock cmpxchg</code>' and more advanced sync primitives like mutexes, semaphores, etc come into play.
<br><br>
Is this a correct and comprehensive summary?
</td></tr>
</table>

Yes, that's a correct and comprehensive summary. I'd only add one small clarification on the granularity of the building blocks you mentioned.

`LOCK CMPXCHG` (and other atomic RMW instructions) still operate on a **single memory location** atomically. They don't, by themselves, give you multi-field atomicity. What they give you is the ability to _build_ the synchronization primitives that do. For example, a mutex is typically implemented using something like `LOCK CMPXCHG` on the lock word - and then the mutex is what provides the atomicity over the larger compound operation spanning multiple fields.

So the layering is really:

- **`MFENCE`** closes the SC→linearizability gap for individual loads and stores.
- **Atomic RMW instructions** (`LOCK CMPXCHG`, `LOCK XADD`, etc.) give you linearizable single-location read-modify-write - which is more than a plain store but still one location.
- **Synchronization primitives** (mutexes, semaphores, etc.), built _from_ those atomic RMWs, give you the ability to make arbitrary multi-location compound operations appear atomic and linearizable to outside observers.

Each layer builds on the one below it. Your summary captured the essential structure correctly - this is just making the middle step explicit.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Summing up the whole discussion from the beginning, do you see any gaps/omissions that might be worth mentioning for the topic at hand?
</td></tr>
</table>

Looking back at the full thread, I think we covered the hardware side quite thoroughly. There are a few things worth flagging:

**Compiler reordering - the gap we never mentioned.** Everything we discussed was about _hardware_ reordering. But in a real program, the compiler sits between your source code and the machine instructions, and it's free to reorder memory accesses aggressively. An `MFENCE` in your assembly means nothing if the compiler moved the store past the load before it even emitted the instructions. This is why in practice you work with language-level constructs like C++ `std::atomic` with `memory_order_seq_cst` - these emit both the right hardware fences _and_ act as compiler barriers. If you're writing raw assembly, this doesn't apply, but the moment you're in a higher-level language it's arguably the most important practical concern.

**`LOCK`-prefixed instructions already imply a full fence on x86.** We mentioned `LOCK CMPXCHG` as a building block for synchronization primitives, but didn't note that every `LOCK`-prefixed instruction on x86 has implicit full-fence semantics. This means that in practice, explicit `MFENCE` instructions are rarer than you'd expect - the locked instructions used to build mutexes and atomic RMWs already provide the necessary ordering. This is also why the common `seq_cst` implementation strategy works: you only need an `MFENCE` after plain `seq_cst` stores, not after locked operations.

**Lock-free data structures as a middle ground.** We presented a clean two-level picture - atomic RMWs for single locations, mutexes for compound operations. But there's an important middle ground: lock-free (and wait-free) data structures that achieve linearizability of compound operations _without_ mutexes, through careful choreography of atomic operations. Think of a lock-free queue or a concurrent skip list. These are harder to get right, but they demonstrate that mutexes aren't the _only_ path from single-location atomics to linearizable objects.

**Cost.** We never mentioned that `MFENCE` is expensive - typically tens of nanoseconds, because you're stalling the pipeline waiting for the store buffer to drain. This is precisely why nobody actually puts one after every store. The whole art of concurrent programming on x86 is exploiting TSO's already-strong guarantees to minimize the number of fences and locked instructions you actually need.

I think those are the main gaps. The compiler reordering one is probably the most consequential for anyone going from this theoretical understanding to writing real code.

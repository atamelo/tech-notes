# Read-Acquire / Write-Release, x86-TSO, Fences and Lock-Free Techniques (ChatGPT)

*A conversation with ChatGPT: what acquire/release ordering solves and how it's implemented, the message-passing and store-buffering litmus tests on x86 vs ARM, how `MFENCE` and locked instructions restore sequential consistency, `lock xchg` vs `mfence`, and how mutexes and lock-free structures are built from atomics.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Explain "read acquire" and "write release" semantics of memory load and store operations in the context of cpu memory models. Specifically:
<br>
1. What problems they are meant to solve
<br>
2. How exactly do they achieve/implement those semantics
<br>
3. Code examples
</td></tr>
</table>

“**Read-acquire**” and “**write-release**” are *ordering* guarantees you can attach to atomic loads/stores so you can safely “publish” data between threads without paying for a full fence / total ordering.

They’re easiest to understand as: **release = “push my earlier writes out before I signal”**, **acquire = “once I observe the signal, pull in everything that happened-before it.”**

---

## 1) What problems they’re meant to solve

### The core problem: *reordering + visibility*

Modern CPUs and compilers reorder memory operations (within the rules of the architecture / language) for performance:

* A **writer** can make a “flag” visible to other cores *before* the data the flag is supposed to protect becomes visible.
* A **reader** can speculatively / out-of-order read data *before* it has reliably observed the flag.

That gives you classic “saw the flag but not the data” bugs.

### What acquire/release gives you

Acquire/release is designed to solve **one-way handoff** problems:

* **Safe publication / message passing**
* **Hand-over-hand state machines** (publish a pointer to an initialized object)
* **Lock implementations** (unlock = release, lock = acquire)
* **Producer/consumer queues** (head/tail publication)

What it does **not** solve by itself:

* **Mutual exclusion** (it doesn’t prevent two writers from racing unless you use an RMW like CAS/test-and-set).
* **Data races on non-atomics** (you still need a proper synchronization edge; acquire/release is often that edge, but only if used correctly).

---

## 2) How they achieve those semantics

### The semantic rule (language-level / model-level)

Acquire/release is about forbidding certain reorderings:

* **Acquire load**: prevents later reads/writes in this thread from being moved **before** the acquire-load.

  * Think: “a fence **after** the load” (conceptually).
* **Release store**: prevents earlier reads/writes in this thread from being moved **after** the release-store.

  * Think: “a fence **before** the store” (conceptually).

And crucially, you only get the *cross-thread* guarantee when they **pair**:

> If Thread B does a **load-acquire** that reads the value written by Thread A’s **store-release** (same atomic), then A’s prior writes **happen-before** B’s subsequent reads.

That’s the “publication edge”.

### How it’s implemented (CPU + compiler)

Two parts:

1. **Compiler ordering**

   * The compiler is not allowed to reorder ordinary loads/stores across an acquire/release atomic in ways that would violate the above rule.

2. **CPU/hardware ordering**

   * On weakly-ordered CPUs, the compiler emits special instructions or fences:

     * **ARMv8**: `LDAR` (load-acquire), `STLR` (store-release), or `DMB` fences around plain loads/stores.
     * **RISC-V**: load/store with `aq`/`rl` bits or fences.
   * On strongly-ordered CPUs (notably **x86 TSO**), plain loads/stores are already close enough that:

     * acquire-load often compiles to a plain `mov` load
     * release-store often compiles to a plain `mov` store
     * (but stronger orders like `seq_cst`, or specific patterns like store→load ordering, may require locked ops / fences)

> Cache coherency (MESI-like) ensures *a single location* becomes visible consistently; acquire/release ensures **the right *order across multiple locations***.

---

## 3) Code examples

### A. Safe publication (the canonical example)

#### C++ (C++11 atomics)

```cpp
#include <atomic>
#include <cassert>

struct Payload {
    int x;
    int y;
};

Payload payload;
std::atomic<int> ready{0};

void producer() {
    payload.x = 1;
    payload.y = 2;
    ready.store(1, std::memory_order_release);  // publish
}

void consumer() {
    while (ready.load(std::memory_order_acquire) == 0) {
        // spin
    }
    // After acquire observes 1, payload.{x,y} must be visible
    assert(payload.x == 1);
    assert(payload.y == 2);
}
```

What this prevents:

* Producer’s writes to `payload` being observed **after** `ready = 1`.
* Consumer reading `payload` **before** it has actually “seen” the publication.

---

### B. Spinlock (acquire on lock, release on unlock)

#### C++ `atomic_flag`

```cpp
#include <atomic>

std::atomic_flag lockFlag = ATOMIC_FLAG_INIT;

void lock() {
    while (lockFlag.test_and_set(std::memory_order_acquire)) {
        // spin
    }
}

void unlock() {
    lockFlag.clear(std::memory_order_release);
}
```

Meaning:

* `unlock(release)` ensures all critical-section writes are visible before the lock is released.
* `lock(acquire)` ensures once you acquire the lock, you see everything the previous holder did.

---

### C. Rust equivalents

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

static READY: AtomicUsize = AtomicUsize::new(0);
static mut DATA: usize = 0;

fn producer() {
    unsafe { DATA = 123; }
    READY.store(1, Ordering::Release);
}

fn consumer() -> usize {
    while READY.load(Ordering::Acquire) == 0 {}
    unsafe { DATA }
}
```

Same publication edge: `store(Release)` + `load(Acquire)`.

---

### D. .NET mapping (since it often comes up)

* `Volatile.Write(ref x, v)` ≈ **release store**
* `Volatile.Read(ref x)` ≈ **acquire load**
* `Interlocked.*` operations are typically **stronger** (often full-fence / RMW semantics depending on operation)

Example:

```csharp
using System.Threading;

static int data;
static int ready;

static void Producer()
{
    data = 42;
    Volatile.Write(ref ready, 1); // release
}

static int Consumer()
{
    while (Volatile.Read(ref ready) == 0) { } // acquire
    return data;
}
```

---

## A couple of sharp edges to keep in mind

* **Acquire/release only orders *when the acquire actually observes the released value*.**

  * If the reader sees an older value, you don’t get the happens-before edge yet.
* **It does not make non-atomic races “okay.”**

  * The pattern works because `ready` is atomic and creates the ordering edge that “carries” visibility for surrounding non-atomic data.

If you want, I can also walk through a specific litmus test (message-passing / store-buffering) and show exactly which outcomes are allowed under relaxed vs acquire/release vs seq_cst - and how that maps to x86 vs ARM.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Please do
</td></tr>
</table>

Cool - let’s do the two classic litmus tests and be very explicit about:

* what outcomes are possible,
* what acquire/release changes (and what it **doesn’t**),
* what x86 (TSO) vs ARMv8 (weak) typically allow.

I’ll use C++ atomics (`std::atomic<int>`) because the memory orders map directly to the terminology.

---

## Litmus 1: Message Passing (MP) - the thing acquire/release is *made* for

**Initial state:** `x = 0, y = 0`

**Threads:**

```cpp
// Thread 0 (publisher)
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_relaxed);

// Thread 1 (observer)
int r1 = y.load(std::memory_order_relaxed);
int r2 = x.load(std::memory_order_relaxed);
```

### The “bad” outcome

`r1 == 1 && r2 == 0`

Interpretation: the reader saw the “flag” (`y==1`) but not the data (`x==0`).

### Is `r1=1, r2=0` allowed?

* **ARMv8 (and most weak models):** **allowed** with relaxed (stores to different locations can become visible out of order to another core).
* **x86 (TSO):** **not allowed** at the hardware level, because x86 preserves **Store→Store** order as observed by other cores.

> Key point: MP is *portable* only if you use acquire/release. x86 “happens to be strong enough” here, ARM isn’t.

### Fix with release/acquire (canonical publication)

```cpp
// Thread 0
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_release);   // publish

// Thread 1
int r1 = y.load(std::memory_order_acquire);
int r2 = x.load(std::memory_order_relaxed);
```

Now:

* If `r1` reads `1`, that acquire-load **synchronizes-with** the release-store on `y`.
* That creates a **happens-before** edge:

  * Thread0’s earlier writes (including `x=1`) must be visible to Thread1’s later reads.
* Therefore **`r1=1, r2=0` becomes forbidden** (on *all* architectures).

### Outcome summary (MP)

| Orders used                            |  Can `r1=1, r2=0` happen? | Why                                                                       |
| -------------------------------------- | ------------------------: | ------------------------------------------------------------------------- |
| relaxed / relaxed                      | **ARM: yes**, **x86: no** | ARM allows store visibility reordering; x86 doesn’t (Store→Store ordered) |
| `y.store(release)` + `y.load(acquire)` |         **No (portable)** | acquire that reads-from release forces visibility of prior writes         |
| seq_cst everywhere                     |         **No (portable)** | stronger than needed; also forbids it                                     |

### What the CPU usually emits

* **ARMv8:** `store-release` ≈ `STLR`, `load-acquire` ≈ `LDAR`
* **x86:** both typically compile to plain `mov` (because TSO already satisfies the needed ordering)

---

## Litmus 2: Store Buffering (SB) - the thing acquire/release does *not* fix

**Initial:** `x = 0, y = 0`

**Threads:**

```cpp
// Thread 0
x.store(1, std::memory_order_relaxed);
int r1 = y.load(std::memory_order_relaxed);

// Thread 1
y.store(1, std::memory_order_relaxed);
int r2 = x.load(std::memory_order_relaxed);
```

### The “interesting” outcome

`r1 == 0 && r2 == 0`

Interpretation: each core did its store, but each still read the old value from the other location.

### Is `r1=0, r2=0` allowed?

* **x86 (TSO):** **allowed**. Classic explanation: **store buffers**.

  * Each core can perform its own store into a buffer (not yet globally visible),
  * then perform the load and still see the old value from memory / cache.
* **ARMv8:** **also allowed**.

### Does acquire/release fix SB?

Try:

```cpp
// Thread 0
x.store(1, std::memory_order_release);
int r1 = y.load(std::memory_order_acquire);

// Thread 1
y.store(1, std::memory_order_release);
int r2 = x.load(std::memory_order_acquire);
```

**`r1=0, r2=0` is still allowed.**

Why, precisely:

* Acquire/release only *forces order/visibility when an acquire-load actually reads the value written by a release-store on the same atomic.*
* In the `0/0` outcome, neither load reads-from the other thread’s store, so **no synchronizes-with edge** is formed.
* Also, release does **not** impose **Store→(later)Load** ordering within the same thread. That’s the specific hole that SB exploits.

### Outcome summary (SB)

| Orders used                  | Can `r1=0, r2=0` happen? | Why                                                                  |
| ---------------------------- | -----------------------: | -------------------------------------------------------------------- |
| relaxed / relaxed            |      **Yes** (x86 + ARM) | store buffering / weak ordering                                      |
| release store + acquire load |      **Yes** (x86 + ARM) | no reads-from pairing in the 0/0 case; also doesn’t block Store→Load |
| seq_cst everywhere           |        **No (portable)** | SC forbids the cyclic ordering required for 0/0                      |

### How to actually forbid `0/0`

**Option A (simple): make them seq_cst**

```cpp
x.store(1, std::memory_order_seq_cst);
int r1 = y.load(std::memory_order_seq_cst);
```

(and similarly in the other thread)

**Option B (targeted): insert a real Store→Load fence**

```cpp
x.store(1, std::memory_order_relaxed);
std::atomic_thread_fence(std::memory_order_seq_cst);  // acts like mfence/dmb-ish
int r1 = y.load(std::memory_order_relaxed);
```

On typical hardware:

* **x86:** fence → `MFENCE` (or equivalent)
* **ARMv8:** fence → `DMB ISH`

That fence specifically prevents “my store stays buffered while I perform a later load”.

---

## One-line takeaway

* **MP:** acquire/release is the right tool: “publish data, then publish flag” / “read flag, then read data”.
* **SB:** acquire/release is *not* enough; you need **seq_cst** or an explicit **Store→Load** barrier (or different algorithmic structure).

If you want, I can add a third litmus (e.g., **IRIW** or **Load Buffering**) to show where *per-location coherence* ends and where you start needing stronger global ordering than acquire/release.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; 1. Do I get it right that on x86 every store is "release" and every load is "acquire"?
<br>
2. Load/store reordering is allowed on x86?
<br>
3. Do I get it right that in the SB example you have above the threads are allowed to see the 'old' values since both the threads are not reading the memory location each of them updated itself (==has in its L1 cache store buffer), but rather are reading the value updated by the other thread which has not yet been propagated from the SB to L1 cache?
</td></tr>
</table>

### 1) “On x86 every store is release and every load is acquire” - yes, **for normal cached memory**, with some nuance

At the **hardware memory-model** level (x86-TSO, for ordinary WB cacheable memory):

* A plain **store** has **release-like** ordering: earlier loads/stores in program order won’t become visible *after* that store.
* A plain **load** has **acquire-like** ordering: later loads/stores in program order won’t be observed *before* that load.

That’s why in C/C++/Rust, `load(acquire)` / `store(release)` usually compile to plain `mov` on x86.

Nuance / footnotes:

* This is about **hardware ordering**. Language-level acquire/release also constrains **compiler reordering** (atomics do; non-atomics don’t).
* Weird memory types (UC/WC), non-temporal stores, device MMIO, etc., have different rules; I’m assuming normal RAM.

---

### 2) “Load/store reordering is allowed on x86?” - **only Store→Load (effectively)**

Under **TSO**, program order is preserved except that a later **load** may be performed/observed before an earlier **store** to a *different* location becomes globally visible.

So, allowed (architecturally):

* **Store → Load** reordering (the classic store-buffer effect)

Not allowed (architecturally):

* **Load → Load**
* **Store → Store**
* **Load → Store**

(Again, ignoring special memory types / NT stores.)

This is exactly why SB (“store buffering”) is possible on x86.

---

### 3) Your SB interpretation - basically right, here’s the precise picture

In the SB test:

```cpp
// T0
x = 1;      // goes into T0's store buffer (initially)
r1 = y;     // can execute before x=1 is globally visible

// T1
y = 1;      // goes into T1's store buffer (initially)
r2 = x;     // can execute before y=1 is globally visible
```

The `r1=0 && r2=0` outcome happens because:

* Each core’s **own store** can sit in its **store buffer** for a bit before it *drains* into the coherent cache hierarchy (at which point it becomes globally visible and triggers/participates in coherency actions).
* Meanwhile, each core’s **load of the other variable** (`y` for T0, `x` for T1) can complete **before** the other core’s store drains, so it reads the **old value** (0) from its cache / shared cache / memory.

And yes: if a core *did* load the **same location it just stored**, it would normally get **store-to-load forwarding** (read its own buffered store). But in SB, each thread reads the *other* location, so forwarding doesn’t apply.

So your summary is correct: the reads see “old” values because the *other core’s* store hasn’t become globally visible yet (store buffer not drained), not because of missing cache coherency.

---

If you want, I can also write the same SB example in terms of explicit x86 behavior (“store buffer + invalidate on drain”), and show what instruction(s) on x86 would forbid `0/0` (spoiler: you need something that orders **Store→Load**, e.g. `MFENCE` or a `LOCK`ed op / `XCHG`).

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Please do what you have suggested at the end of your previous reply + explain how (if at all) that relates to the sequential consistency.
</td></tr>
</table>

Let’s pin this to the **store-buffering (SB)** test and describe what *x86 actually does*, what **MFENCE / LOCKed ops** change, and how that connects to **sequential consistency (SC)**.

---

## 1) What’s happening on x86 in SB (store buffer + coherency)

SB test (one possible formulation):

```cpp
// initially x=0, y=0

// T0                        // T1
x.store(1, relaxed);         y.store(1, relaxed);
r1 = y.load(relaxed);        r2 = x.load(relaxed);
```

### Why `r1=0 && r2=0` can happen on x86 (TSO)

x86 cores have a **per-core store buffer**. A store typically:

1. **Enters the core’s store buffer** (so the core can keep executing).
2. Later **drains** into the cache-coherent hierarchy (L1 gets the line in Modified/Exclusive state, other cores get invalidations, etc.).
3. Only after draining does it become *globally visible* to other cores.

Meanwhile, loads can execute while an earlier store to a **different** address is still buffered (this is the one key relaxation of x86-TSO: **Store→Load**). The SB outcome is a textbook example of “non-SC behaviour allowed by store buffering” under TSO. ([Computer Laboratory][a4-1])

A concrete execution that yields `0/0`:

**Core 0**

* `x=1` goes into **SB0** (store buffer of core 0). Core 0 will see it via *store-to-load forwarding* **only if it later loads `x`** (it doesn’t).
* `load y` happens before Core 1’s `y=1` has drained → reads `y==0` from cache.

**Core 1**

* `y=1` goes into **SB1**.
* `load x` happens before Core 0’s `x=1` has drained → reads `x==0`.

Only later do SB0/SB1 drain, invalidating the other core’s cached copy and making `x=1`, `y=1` globally visible - but too late for the already-completed loads.

So yes: your “hasn’t propagated out of the store buffer yet” mental model is basically right - with the refinement that “propagation” is via **draining into the coherent cache hierarchy** (not “into L1 and stop”, but “into L1 *and* coherency traffic happens so other cores can no longer read the old line”).

---

## 2) What forbids `0/0` on x86: MFENCE or a LOCKed instruction

### A) Insert a Store→Load fence (`MFENCE`)

Assembly shape:

```asm
; T0
mov  dword ptr [x], 1
mfence
mov  eax, dword ptr [y]      ; r1

; T1
mov  dword ptr [y], 1
mfence
mov  eax, dword ptr [x]      ; r2
```

`MFENCE` is specified to order prior loads/stores before later loads/stores (i.e., it drains/blocks so you don’t get Store→Load reordering). ([Stack Overflow][a4-2])

With `mfence` in each thread, each load can’t run until *that thread’s* earlier store has become globally visible. That’s enough to make `0/0` impossible: at least one of the loads will occur after the other thread’s store has become visible.

(You can still get `0/1` or `1/0` depending on timing, just not `0/0`.)

### B) Use a LOCKed operation (e.g., `XCHG` with memory)

On x86, `xchg reg, [mem]` is **implicitly locked** when the memory operand is used. ([felixcloutier.com][a4-3])
And x86’s ordering rules prevent loads/stores from being reordered across locked instructions (they act as a full barrier for ordinary memory ordering purposes). ([cfsamsonbooks.gitbook.io][a4-4])

So you can write:

```asm
; T0
mov  eax, 1
xchg dword ptr [x], eax      ; locked
mov  eax, dword ptr [y]      ; r1

; T1
mov  eax, 1
xchg dword ptr [y], eax      ; locked
mov  eax, dword ptr [x]      ; r2
```

This also kills `0/0` for essentially the same reason: the locked op can’t “float” past the following load, and (critically) it forces the core to make the write globally visible as part of the locked transaction.

---

## 3) How this relates to sequential consistency (SC) and `seq_cst`

### Hardware SC vs x86-TSO

**Sequential consistency** means: *there exists a single global interleaving of all memory operations that respects each thread’s program order.*

In SB, getting `r1=0 && r2=0` would require:

* T0’s `load y` to occur before T1’s `store y`
* T1’s `load x` to occur before T0’s `store x`
  but also each thread’s program order says its store comes before its load.

That creates a cycle - no single interleaving can satisfy all of it - so **SC forbids `0/0`**. x86-TSO allows it specifically via store buffering / Store→Load relaxation. ([Computer Laboratory][a4-1])

Adding `MFENCE` (or a locked op) removes that Store→Load relaxation *for that code path*, making the execution SC-like for this pattern.

### C++ / Rust `seq_cst`

Language-level `memory_order_seq_cst` is essentially: *make atomics behave as if they participate in one global total order consistent with program order* (for seq_cst operations), which is strong enough to forbid SB’s `0/0`.

On x86, compilers commonly implement **seq_cst stores** using a locked instruction such as `xchg` (exactly because plain `mov` would still allow Store→Load via store buffers). The Cambridge “C/C++ mappings” explicitly notes `xchg`’s implicit LOCK behavior in this context. ([Computer Laboratory][a4-5])

And that matches what you see in real codegen: `acquire`/`release` often compile to plain `mov` on x86, while `seq_cst` stores often become `xchg` (or a `mov` + fence, depending on compiler/version/target).

---

### Quick “when do I need what” rule for x86

* **Message passing / publication:** release-store + acquire-load is enough (and usually just `mov`/`mov`).
* **Prevent SB (`0/0`) / enforce Store→Load order:** you need **MFENCE** or a **LOCKed op** (or `seq_cst`, which typically *uses* a LOCKed op / fence).

If you want, I can also show the *exact* C++ litmus code (with `relaxed`, `acq/rel`, `seq_cst`) and the corresponding x86 assembly patterns (mov/mov vs xchg/mov vs mov+mfence+mov), plus which outcomes each permits.

[a4-1]: https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-745.pdf?utm_source=chatgpt.com "A better x86 memory model: x86-TSO (extended version)"
[a4-2]: https://stackoverflow.com/questions/40409297/does-lock-xchg-have-the-same-behavior-as-mfence?utm_source=chatgpt.com "Does lock xchg have the same behavior as mfence?"
[a4-3]: https://www.felixcloutier.com/x86/xchg?utm_source=chatgpt.com "XCHG - Exchange Register/Memory With Register"
[a4-4]: https://cfsamsonbooks.gitbook.io/explaining-atomics-in-rust?utm_source=chatgpt.com "Explaining Atomics in Rust"
[a4-5]: https://www.cl.cam.ac.uk/~pes20/cpp/cpp0xmappings.html?utm_source=chatgpt.com "C/C++11 mappings to processors"

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Is there any difference between 'lock xchg' and 'mfence' on x86 both in terms of guarantees/semantics they provide and in terms of performance? Which is preferable when?
</td></tr>
</table>

Yes - there are differences, even though **both can act like a “full fence”** for normal cacheable (WB) memory.

## 1) Guarantees / semantics

### `MFENCE`

`MFENCE` is *pure ordering*:

* It guarantees that **every load/store before the MFENCE becomes globally visible before any load/store after it becomes globally visible**. ([Félix Cloutier][a5-1])
* It **does not** perform an atomic update of any memory location.
* It “does not serialize the instruction stream” (i.e., it’s about memory visibility/ordering, not “stop the world” retirement). ([Félix Cloutier][a5-1])

### `XCHG [mem], reg` (implicitly locked) / “`lock xchg`”

`XCHG` with a **memory** operand is a *locked instruction*:

* If a memory operand is referenced, the CPU’s **locking protocol is automatically implemented**, regardless of whether you write an explicit `LOCK` prefix. ([Félix Cloutier][a5-2])
* Locked instructions are **indivisible atomic RMW** operations (load(s) + store(s) as one uninterruptible unit). ([CMU School of Computer Science][a5-3])
* On WB memory, Intel’s own memory-ordering doc states:

  * **Locked instructions have a total order** (all cores agree on their order). ([CMU School of Computer Science][a5-3])
  * **Loads and stores are not reordered with locked instructions** (full-barrier effect w.r.t. surrounding loads/stores). ([CMU School of Computer Science][a5-3])

So as *a fence*, a locked `xchg` is at least as strong as `mfence` for the usual WB-memory model - but it also has a **side effect**: it’s an atomic read-modify-write on *some* cache line, and it participates in the global total order of locked ops.

**Important practical nuance:** In higher-level language terms (C++/Rust), this “total order of locked ops” is one reason compilers like using `xchg` to implement a `seq_cst` store on x86 (another valid choice is `store + mfence`). ([Stack Overflow][a5-4])

---

## 2) Performance differences

### Locked `xchg` cost profile

A locked op has to obtain **exclusive ownership** of the cache line and exclude interference on that line; that can trigger cache invalidations and, if contended, cache-line “ping-pong”.

Agner Fog summarizes it bluntly: **LOCK-prefixed instructions typically cost >100 cycles**, depending on cache/RAM effects, and this *also applies to `XCHG` with a memory operand*. 

Also, a locked op can have broader **system impact** (it creates coherence traffic and can hurt other cores), while a fence is more “local” in effect. ([Intel Community][a5-5])

### `MFENCE` cost profile

`MFENCE` doesn’t grab exclusive ownership of some arbitrary line, but it can still be expensive because it blocks progress until ordering conditions are satisfied.

Agner’s tables show **tens of cycles** latency/throughput impact for `MFENCE` on some µarches (e.g., values like ~23 / ~33 cycles show up in his listings). 
(Exact numbers depend heavily on microarchitecture and surrounding code.)

**Rule of thumb:** if you don’t actually need an atomic RMW, **a locked RMW is usually the more disruptive hammer**.

---

## 3) Which is preferable when?

### Prefer a locked op (`xchg`, `lock cmpxchg`, `lock xadd`, …) when:

* You **need atomic read-modify-write** semantics (locks, refcounts, CAS loops, etc.).
* You need an operation that participates in the **global total order of locked ops** (relevant for implementing strong language atomics like `seq_cst` in some mappings). ([CMU School of Computer Science][a5-3])

### Prefer `MFENCE` when:

* You **only** need ordering (especially the x86 “hole”: preventing **Store→Load** reordering / store-buffering effects), and you **don’t** want to touch a shared cache line just to get a fence. (`MFENCE` gives the “globally visible before/after” guarantee directly. ([Félix Cloutier][a5-1]))

### Real-world advice (most important)

In C++/Rust/.NET, you almost never want to hand-pick `mfence` vs `xchg` in application code:

* Use language atomics (`acquire/release/seq_cst`) and let the compiler pick the right mapping.
* On x86, `acquire/release` is usually “free” (plain loads/stores), while `seq_cst` often forces something expensive (locked op or fence). ([Stack Overflow][a5-4])

If you want, paste the exact pattern you’re considering (e.g., “I have SB and want to forbid 0/0” vs “I’m implementing a spinlock”), and I’ll tell you what I’d use and why for that case.

[a5-1]: https://www.felixcloutier.com/x86/mfence "MFENCE - Memory Fence"
[a5-2]: https://www.felixcloutier.com/x86/xchg "XCHG - Exchange Register/Memory With Register"
[a5-3]: https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf "Microsoft Word - Intel-64_MO_v1.0_publish.doc"
[a5-4]: https://stackoverflow.com/questions/49107683/why-does-a-stdatomic-store-with-sequential-consistency-use-xchg?utm_source=chatgpt.com "Why does a std::atomic store with sequential consistency ..."
[a5-5]: https://community.intel.com/t5/Intel-Moderncode-for-Parallel/LOCK-vs-MFENCE/td-p/898179?utm_source=chatgpt.com "LOCK vs MFENCE"

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Do I get it right that the 'higher level' synchronization primitives like mutex are implemented with lock-free techniques using acquire/release/barriers where needed or CAS/RMW operations ?
</td></tr>
</table>

Yes, that’s essentially right - with a few important precisions.

* **Mutexes/condvars/semaphores** are *not* “lock-free” in the sense of guaranteeing global progress without blocking. They are **blocking primitives**.
* But their **fast path** is usually built from **atomic RMW/CAS** plus **acquire/release** ordering, and only *then* they may fall back to the OS to block/wake threads.

## Typical mutex structure (conceptually)

### 1) Fast path: pure user-space atomics

A common design is:

* Lock: try to transition state `0 → 1` using an atomic RMW (CAS / test-and-set).

  * On success: you acquired the lock and proceed in user-space.
  * Memory order: the acquiring operation is **acquire**.

* Unlock: store state `1 → 0` (often an atomic store; sometimes an RMW depending on contention).

  * Memory order: the releasing operation is **release**.

Why acquire/release matters:

* **unlock (release)** makes all critical-section writes visible *before* the lock becomes available.
* **lock (acquire)** ensures that after you take the lock, you see what the previous owner wrote.

### 2) Contended path: blocking via OS help

If the lock is contended, most implementations avoid burning CPU:

* They spin briefly (adaptive spinning), then
* Park the thread using an OS primitive (Linux `futex`, Windows `WaitOnAddress`, etc.),
* And unlock will wake a waiter.

Even in this slow path, the *handoff* still relies on the same atomic state and ordering rules; the OS mechanism is just an efficient sleep/wakeup channel.

## How it maps to hardware (x86 vs weak arch)

* On **x86**, acquire/release often compile to plain loads/stores, but the **RMW** itself (e.g. `LOCK CMPXCHG`, `XCHG`) already has strong ordering properties.
* On **ARM/RISC-V**, acquire/release may require acquire/release-form instructions (e.g., `LDAR/STLR`) or fences, and RMW ops have explicit ordering bits / barriers.

## Minimal illustrative mutex sketch (C++-ish)

(Not production-quality, just to show the ingredients.)

```cpp
std::atomic<int> state{0}; // 0 unlocked, 1 locked

void lock() {
    int expected = 0;
    while (!state.compare_exchange_weak(expected, 1,
                                        std::memory_order_acquire,
                                        std::memory_order_relaxed)) {
        expected = 0;
        // spin / backoff; then possibly block (futex/OS) if contended
    }
}

void unlock() {
    state.store(0, std::memory_order_release);
    // if waiters exist, wake one (OS primitive)
}
```

That’s the core idea you described: **CAS/RMW + acquire/release (+ fences if required by the target ISA)**, with OS blocking on contention.

So: **yes** - higher-level sync primitives are built on atomics/RMW plus the required ordering (acquire/release / fences). Just don’t call the primitives themselves “lock-free”; they’re typically *implemented* using lock-free techniques on the fast path, but they provide blocking semantics.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; By 'lock-free' I meant the coding techniques that themselves do not (or simply cannot) use anything like mutex/semaphore/etc. But rather rely on a particular memory model for ordering, visibility and on RMW operations for atomicity.
</td></tr>
</table>

Yep - in that sense you’re describing **“atomics + memory-ordering + RMW as the only primitives”** (i.e., *no* mutex/semaphore/condvar inside the algorithm). That’s exactly the toolbox used for:

1. **lock-free / wait-free algorithms** (queues, stacks, hash maps, refcounts, etc.), and
2. the **core** of many “higher-level” primitives (mutexes, rwlocks, once-init, etc.) - even though those primitives may *also* call into the OS to block/wake.

The key distinction is **technique vs progress guarantee**:

* **Technique:** “I only use atomic loads/stores/RMW + acquire/release/fences.”
* **Property:** “This algorithm is *lock-free* / *wait-free* / *blocking*.”

A **mutex** is typically implemented using those *techniques* on the fast path, but the mutex **itself** is still a **blocking** primitive (a thread can be parked indefinitely if the owner is preempted or dies).

---

## How the “atomics-only” technique maps to the memory model

### Atomicity

You need **RMW/CAS** (e.g., `compare_exchange`, `fetch_add`, `xchg`) for:

* “claiming” a state transition (`UNLOCKED → LOCKED`)
* updating shared structure pointers (`head → newHead`)
* providing a **linearization point** (the single atomic step that defines “the operation happened here”).

### Ordering / visibility

You use memory order to connect independent locations:

* **Release**: publish prior writes (data) before publishing a pointer/flag/state.
* **Acquire**: once you observe the pointer/flag/state, you must observe the prior writes it protects.

This is the same pattern whether you’re:

* publishing a node in a lock-free queue, or
* handing off ownership in a lock/unlock pair.

---

## Examples that show the technique

### 1) Publication (no mutex, just ordering)

```cpp
// writer
data = 123;
ready.store(1, std::memory_order_release);

// reader
while (ready.load(std::memory_order_acquire) == 0) {}
int x = data;   // guaranteed to see 123
```

### 2) Lock-free stack push (CAS + release)

```cpp
struct Node { Node* next; int v; };
std::atomic<Node*> head{nullptr};

void push(Node* n) {
    Node* old = head.load(std::memory_order_relaxed);
    do {
        n->next = old;
    } while (!head.compare_exchange_weak(
        old, n,
        std::memory_order_release,   // publish node contents + link
        std::memory_order_relaxed));
}
```

A pop would typically use **acquire** on success so it can safely read `n->next` / `n->v`.

### 3) Mutex *core state machine* uses the same building blocks

A typical mutex does something like:

* CAS to acquire (**acquire**)
* store to release (**release**)
* if contended, it additionally uses OS sleep/wake (futex/WaitOnAddress/etc.)

So: same technique, different overall semantics (blocking).

---

## One more important footnote: memory reclamation

Once you go beyond flags and counters, lock-free DS need safe reclamation:

* hazard pointers, epochs/RCU, reference counting, etc.
  This is where people often discover that “atomics-only” is necessary but not sufficient.

---

So your statement is right if you phrase it like this:

> “Many synchronization primitives and lock-free data structures are built primarily from atomic loads/stores/RMW plus acquire/release (and fences when required), rather than using mutex/semaphore internally - but whether the *resulting primitive* is lock-free depends on its progress guarantee (mutexes aren’t).”

If you tell me what you’re implementing (spinlock, once-init, queue, etc.), I can point at the exact acquire/release edges and the usual linearization points.

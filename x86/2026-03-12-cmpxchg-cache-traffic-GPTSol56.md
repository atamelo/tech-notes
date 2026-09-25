# CMPXCHG vs. XCHG Spin-Waits: Cache-Line Ping-Pong, TTAS and PAUSE (ChatGPT)

*A conversation with ChatGPT: why spinning on a locked `CMPXCHG` floods the coherence fabric just like `XCHG`, Intel's recommended test-and-test-and-set spin lock (assembly and C++), what `PAUSE` does, and why a failed locked `CMPXCHG` still counts as a write.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Does implementing the spin-wait using <code>CMPXCHG</code> on x86 incur the same cache traffic flood as using <code>XCHG</code>?
</td></tr>
</table>

Yes - **if you spin by repeatedly doing a memory `CMPXCHG` attempt**, it creates essentially the **same kind of coherency/cache-line ping-pong problem** as repeatedly using `XCHG` on the lock word. `XCHG` with a memory operand is always locked on x86, and `CMPXCHG` used for lock acquisition is atomic only with `LOCK`; Intel’s instruction description also says that locked `CMPXCHG` causes the destination to receive a **write cycle regardless of whether the comparison succeeds**. In other words, even a *failed* locked `CMPXCHG` is not “just a read.” ([felixcloutier: XCHG][ref-xchg], [felixcloutier: CMPXCHG][ref-cmpxchg])

That is the core reason the traffic pattern is the same in spirit: every waiter keeps trying to get the cache line in an exclusive/modified state so it can perform a locked RMW, which makes the lock line bounce among cores. This is exactly what you want to avoid under contention. Intel’s optimization guidance for spin locks describes the better pattern: **spin using ordinary reads while the lock is held, and only issue the locked instruction when the lock appears free**. Intel explicitly describes a spinlock as checking whether the lock is free and then attempting acquisition with a locked instruction; if not free, the thread spins using a **read operation that typically completes from the local data cache**. ([Intel Optimization Manual][ref-intel-opt])

So the practical answer is:

- **Repeated `xchg [lock], reg` in the wait loop** → bad, coherency flood.
- **Repeated `lock cmpxchg [lock], reg` in the wait loop** → also bad, same basic flood.
- **Read-spin (`mov/cmp/test` loads only), then one `lock cmpxchg` when it looks free** → much better. ([Intel Optimization Manual][ref-intel-opt])

So the distinction is **not really `CMPXCHG` vs `XCHG`**. The real distinction is:

> **locked RMW on every iteration** vs **test-and-test-and-set**.

If you want, I can also draw the MESI/MOESI-style cache-line state transitions for the two cases.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; Yes, please do.
<br><br>
Also:
<br>
1. Explain in more detail why in the <code>CMPXCHG</code> case the same kind of coherency/cache line ping-pong happens
<br>
2. give me an implementation of the intel recommended approach
</td></tr>
</table>

Yes. Here is the concrete picture.

`lock cmpxchg [lock], 1` in a retry loop causes the same **kind** of coherency storm as `xchg [lock], reg` in a retry loop, because on x86 a locked `CMPXCHG` is not a “read-only failed try.” Intel states that with `LOCK`, `CMPXCHG` gives the destination a **write cycle regardless of the comparison result**; on failure, the old value is written back, and “the processor never produces a locked read without also producing a locked write.” Intel also states that `XCHG` with memory always uses locked semantics. ([felixcloutier: CMPXCHG][ref-cmpxchg], [felixcloutier: XCHG][ref-xchg])

So the issue is not “did the logical lock value change?” The issue is “did the core perform a locked read-modify-write on that cache line?” For a locked RMW on a cacheable single-line WB location, Intel says modern x86 usually uses **cache locking** rather than a full bus lock, and coherence makes other processors write back / invalidate as needed so the atomic operation can proceed. That means each waiter repeatedly tries to get the lock line under exclusive control in its own cache. ([felixcloutier: LOCK][ref-lock])

#### Why `CMPXCHG` ping-pongs the line

Assume core A currently holds the lock, so the lock word is `1`.

Now cores B, C, and D spin like this:

```asm
retry:
    mov eax, 0
    mov ecx, 1
    lock cmpxchg [lock], ecx
    jnz retry
```

What happens conceptually:

1. Each waiter executes a **locked RMW** against the same cache line.
2. A locked RMW is a write-class coherence event, not a passive load, because Intel says the destination receives a write cycle even on failure. ([felixcloutier: CMPXCHG][ref-cmpxchg])
3. To do that atomic write-class operation on a cached line, the core must obtain the line in an exclusive/modified form locally; Intel describes this as cache locking with coherence managing the other cached copies properly. ([felixcloutier: LOCK][ref-lock])
4. So B pulls the line over and invalidates competing copies as needed.
5. Then C does the same, pulling the line away from B.
6. Then D does the same, pulling it away from C.
7. None of them acquires the lock, because the value is still `1`, but they still generated ownership transfers / invalidations on every failed attempt. ([felixcloutier: CMPXCHG][ref-cmpxchg], [felixcloutier: LOCK][ref-lock])

That is the ping-pong.

In other words, with naive `lock cmpxchg` spinning, the lock line bounces like this:

```text
time →
B tries:   line -> B (exclusive for locked RMW)   fail
C tries:   line -> C (exclusive for locked RMW)   fail
D tries:   line -> D (exclusive for locked RMW)   fail
B tries:   line -> B                              fail
C tries:   line -> C                              fail
...
```

> Even though the stored value remains `1`, the **ownership** of the cache line keeps moving.

#### Why read-spinning is different

Intel’s recommended spinlock structure is: first check whether the lock is free; if it is not, spin using a **read operation** that “typically completes from the local data cache”; only when it appears free do a locked instruction to try to acquire it. ([Intel Optimization Manual][ref-intel-opt])

That changes the coherence story:

- While the lock is held, waiters do only ordinary loads.
- Ordinary loads can sit on shared copies of the line.
- Shared reads do **not** demand exclusive ownership every iteration.
- So during the hold period, the line does not bounce B→C→D→B on each poll. ([Intel Optimization Manual][ref-intel-opt])

The traffic picture becomes:

```text
Holder A owns line with lock=1

B/C/D spin with plain loads:
    they mostly keep/read shared copies locally
    no locked RMW flood while lock stays 1

A unlocks with plain store lock=0

Now B/C/D notice 0 and race:
    one or more attempt locked acquire
    brief burst of contention here
    winner gets lock
```

> So TTAS does **not** eliminate contention after unlock. It eliminates the constant invalidation storm **while the lock is still held**.

#### Side-by-side picture

##### Bad: TAS / unconditional locked retry
`xchg` every time, or `lock cmpxchg` every time

```text
lock held = 1

B: locked RMW -> needs exclusive ownership -> fail
C: locked RMW -> needs exclusive ownership -> fail
D: locked RMW -> needs exclusive ownership -> fail
B: locked RMW -> needs exclusive ownership -> fail
...
Result: line ping-pongs continuously
```

##### Better: TTAS
plain load while busy, locked op only on transition-to-free

```text
lock held = 1

B: load -> sees 1
C: load -> sees 1
D: load -> sees 1
B: load -> sees 1
...

A: store 0  (unlock)

B/C/D: now one or more try locked acquire
Result: mostly quiet while held; contention concentrated near unlock
```

#### Intel-recommended approach

Intel’s optimization manual literally describes the pattern as:

- check whether the lock is free,
- if not, spin with a read from the local cache,
- when it looks free, retry the locked acquire,
- include `PAUSE` in the wait loop. ([Intel Optimization Manual][ref-intel-opt])

### Assembly implementation using `CMPXCHG` (TTAS)

Here is a straightforward x86-64 version using `lock cmpxchg` only for acquisition, and plain load + `pause` while waiting.

```asm
; int lock == 0  -> unlocked
; int lock == 1  -> locked

; void spin_lock(int *lock)
spin_lock:
.retry:
    ; optimistic acquire attempt: try 0 -> 1
    xor     eax, eax          ; expected = 0
    mov     edx, 1            ; desired  = 1
    lock cmpxchg dword ptr [rdi], edx
    je      .acquired         ; success if ZF=1

.wait:
    pause
    cmp     dword ptr [rdi], 0
    jne     .wait             ; still locked, keep read-spinning
    jmp     .retry            ; looks free, try locked acquire once

.acquired:
    ret

; void spin_unlock(int *lock)
spin_unlock:
    mov     dword ptr [rdi], 0
    ret
```

Why this matches the recommendation:

- the wait loop uses only `pause` + plain load/compare,
- the locked RMW is attempted only when the lock looks free,
- unlock is a regular store. Intel’s simple spinlock example uses the same structure: wait with reads, acquire with a locked instruction, release with a regular store. ([Intel Optimization Manual][ref-intel-opt])

### C/C++ version

```cpp
#include <atomic>
#include <immintrin.h>

struct SpinLock
{
    std::atomic<int> value { 0 };

    void lock()
    {
        for (;;)
        {
            int expected = 0;
            if (value.compare_exchange_weak(
                    expected,
                    1,
                    std::memory_order_acquire,
                    std::memory_order_relaxed))
            {
                return;
            }

            while (value.load(std::memory_order_relaxed) != 0)
            {
                _mm_pause();
            }
        }
    }

    void unlock()
    {
        value.store(0, std::memory_order_release);
    }
};
```

That is the same TTAS algorithm in portable form.

#### One subtle but important nuance

It is better to say:

- **naive `lock cmpxchg` spinning and naive `xchg` spinning have the same pathology**, namely repeated exclusive ownership grabs on the same line,
- not necessarily that they are bit-for-bit identical in every microarchitectural detail or cost.

But for the purpose that matters here - **coherence traffic under contention** - they are in the same bad category. Intel’s own guidance is to avoid that by waiting with reads and using the locked instruction only when the lock appears free. ([Intel Optimization Manual][ref-intel-opt])

#### Practical improvement

Under heavy contention, add backoff to the read-spin phase. TTAS already removes the worst steady-state flood, but after unlock you can still get a herd of contenders all attempting the locked acquire at once. Backoff reduces that post-unlock burst; fairness is still a software concern, and Intel explicitly notes contested locks may require software algorithms for fairness. ([Intel Optimization Manual][ref-intel-opt])

If you want, I can next draw the same story in explicit MESI-state terms (`S`, `E`, `M`, invalidations, ownership transfer on failed `cmpxchg`).

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; 1. How does the '<code>pause</code>' work in the code above? Purpose?
<br>
2. You said above "A locked RMW is a write-class coherence event, not a passive load, because Intel says the destination receives a write cycle even on failure" - can you explain exactly why that seemingly read-only operation entails write in the comparison failure case?
</td></tr>
</table>

1. **`PAUSE` is not a sleep, yield, or unlock.** It is a **spin-wait hint** meant for short wait loops. Intel says it is intended to let the sibling logical processor on the same core use competitively shared resources more effectively, and to save power versus an equivalent tight loop without `PAUSE`. Intel also notes the latency is microarchitecture-dependent: older generations were around ~10 cycles, while Skylake Client extended it to as many as ~140 cycles. ([Intel Optimization Manual][ref-intel-opt])

2. In practice, `PAUSE` does three useful things in a spin loop:
   - it slightly throttles the loop instead of hammering the core with a pure `load/jcc/load/jcc/...` sequence,
   - it is friendlier to Hyper-Threading siblings because it gives up some shared front-end / scheduler opportunities,
   - it reduces power while you are polling.  
   But it does **not** release the lock, does **not** put the thread to sleep, and does **not** by itself reduce coherence traffic caused by locked RMWs; that traffic reduction comes from the fact that the loop around it uses **plain loads** while waiting, not from `PAUSE` alone. Intel’s optimization manual shows exactly this pattern: `cmpxchg` to try acquisition, then `while (lock == busy) pause;`, with optional backoff. ([Intel Optimization Manual][ref-intel-opt])

3. So the purpose of `PAUSE` in the code I gave is:
   - while the lock is still busy, poll with ordinary reads,
   - insert `PAUSE` so that polling is less hostile to the core and sibling thread,
   - retry the locked acquire only after the load sees the lock become free. ([Intel Optimization Manual][ref-intel-opt])

---

On your second question, the key point is:

4. A failed locked `CMPXCHG` is **not architecturally a read-only operation** on x86. Intel’s instruction definition explicitly says that with `LOCK`, “the destination operand receives a write cycle without regard to the result of the comparison,” and if the comparison fails, “the destination operand is written back.” The pseudocode in the manual reflects that too: on failure it does `accumulator ← TEMP; DEST ← TEMP;`. Intel also states: “The processor never produces a locked read without also producing a locked write.” ([felixcloutier: CMPXCHG][ref-cmpxchg])

5. So why did Intel define it that way? The manual gives the reason directly: **“to simplify the interface to the processor’s bus.”** In other words, hardware is not exposing a special external transaction meaning “perform a locked read, compare internally, and maybe later decide whether this should become a write.” Instead, locked `CMPXCHG` is treated as a locked read-modify-write class operation every time, even if the value ultimately written back is the same as the old one. ([felixcloutier: CMPXCHG][ref-cmpxchg])

6. That is why the failure case still causes the same coherence trouble: on modern Intel processors, for cached WB memory, locked operations generally use **cache locking** rather than asserting the external `LOCK#` bus signal. Intel says that in this case the processor modifies the memory location internally and relies on cache coherence to ensure atomicity. To do that, the core must get the line under write ownership/exclusive control for the locked operation. ([felixcloutier: LOCK][ref-lock])

7. In MESI-ish terms, the important event is not “did the bits change?” but “did this core have to get the line in a write-capable state?” A failed locked `CMPXCHG` still has to be treated as a write-class atomic operation, so the contender core must obtain ownership of the line; that invalidates or displaces other cores’ copies as required by coherence. Then another waiter does the same thing, and ownership bounces again. That is the ping-pong. Even if the local core ends up writing back the **same** value (`1` stays `1`), the coherence fabric still saw a write-intent atomic access. ([felixcloutier: CMPXCHG][ref-cmpxchg], [felixcloutier: LOCK][ref-lock])

8. Put differently:  
   - **plain load in spin loop** → can often stay satisfied from a shared cached copy; no need to grab write ownership each iteration.  
   - **locked `cmpxchg` in spin loop** → every try is a write-class atomic RMW, so every try wants ownership of the line.  
   That is the whole reason TTAS is better than “retry locked op on every iteration.” Intel’s own examples show exactly that split: `cmpxchg` for the acquisition attempt, then `while (lock == busy) pause;`, optionally with backoff. ([Intel Optimization Manual][ref-intel-opt])

9. One subtle point: whether the core literally rewrites the same bits in the local cache array on failure is a **microarchitectural** detail Intel does not spell out here. What **is** architecturally guaranteed is the write cycle / locked-write semantics, and that is enough to force the coherence consequences that matter for contention. ([felixcloutier: CMPXCHG][ref-cmpxchg])

If you want, I can next draw the exact MESI state transitions for:
- failed `lock cmpxchg` by B while A holds the lock, and
- the TTAS version with `pause` + plain loads.

[ref-xchg]: https://www.felixcloutier.com/x86/xchg "XCHG - Exchange Register/Memory With Register"
[ref-lock]: https://www.felixcloutier.com/x86/lock "LOCK - Assert LOCK# Signal Prefix"
[ref-cmpxchg]: https://www.felixcloutier.com/x86/cmpxchg "CMPXCHG - Compare and Exchange"
[ref-intel-opt]: https://cdrdv2-public.intel.com/821614/356477-Optimization-Reference-Manual-V2-050.pdf "Intel 64 and IA-32 Architectures Optimization Reference Manual, Volume 2 (356477-050)"

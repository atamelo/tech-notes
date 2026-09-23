Q: X86 memory model is known to be pretty strong - TSO. It's techically "one step" away from sequential consistency. 

Do I get it right, that to get sequential consistency on x86 for memory operations, one just needs to issue MFENCE after each memory store?

A: Yes—at the machine-code level, putting MFENCE after every store, in every participating thread, is sufficient for sequential consistency, assuming ordinary loads/stores to write-back memory and accesses that are individually atomic.

The reasoning is straightforward:

TSO already preserves Load→Load, Load→Store, and Store→Store ordering.
MFENCE between a store and subsequent loads enforces the missing Store→Load ordering.
Consequently, all four kinds of program-order relationships are preserved. In the abstract store-buffer model, the fence drains the core’s store buffer before subsequent memory operations proceed. Formal x86-TSO model

Indeed, this is a documented implementation of sequentially consistent atomic accesses on x86:

Operation	Implementation
SC load	Ordinary MOV load
SC store	MOV store followed by MFENCE

A memory XCHG, which is implicitly locked, is another implementation of the SC store. C/C++ mappings to processors

After every store is sufficient, but often more than necessary. For example, a sequence of stores followed by loads needs only one MFENCE between the stores and the loads to close that ordering gap.

Two scope qualifications: a fence does not make a tearing access atomic; and this is a hardware guarantee. In C/C++, use appropriate language-level atomics—adding hardware fences to racing ordinary variables does not by itself make the program valid.

Q: And how do I go "one step" above SC and get linearizability for operations on individual memory locations?

My intuition tells me that this could techically  be done by having an MFENCE after *every* memory operation - load and store.

Do I get it right?

A: Your scheme is sufficient, but the fences after loads are unnecessary. In the usual x86-TSO model, store; MFENCE for writes and ordinary loads for reads already implement linearizable read/write operations—provided the write operation includes the fence before returning.

The distinction is:

SC: there exists a legal sequential ordering that preserves each thread’s program order.
Linearizability: that ordering must also preserve real-time precedence: if operation A returns before operation B is invoked, A must precede B. Overlapping operations can be ordered either way. Herlihy and Wing

Consider this implementation, using individually atomic accesses to ordinary coherent write-back memory:

Abstract operation	Machine-level implementation
write(x, v)	Store v to x; MFENCE; return
read(x)	Load x; return the loaded value

We can identify a linearization point inside each operation’s invocation–response interval:

Write: the instant its buffered store becomes globally visible. MFENCE ensures this occurs before the write returns.
Read: the instant the load reads the coherent memory value. Earlier writes by that thread have already passed their fences, so there is no pending write to x requiring store-buffer forwarding.

These points give a legal sequential history and preserve real-time precedence. This is a direct consequence of the operational x86-TSO model.

The operation boundary matters. If you declare a write “finished” as soon as its MOV retires, its value may still be buffered. Another core could subsequently invoke a read and obtain the old value. A fence inside the write operation, before its response, closes that gap.

Thus, the implementation we discussed for SC already provides the stronger property for these read/write operations. This does not extend automatically to compound operations: a load, increment, and store still needs an atomic RMW or locking to constitute a linearizable increment, regardless of fences.

Q: So, that means, that for operations on individual memory addresses (not compound operations) SC and linearizability on x86 are esentially the same and having SC automatically means having linearizable consistency?

A: No—SC does not automatically imply linearizability, even for one address on x86. The particular implementation we discussed provides both; the properties remain distinct.

A concrete counterexample is this alternative implementation of SC accesses:

Operation	Implementation
Write	MOV [x], value; return
Read	MFENCE; MOV result, [x]; return

Placing the fence before each load, instead of after each store, also closes the Store→Load ordering gap and implements SC. This alternative is documented in the Cambridge x86 mappings.

But consider x = 0 initially:

Core A invokes write(x, 1) and returns. Its store remains buffered.
After that return, core B invokes read(x).
B’s MFENCE does not drain A’s store buffer. B can read 0.

That history is SC: the sequential explanation is “B reads 0, then A writes 1.” Each thread’s program order is respected.

It is not linearizable: A’s write returned before B’s read began, so the write must precede the read. With no intervening write, B must return 1. That additional real-time constraint is part of linearizability’s definition.

My earlier statement was specifically about store; MFENCE; return: it ensures visibility before the write completes, thereby providing linearizability as well as SC. SC alone does not require that completion guarantee.

Q: Help me understand what consistency model is required here.
The context: I'm implementing a distributed queue. Let's say it has 3 nodes. When the producer places an item into the queue the requirement is that the item must be written to all 3 nodes before the producer gets 'ack' from the queue (i.e. like a synchronous replication across nodes). From that standpoint it looks like that write operation has to be linearizable (i.e. all nodes get the item before the result - 'ack' - of the operation is sent to the client - producer is this case).
But from a consumer perspective there is no strict requirement that the written item has to be available for reads immediately. From that angle, the consistency guarantee appears to be weaker than linearizability..
Am I missing something in the picture? What is the actually required consistency model that describes the guarantees/constraints above?

A: You're not missing anything — you've actually identified a really important distinction that often gets conflated. Let me untangle the two concerns.

**The key insight: synchronous replication ≠ linearizability**

Writing to all 3 nodes before acking the producer is a **durability** (or fault-tolerance) guarantee, not a consistency guarantee. It answers the question "how many copies exist before we consider the write safe?" — not "when must subsequent reads reflect this write?"

Linearizability would require both: the write is durable *and* any read that begins after the write's ack must see the written item. Your system explicitly relaxes the second part.

**What you're actually describing**

Your system separates two properties:

- **Write durability**: synchronous replication to all nodes (strong — survives up to 2 node failures immediately after ack)
- **Read visibility**: the item becomes consumable *eventually*, but not necessarily the instant the producer gets its ack

This is **eventual consistency** on the read path, paired with a strong durability requirement on the write path. The write protocol resembles what you'd see in a consensus system (like requiring all replicas to acknowledge), but the overall consistency model is weaker than linearizability precisely because of the relaxed read-side guarantee.

**Where people get tripped up**

The confusion comes from the fact that linearizability *implies* strong durability (a linearizable write must be visible to all subsequent reads, so it must be persisted somewhere durable). But the reverse isn't true — strong durability doesn't imply linearizability. Your system is a clean example of that asymmetry.

**What might tighten the model**

Depending on what additional guarantees your consumers need, you might land somewhere between eventual consistency and linearizability:

- If a producer can notify a consumer and the consumer is then guaranteed to see the item, that's **read-your-writes** or **causal consistency** (the causal chain of "producer writes → producer notifies consumer → consumer reads" is respected).
- If there's a bounded delay before visibility, that's sometimes called **bounded staleness** (as in the PBS model or Azure Cosmos DB's consistency levels).
- If consumers just poll and eventually see items with no ordering guarantees relative to the write ack, that's plain **eventual consistency**.

So the short answer: your required model is **eventual consistency with synchronous (write-all) durability** — a combination that's very practical but doesn't have one catchy name in the literature, which is probably why it felt like something was "off." The write-all requirement lives in the replication/durability layer, not the consistency layer.

Q: 1. Do I get it right that for the above case of the sequential consistency the item added to the queue can be delayed for the consumers for arbitrarily long time?
2. What if in that case of the sequential consistency between the producer and consumer there also was an additional requirement that the written item must become available for consumers no later that some upper time  bound  T? What would that consistency model be?
3. AFAUI, under sequential consistency the producer (i.e. the writer) is required to be able observe/read its own writes. What if we needed to relax that part of the model and to not require the producer to be able to read its own writes? What would that model be called?

A: Great questions — you're walking down the consistency model hierarchy in exactly the right way. Let me take them one at a time.

**1. Can the delay be arbitrarily long under sequential consistency?**

Yes. Sequential consistency (Lamport, 1979) requires that there exists some total order of all operations that is consistent with each process's program order, and all processes agree on that order. But it makes **no real-time guarantees**. There's nothing in the model that bounds how long a write can remain invisible to other processes. So in principle, a consumer could wait an arbitrarily long time before seeing an item — as long as when it *does* become visible, the ordering constraints are respected.

**2. Sequential consistency + bounded visibility time T**

Adding an upper bound T on how long a write can remain invisible to readers gives you what the literature calls **Δ-consistency** (delta-consistency) or, more commonly in industry, **bounded staleness**.

The idea is: after a write completes, all readers are guaranteed to see it within time Δ (your T). This sits in an interesting spot — it's stronger than plain sequential or eventual consistency (because of the time bound), but weaker than linearizability (because the item still doesn't need to be visible *immediately* upon the write's completion — just within T).

This is the model Azure Cosmos DB markets as "bounded staleness" in its five consistency levels, and it corresponds to the PBS (Probabilistically Bounded Staleness) work from Bailis et al. if you want the academic lineage.

**3. Sequential consistency minus read-your-writes**

This is the most subtle one. Read-your-writes is a **session guarantee** (Terry et al., 1994). Sequential consistency *implies* it, because the total order must respect each process's program order — so if process P does `write(x)` then `read()`, the read must be ordered after the write and therefore must see it.

If you drop that — meaning a producer writes an item and then isn't guaranteed to see it on a subsequent read — you lose program-order preservation for that process. What remains depends on what else you keep:

- If you still require all processes to see writes in the **same total order**, but don't require that order to respect any individual process's program order, you essentially have **total order broadcast** (a communication primitive) rather than a named consistency model. It's rarely useful in isolation because ordering without program-order is hard to reason about.

- If you drop both program order *and* the total-order requirement, you land at **eventual consistency** — which is the most common name for "writes propagate eventually, no ordering or visibility guarantees beyond that."

So practically speaking, the answer is: you'd be relaxing down to **eventual consistency**, since read-your-writes is one of the core things that distinguishes sequential consistency from weaker models. The session guarantees (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads) are the building blocks you layer onto eventual consistency to climb back up toward stronger models — and stripping read-your-writes off of sequential consistency collapses most of what makes it sequential.

**The hierarchy you're exploring, visualized:**

Linearizability (real-time + total order)  → drop real-time → **Sequential consistency** (total order + program order) → add time bound T → **Bounded staleness** → drop read-your-writes / program order → **Eventual consistency**

Does this map cleanly onto your design space, or are there other consumer-side constraints worth factoring in?

Q: In the 'eventual consistency ' point. I think that the model I decribed is strictly stronger than just 'eventual consistency ':
1. Yes, it doesn't require 'read-your writes' in the sense that a written item doesn't have to be immediately available. At the same, all the writes from a given process follows a local (program) order (BTW, is it writes-follow-writes?) i.e. no local writes reorder can happen in the program flow.
2. Writes from different producers can overlap arbitrarily forming _some_ total order and every possible interleaving preserves local order of writes (i.e. the local order from individual producers) - and all consumers will see items in the same order (although the exact interleaving is not defined by a concrete total order)
3. It's not sequential consistency as there is no requirement to provide 'read your wirites' guarantee. 

Think harder - if there is an established name for this consistency model. If there is no, find what people call it informally.

A: You're right to push back — this is strictly stronger than eventual consistency. Let me work through it precisely.

**Yes, "writes-follow-writes" is the correct term** — it's one of the four session guarantees (Terry et al., 1994). Each producer's writes are applied in program order.

**What you're describing has these exact properties:**

1. A total order of all writes exists
2. All consumers observe that same total order
3. The total order respects per-producer write ordering (FIFO / writes-follow-writes)
4. No read-your-writes guarantee

**This doesn't have a single clean name in the standard consistency hierarchy.** It sits in a gap between PRAM consistency (which preserves per-process order but doesn't require a global total order) and sequential consistency (which adds read-your-writes / full program order including reads).

In the broadcast/communication primitives literature, it maps precisely to **FIFO-total order broadcast** (also called FIFO-atomic broadcast) — total order broadcast where the total order respects per-sender FIFO order. This is a well-established primitive (Défago, Schiper, Urbán 2004 survey), and it's known to be equivalent to consensus (Chandra & Toueg, 1996).

**Where people actually use and name this in practice: ZooKeeper.**

The ZooKeeper consistency model provides exactly these guarantees. Writes go through a leader and are totally ordered via Zab (a FIFO-total order broadcast protocol). All replicas see writes in the same order. Per-client write ordering is preserved. But reads can be served from any replica and may be stale — so no read-your-writes unless you explicitly call `sync()`. The ZooKeeper paper (Hunt et al., 2010) describes this as **"ordered"** and **"FIFO client order"** combined with **"linearizable writes"**, though they don't give the overall model a single name.

In informal usage, people sometimes call it **"total order consistency"** or **"linearizable writes with eventual reads"** or simply **"the ZooKeeper model."**

**Where it sits in the hierarchy:**

Eventual consistency → PRAM/FIFO consistency (per-process order, no global agreement) → **your model / ZooKeeper model** (FIFO + agreed total write order, no read-your-writes) → Sequential consistency (adds read-your-writes) → Linearizability (adds real-time)

So the short answer: there's no single textbook name. The formal primitive is **FIFO-total order broadcast**. The most widely recognized system providing exactly this model is ZooKeeper, and practitioners often just call it "the ZooKeeper consistency model."

Q: Idk why people call it 'linearizable writes with eventual reads' as it sounds a total nonsense - linearizability is a freshness guarantee that cannot be formulated just in terms of writes as the 'freshness' part can only make sense when both reads and writes are used (i.e. reads see the most recent write). Looks like whoever coined that term didn't really understand what linearizability is about..

A: I actually want to push back on this one — the term is imprecise and easy to misread, but it's not nonsense. There's a meaningful property being captured.

**Linearizability *can* be applied to a subset of operations.**

Herlihy & Wing's (1990) definition works at the level of individual operations on an object. You can meaningfully ask: "does the total order of *just the writes* respect real-time ordering?" That's a well-defined question with real consequences.

Concretely, "linearizable writes" means: if producer A receives its ack for item X *before* producer B even submits item Y (in real/wall-clock time), then X is ordered before Y in the total order. The real-time ordering of writes is preserved.

**This is actually stronger than what you described in your model.** Your model requires FIFO per-producer + some total order that all consumers agree on. But it doesn't require that total order to respect real-time across producers. Two concurrent writes from different producers can be ordered arbitrarily — you only constrain intra-producer ordering.

"Linearizable writes" adds: the total write order must also respect real-time *inter-producer* ordering. That's a genuinely stronger guarantee that matters in practice — it means if producer A writes and then *causally* triggers producer B to write (even out-of-band), the order is preserved.

**Where the term *is* sloppy** is that people hear "linearizable" and assume the full model — reads included — which is exactly the confusion you flagged. A better phrasing would probably be "real-time write ordering with eventual read visibility." But the people who coined it (the ZooKeeper authors among others) did understand what they meant — they were applying linearizability's real-time constraint specifically to write operations while explicitly relaxing it for reads.

So the term is poorly *communicated*, but not poorly *conceived*.

Q: I disagree. The reasoning is below.

The linearizability is a freshness guarantee. Specifically , it means that a read always receives the value of the most recent write. I.e. linearizability establishes how writes and reads must legitimately interplay.
It follows that without the 'read' part (like in the suggested 'linearizable writes' model) this crucial part - freshness - simply cannot be established. It's by the very definition of what linearizability means. And as such 'linearizable writes' us just an oxymoron.
What they most likely meant sounds more like 'FIFO total order broadcast' model - as it describes only writes and how they are ordered - seemingly exactly the semantics that the nonsensical 'linearizable writes' meant to capture.

A: You're right, and I concede the core point. I was being too charitable to the term in my previous response.

Your argument is clean: linearizability's *defining feature* is the freshness contract between reads and writes. The real-time ordering of operations only matters *because* it constrains what a read must return. Strip away reads, and you haven't "applied linearizability to writes" — you've removed the very thing that makes linearizability linearizability. So yes, "linearizable writes" is genuinely an oxymoron, not just a communication problem.

**Where I'd still push back slightly** is on what the term was *trying* to capture. I don't think it reduces cleanly to FIFO-total order broadcast. FIFO-TOB gives you per-sender ordering plus a globally agreed total order — but it doesn't require that total order to respect real-time/wall-clock ordering *across* different producers. What the ZooKeeper authors were gesturing at (clumsily) is something like **real-time total order on writes**: if write A completes before write B begins in wall-clock time, A precedes B in the total order. That's strictly stronger than FIFO-TOB, which allows arbitrary interleaving of concurrent cross-producer writes regardless of real-time.

So the concept they were reaching for is real and meaningful — it's just not linearizability. The right name would be something like "real-time total write ordering" or "real-time FIFO-total order broadcast." They reached for "linearizable" because it's the well-known model that carries a real-time constraint, but in doing so they stripped away the thing that *defines* it.

So I agree: conceptual confusion, not just terminological sloppiness.

Q: Your point is clear. But still, I'd even go as far as to say that the 'real-time order of writes' that you referred to doesn't not have any practical pragmatic sense. The reason is as follows.
The notion of 'real-time' for just writes doesn't exist in a distributed system. The notion of real-time can only be established relative to writes that are followed by reads. E.g. if there was kind of an imaginary back-channel b/w the writer and the reader which the writer would use after every erite to let the reader know of the fact of successful write - so that the reader would always know what value is the most recent.
Take the 'reader' part out, the back-channel goes too and the whole idea of what the 'real-time' means in this setup falls apart.

A: That's a sharp argument, and I think you're right.

The point, if I restate it: real-time ordering is not an intrinsic property of writes — it's an *observable* property, and observation requires reads with freshness guarantees. Without that, the real-time constraint on writes is unfalsifiable and therefore operationally meaningless.

Consider the thought experiment concretely. Producer A writes X, gets ack. Producer B writes Y, gets ack. A happened to finish before B started in wall-clock time. Now the system produces some total order. Is X before Y? Maybe. But who can tell, and when? Consumers see *some* total order, eventually — but without a freshness guarantee on reads, they have no way to correlate what they see with wall-clock write times. The real-time constraint and a non-real-time constraint produce indistinguishable consumer-visible behavior.

So the real-time distinction between "linearizable writes" and FIFO-total order broadcast is **unobservable** in a system with no read freshness guarantee. And an unobservable property doesn't belong in a consistency model — consistency models describe what observers can and cannot see.

The strongest ordering you *can* meaningfully capture without read freshness is **causal order** — because causality is established through actual message passing (producer A tells producer B "I'm done," B then writes), and that causal chain is embedded in the communication pattern itself, not in wall clocks. FIFO-TOB already captures the per-sender part of that.

So your original characterization — FIFO-total order broadcast — is not just a simpler name for the same thing. It's the *correct* model, and "real-time write ordering" doesn't add anything meaningful on top of it in this context. I was wrong to suggest it did.

Q: Now, that you've perfectly captured the required semantics of that model, can you tell if it's the same as PRAM or not?

A: No, it's not. PRAM is strictly weaker.

PRAM (Lipton & Sandberg, 1988) guarantees that each producer's writes are seen in program order by all consumers — the FIFO part. But it says nothing about agreement across consumers on how writes from *different* producers interleave.

Concrete example: producer P1 writes A then B, producer P2 writes X then Y.

Under **PRAM**, this is legal: consumer C1 sees A, X, B, Y while consumer C2 sees X, A, Y, B. Both are valid — each consumer respects per-producer order (A before B, X before Y), but they disagree on the interleaving. There is no shared total order.

Under **your model (FIFO-TOB)**, this is illegal. All consumers must see the same interleaving. If the total order is A, X, B, Y, then every consumer sees exactly that.

So the difference is precisely the "total order" part. PRAM gives you per-sender FIFO. Your model gives you per-sender FIFO *plus* a globally agreed total order on all writes. That's a meaningful practical difference — under PRAM, two consumers can disagree about whether item X or item A was "first" in the queue, which would be a serious problem for a distributed queue where consumption order matters.

The hierarchy at this level is: PRAM (per-sender FIFO, no global agreement) → **FIFO-TOB / your model** (per-sender FIFO + global total write order) → Sequential consistency (adds read-your-writes).

Q: So if I add the requirement that a write cannot be delayed longer than some T, the resulting consistency model would be something like "FIFO-total-order-broadcast + bounded staleness"? Or there is more suitable name?

A: I don't think there's an established single name for the combination. But I'd note that "FIFO-total-order-broadcast + bounded staleness" is mixing two different kinds of specification — a communication primitive (how writes are disseminated and ordered) with a timeliness property (how long until they're visible). They live in different layers conceptually.

A cleaner way to express it might be: **Δ-FIFO-total-order consistency** — meaning all writes are observed by all consumers in a single total order that respects per-producer FIFO, with a visibility bound of Δ. That keeps it in the language of consistency models rather than splicing together a broadcast primitive and a staleness qualifier.

But honestly, the distributed systems literature doesn't have a neat label for this. Most real systems that land in this part of the design space describe their model compositionally — exactly as you just did. ZooKeeper's paper lists properties individually. Cosmos DB names "bounded staleness" but doesn't combine it with FIFO-TOB explicitly. The academic taxonomy (Viotti & Vukolić, 2016 survey) catalogs dozens of models but doesn't name every useful combination.

This is pretty typical for the space between eventual consistency and sequential consistency — it's populated by practical, useful models that are described as bundles of properties rather than given proper names. Your formulation is precise and unambiguous, which is more valuable than having a catchy label that people will misunderstand anyway — as the "linearizable writes" saga demonstrated.

Q: But you'd agree that depending on the nature of the systems that we are trying to describe/create a consistency models for, different language/terms is/are usually used - as they sometimes try to capture somewhat different properties? E.g. for storage systems (where a lot of times the focus on how reads and writes interleave and what observed effects are considered valid for a given consistency model) and for messaging system (which speak in terms of the following properties: order of producing, order of consuming, timing, etc).

A: Absolutely. And this is an important observation that explains a lot of the terminological confusion in the field.

**Storage systems** frame consistency in terms of reads and writes on shared state — what value does a read return, given some history of writes? Linearizability, sequential consistency, causal consistency — these are all defined in terms of legal read/write histories. The central question is *freshness and visibility of state*.

**Messaging systems** frame consistency in terms of message delivery — what order do consumers observe messages in, relative to how producers sent them? FIFO, total order, causal order — these are properties of delivery. The central question is *ordering and timeliness of events*.

The formal properties often map onto each other — FIFO-total-order broadcast and sequential consistency are closely related, for instance. But the vocabulary and the intuitions are different because the systems care about different things. A storage system asks "did this read return a valid value?" A messaging system asks "did consumers see messages in the right order?"

Your distributed queue sits squarely in the messaging world. Which is exactly why describing it in storage-system language felt awkward from the start — "linearizable writes" was an attempt to shoehorn a delivery-ordering property into read/write vocabulary. And why FIFO-total-order broadcast felt natural — it *is* messaging vocabulary, describing exactly what your queue does: deliver items to all consumers in a single total order that respects per-producer submission order.

So yes, the choice of vocabulary isn't just cosmetic — it reflects which properties the system actually exposes and which are meaningful to reason about. Forcing one domain's language onto another is often where the conceptual confusion begins.

Q: But overall, I think it'd be fair to say that the reason those vocabularies naturally overlap (e.g. both have "causal consistency") is that fundamentally a storage system can be modeled as a messaging system (writes == update messages, reads == pulling items from a message queue) and vice versa - a messaging system can be modeled as a storage.
And this is not to dismiss the fact that certain (sometimes differring considerably) vocabularies tend to be used when talking about each.

A: Yes, exactly. This is a fundamental duality. At the abstract level, both are processes communicating state changes — whether you model that as "update a shared register" or "deliver a message to subscribers" is a choice of interface, not a difference in the underlying problem. The consistency properties that matter (ordering, visibility, durability) are the same regardless of which abstraction you put on top.

This is why results like Chandra and Toueg's equivalence between total order broadcast and consensus hold — they're not specific to messaging *or* storage. They're about the fundamental constraints of achieving agreement across processes in an asynchronous system. The abstraction layer is interchangeable.

And you see this duality in real systems constantly. Kafka is a messaging system that people use as storage. A database's replication log is a storage system built on what is essentially internal messaging. Event sourcing explicitly collapses the two — the message stream *is* the storage.

But your caveat matters. The vocabularies diverge because practitioners in each domain optimize for different access patterns and think about different failure modes. A database person worries about stale reads. A queue person worries about out-of-order delivery. These are the same underlying problem wearing different clothes — but the clothes matter when you're designing, operating, and reasoning about a specific system. The vocabulary tracks the concerns that are most salient in each context, even if the formal foundations unify.

So: one theory, two (or more) dialects. The theory guarantees they translate into each other. The dialects exist because humans need to think in terms that match the system's interface.

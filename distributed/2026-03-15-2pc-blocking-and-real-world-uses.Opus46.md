# Two-Phase Commit: Blocking in Theory vs. Kafka and CockroachDB in Practice (Claude Opus 4.6)

*A conversation with Claude Opus 4.6: why 2PC's coordinator-failure blocking problem doesn't really bite Kafka and CockroachDB - their coordinator state is replicated and recoverable, backed by timeouts and active recovery.*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; The two-phase commit protocol (2PC) is known for being prone to blocking in case the coordinator fails.
<br><br>
And yet, certain real-world distributed systems (e.g. Kafka, CockroachDB) chose to use it for cross node/topic atomicity in transactions.
<br><br>
Can you explain why they chose so and if that choice really exposes them to the aforementioned 2PC deficiency in practice?
</td></tr>
</table>

This is a great question that gets at the gap between textbook distributed systems theory and production engineering tradeoffs. Let me walk through it.

**The textbook problem with 2PC**

The classic blocking scenario: after participants vote "yes" in phase 1, if the coordinator crashes before sending the phase 2 decision, participants are stuck. They've promised to commit but don't know the final decision, and they can't safely abort (the coordinator might have told someone else to commit) or commit (the coordinator might have decided to abort). They hold locks and wait, potentially forever.

The theoretical alternative is 3PC, which adds an extra round to avoid this blocking. But 3PC is almost never used in practice because it's more expensive, more complex, and - critically - it doesn't actually help under network partitions (it can lead to inconsistency in asynchronous networks, which is what real systems face).

**Why Kafka and CockroachDB chose 2PC anyway**

The key insight is that these systems don't use naive textbook 2PC. They've each made architectural choices that neutralize the blocking problem in practice.

*CockroachDB* uses a variant sometimes called "parallel commits" built on top of a Raft-replicated transaction record. The coordinator isn't a single fragile node - it's a replicated state machine. If the node driving the transaction dies, another node can read the transaction record from Raft, determine its state, and either push it forward or abort it. The transaction record itself is durable and replicated, so the "coordinator crashes and nobody knows the decision" scenario simply doesn't arise. Any node that encounters uncertain locks from an abandoned transaction can look up the transaction record and resolve them. This turns the blocking problem into a brief availability hiccup (the time to detect failure and recover), not an indefinite block.

*Kafka's transactional protocol* (for exactly-once semantics across partitions) uses a dedicated Transaction Coordinator backed by a replicated internal topic (`__transaction_state`). The coordinator's state - including the prepare/commit decision - is durably logged to this compacted, replicated topic before being acted upon. If the coordinator broker dies, another broker takes over, reads the log, and finishes the protocol. There's also a transaction timeout: if a producer disappears mid-transaction, the coordinator will proactively abort it after the timeout expires. So again, indefinite blocking is avoided.

**The common pattern**

Both systems apply the same fundamental trick: make the coordinator's state durable and recoverable by replicating it through a consensus protocol (Raft for CockroachDB, Kafka's ISR-based replication for Kafka). This means coordinator failure isn't the catastrophe that textbook 2PC assumes - it's a recoverable event. The remaining 2PC structure then gives you exactly what you want: atomic commit across multiple shards/partitions with minimal round trips.

They also both add **timeouts and active recovery**. Participants don't passively wait forever; there are mechanisms to detect stalled transactions and resolve them (abort on timeout, or consult the replicated transaction record to learn the decision).

**Does the 2PC deficiency still bite them in practice?**

In a narrow sense, yes - there is a window of reduced availability. Between the coordinator failing and recovery completing, in-flight transactions touching those ranges/partitions may stall briefly. Locks might be held for seconds rather than milliseconds. But this is qualitatively different from the textbook problem in important ways. It's *bounded* (timeouts ensure it doesn't last forever), *recoverable* (the replicated state lets a new coordinator finish the job), and *partial* (only transactions in flight at the moment of failure are affected, not the whole system).

So the honest answer is: they chose 2PC because the theoretical blocking problem is solvable with good engineering (replicated coordinator state + timeouts + active recovery), and the alternatives (3PC, consensus-based atomic commit like Paxos Commit) are either broken under partitions or significantly more expensive per transaction. 2PC with a replicated coordinator gives you the atomicity guarantee at roughly the cost of one extra round trip, which is hard to beat.

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; So, in summary, afaiui, the "paper/theoretical" 2PC implies that the transaction state is stored by the coordinator node and is unavailable if the coordinator crashes.
<br><br>
But if the transaction state (i.e. the coordinator state) is externalized and relocated (for availability/failure tolerance) then the coordinator node crash is a totally recoverable event and does not lead to 2PC being blocked..
<br><br>
Right?
</td></tr>
</table>

Yes, that's exactly right. You've nailed the core insight.

The textbook 2PC blocking problem is fundamentally a consequence of the coordinator being a single point of failure *whose state dies with it*. The moment you decouple the coordinator's *role* (driving the protocol) from the coordinator's *state* (the transaction record / decision), the blocking problem collapses into a temporary availability blip rather than a potentially indefinite stall.

One small nuance worth adding: externalizing the state is necessary but not quite sufficient on its own. You also need a mechanism for *someone else to pick up where the failed coordinator left off* - meaning the system needs a way to detect the failure, elect or assign a new coordinator, and have that new coordinator read the replicated state and drive the protocol to completion. That's why these systems pair the replicated transaction log with things like leader election (Raft in CockroachDB, controller-based reassignment in Kafka) and active transaction recovery routines.

But yes, the fundamental point stands: the classic 2PC vulnerability is really about co-locating ephemeral coordinator state with a mortal process. Externalize and replicate that state, add recovery machinery, and you keep 2PC's elegant simplicity and low overhead while sidestepping the problem that made theorists nervous about it for decades.

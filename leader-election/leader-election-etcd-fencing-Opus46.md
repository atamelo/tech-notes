Q: Suppose I have a distributed system with replication (say, factor 3) with a leader shard and two followers. All the operations go through the leader. So the writing party needs to know leader's address. In case the leader changes (failover), it's address changes too. What are the most common ways (used in practice) to let the writing parties know of this new address?

A: Great question — this is one of the core problems in distributed systems. There are several well-established patterns used in production, each with different trade-offs.

1. Centralized configuration/coordination service

This is probably the most common approach. A highly available coordination service (ZooKeeper, etcd, Consul) maintains the current leader's address. Clients query this service to discover the leader. When a failover happens, the new leader registers itself, and clients either watch for changes (via watches/leases) or re-query on the next error. Examples: Kafka uses ZooKeeper (historically) to track partition leaders; HBase uses ZooKeeper for its master address.

2. Redirect from any replica ("ask anyone")

Clients can connect to any node in the cluster. If that node isn't the leader, it either proxies the request to the leader or returns an error/redirect with the leader's address. The client then caches that address and uses it until it fails again. This is simple and very common. Examples: Redis Cluster returns MOVED errors pointing to the correct node; MongoDB's replica sets return "not primary" errors with the current primary's identity; etcd itself returns leader-redirect headers.

3. Client-side membership list with discovery protocol

The client is configured with a list of all (or several) replica addresses. On connection or after a failure, the client tries each one, asking "are you the leader?" or getting the leader's identity from whichever node responds. The client library handles this transparently. Examples: MongoDB drivers take a seed list and run the server discovery and monitoring (SDAM) protocol; Cassandra drivers connect to any seed node to learn the full topology; CockroachDB clients can connect to any node which then routes internally.

4. DNS-based discovery

A DNS record (often an SRV record or a simple A record) points to the current leader. On failover, the DNS record is updated. Clients use a short TTL so they pick up changes quickly. This is operationally simple but has a latency floor (TTL propagation). Examples: AWS Aurora exposes a "cluster endpoint" DNS name that always resolves to the current writer; Consul can expose service discovery via DNS.

5. Load balancer or virtual IP (VIP)

A load balancer or floating IP sits in front of the replicas and always routes to the current leader. The failover mechanism updates the LB's backend or moves the VIP to the new leader. Clients never change their target address. Examples: HAProxy in front of PostgreSQL with Patroni; AWS RDS/Aurora writer endpoints; keepalived VIPs for on-prem PostgreSQL clusters.

6. Gossip/membership protocol

Nodes use a protocol like SWIM or epidemic-style gossip to propagate cluster state (including who the leader is) to all members. Clients connected to any node get the current view. This avoids a central coordination service but introduces convergence delay. Examples: Consul's Serf layer; Cassandra's gossip protocol (though Cassandra doesn't have a single leader per se, the pattern applies to token/range ownership).

How these compare in practice:

The trade-off is essentially between simplicity, latency of discovery, and additional infrastructure. Most production systems combine two of these — for instance, a coordination service for ground truth (pattern 1) plus redirects at the node level (pattern 2) so that clients which happen to hit a follower still get routed correctly without a second round-trip to ZooKeeper. The DNS/VIP approaches (4 and 5) are popular in managed cloud databases because they completely hide failover from the client, at the cost of a brief unavailability window during DNS propagation or VIP reassignment.

Q: Give me a detailed breakdown of how that happens with use etcd (i.e. leader change is propagated to etcd, how the "writing parties" get updates from etcd and other relevant details)

A: The setup

Assume you have three database replicas (call them Node A, B, C) forming a replicated shard, and several writer clients that need to send writes to whichever node is currently the leader. etcd is running as a separate, highly available cluster (typically 3 or 5 etcd nodes).

Step 1: Leader registration with a lease

When Node A wins the leader election (however that election happens — more on that below), it needs to advertise itself in etcd. It does two things:

Create a lease. A lease is etcd's mechanism for ephemeral state. It's a time-to-live (TTL) handle — say 10 seconds. As long as the holder keeps renewing it, the lease stays alive. If the holder dies, renewals stop, and after the TTL expires etcd automatically deletes any keys attached to that lease.

# Node A asks etcd for a lease with a 10-second TTL
lease_id = etcd.lease_grant(ttl=10)

Write the leader key with the lease attached.

# Node A writes its address under a well-known key, attached to the lease
etcd.put(
    key   = "/services/mydb/shard-1/leader",
    value = '{"address": "node-a.internal:5432", "node_id": "A"}',
    lease = lease_id
)

Now etcd holds the key /services/mydb/shard-1/leader → Node A's address, and that key will vanish automatically if Node A stops refreshing the lease.

Keep-alive loop. Node A runs a background goroutine (or thread) that periodically sends keep-alive RPCs to etcd, resetting the TTL countdown:

# runs every ~3 seconds (well under the 10s TTL)
while i_am_leader:
    etcd.lease_keep_alive(lease_id)
    sleep(3)

This is sometimes called "heartbeating into etcd."

Step 2: How clients discover the leader

A writing client that just started up needs the current leader. The simplest path:

# Client does a point read
response = etcd.get("/services/mydb/shard-1/leader")
leader = json.parse(response.value)
# → {"address": "node-a.internal:5432", "node_id": "A"}

The client now opens a connection to node-a.internal:5432 and starts writing. It also records the etcd revision from this response — this becomes important next.

Step 3: Watching for changes

The client doesn't want to poll etcd in a loop. Instead it sets up a watch — a long-lived streaming RPC that says "tell me whenever this key changes, starting from revision N":

# This is a persistent streaming call, not a one-shot request
watch_stream = etcd.watch(
    key        = "/services/mydb/shard-1/leader",
    start_rev  = current_revision + 1
)

# In a background thread:
for event in watch_stream:
    if event.type == PUT:
        new_leader = json.parse(event.value)
        reconnect_to(new_leader.address)
    elif event.type == DELETE:
        # Leader key vanished (lease expired, leader died)
        # Wait briefly, then re-read the key for the new leader
        wait_and_rediscover()

The watch is the core mechanism. etcd maintains a multi-version history (MVCC), so even if the client is briefly disconnected, it can resume the watch from the last revision it saw without missing events. This is a key advantage over something like ZooKeeper's watches, which were one-shot in older versions.

Step 4: What happens when the leader dies

Suppose Node A crashes hard (process killed, machine loses power). Here's the sequence:

t=0s   Node A crashes. Keep-alive RPCs stop.
       Clients still have open connections to Node A.
       Those connections break (TCP reset or timeout).

t=0-3s etcd notices no keep-alive has arrived, but the
       TTL hasn't expired yet. The key still exists.

t=10s  The lease TTL expires. etcd DELETES the key
       /services/mydb/shard-1/leader automatically.

       All watchers instantly receive a DELETE event.

The gap between the crash and the key deletion (up to the full TTL) is the detection latency. This is tunable — shorter TTL means faster detection but more keep-alive traffic and higher risk of false positives during network blips.

Step 5: New leader election

Meanwhile, the followers (Node B and Node C) need to decide which of them becomes the new leader. There are two common patterns:

Pattern A: External election using etcd itself. The replicas use etcd as the election mechanism. Each candidate tries to write the leader key using a transaction with a compare-and-swap — essentially "write this key only if it doesn't already exist":

# Node B tries to claim leadership
new_lease = etcd.lease_grant(ttl=10)

success = etcd.transaction(
    compare = [key_not_exists("/services/mydb/shard-1/leader")],
    success = [put("/services/mydb/shard-1/leader",
                   '{"address": "node-b.internal:5432", "node_id": "B"}',
                   lease=new_lease)],
    failure = []  # someone else won, do nothing
)

Because etcd transactions are linearizable, exactly one node wins. The others see success=false and remain followers. This is exactly how projects like Patroni (for PostgreSQL) work.

Pattern B: Internal election, then registration. The database cluster runs its own election protocol (Raft internally, or some custom protocol). Once a winner is determined — say Node C — the new leader writes itself into etcd the same way Node A originally did. etcd is just the registry, not the election mechanism.

Either way, the result is the same: a new PUT event on the leader key with the new leader's address.

Step 6: Clients react to the change

Back in the client's watch loop:

# The watch stream delivers a PUT event
event: PUT  key=/services/mydb/shard-1/leader
        value={"address": "node-b.internal:5432", "node_id": "B"}

# Client reacts:
1. Close connection to old leader (node-a, already broken anyway)
2. Open connection to node-b.internal:5432
3. Resume writing

The total downtime from the client's perspective is roughly:

lease TTL (detection)    ~10s   (tunable)
+ election time          ~1-3s  (fast if using etcd txn)
+ client reconnection    ~0.1s
                         ------
                         ~11-13s
Step 7: Handling edge cases

Stale leader (split brain). Node A might not be dead — it might just be network-partitioned from etcd but still accepting writes. This is dangerous. The standard safeguard is called fencing: Node A itself watches its own lease. If it can't renew the lease, it must step down and stop accepting writes, even though it's still running:

# On Node A (the old leader):
try:
    etcd.lease_keep_alive(lease_id)
except FailedToRenew:
    # I can't reach etcd — I must assume I've been replaced
    self.step_down()
    self.reject_all_writes()

This is critical. Without it, you can have two nodes both thinking they're the leader (split brain), which leads to data corruption.

Client connected during the gap. Between the old leader dying and the new leader registering, the key doesn't exist. A new client arriving during this window would get an empty response from etcd.get(...). The client should either retry with backoff or set up a watch and wait for the key to appear.

etcd cluster itself goes down. If etcd is unavailable, existing leaders keep running (they just can't renew their lease, so they'll eventually step down). This is why etcd itself is run as a highly available cluster (3 or 5 nodes with Raft consensus). The design principle: etcd being down doesn't immediately kill your service, but it disables leader failover until etcd recovers.

The full picture
                    ┌──────────────┐
                    │  etcd cluster │
                    │  (3-5 nodes)  │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
         lease+put      watch        watch
         keep-alive     (stream)     (stream)
              │            │            │
         ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
         │ Leader  │  │Writer   │  │Writer   │
         │ Node B  │  │Client 1 │  │Client 2 │
         └─────────┘  └─────────┘  └─────────┘
              ▲
         write RPCs
              │
         ┌────┴────┐  ┌─────────┐
         │Writer   │  │Follower │  (Followers also watch
         │Client 3 │  │ Node C  │   etcd to know who the
         └─────────┘  └─────────┘   leader is)

Real-world systems using exactly this pattern include Patroni (PostgreSQL HA), Vitess (MySQL sharding — uses etcd or ZooKeeper for topology), and Kubernetes itself (the API server is effectively the leader for cluster state, registered in etcd). The details vary — some use etcd's built-in election recipe, some use the raw transaction primitives — but the lease + watch + fencing trifecta is the core of all of them.

Q: Above you wrote "Stale leader (split brain). Node A might not be dead — it might just be network-partitioned from etcd but still accepting writes. This is dangerous. The standard safeguard is called fencing: Node A itself watches its own lease. If it can't renew the lease, it must step down and stop accepting writes, even though it's still running:

# On Node A (the old leader):
try:
    etcd.lease_keep_alive(lease_id)
except FailedToRenew:
    # I can't reach etcd — I must assume I've been replaced
    self.step_down()
    self.reject_all_writes()
This is critical. Without it, you can have two nodes both thinking they're the leader (split brain), which leads to data corruption."The mechanism you have described here still won't give you 100% guarantee from the split brain. Consider this: assume the Node A was just able to renew its lease and then hit an unusally long GC pause (say, 20 sec). So no heartbeats to etcd were issued. Etcd considered it dead and the new leader has been eleceted. In the meantime the old leader still can proceed wth whatever operation (write) it was in hte middle of right when it hit the GC pause. So the "fencing" technique you have described doesn't work reliably.

A: You're absolutely right, and this is a really important subtlety. What I described is necessary but not sufficient. Let me walk through why it breaks and what actually solves it.

Why the lease-check approach fails

The fundamental issue is that checking a lease and acting on the result is a check-then-act pattern, and any unbounded pause between the two steps breaks it:

Node A (old leader)              etcd                  Node B

1. renew lease → OK
2. begin processing write W
3. ──── GC pause starts ────
                              4. lease TTL expires
                              5. key deleted
                                                    6. wins election
                                                    7. writes to followers
                                                    8. accepts client writes
   ──── GC pause ends ────
9. complete write W
   (still thinks it's leader,
    lease check passed at step 1)

At step 9, Node A has no idea anything happened. The lease renewal succeeded before the pause. The local "am I still leader?" check is based on state from before the pause. The process was literally frozen — it couldn't observe time passing or detect that anything changed. This isn't unique to GC pauses either; the same thing can happen from a VM live-migration pause, a swapped-out process, a long fsync, or even just OS scheduling under extreme load.

This is the point Martin Kleppmann made in his well-known critique of Redlock, and it applies equally to any lease-based scheme: you cannot rely on a local process to police itself, because the very mechanism that would detect the problem (running code) is the thing that's paused.

The real solution: fencing tokens

The fix is to move enforcement away from the leader and onto the receivers of the writes — the followers or the storage layer. The mechanism is called a fencing token (also known as an epoch number, generation number, or term number).

The idea is straightforward. Every time a new leader is elected, it gets a monotonically increasing number. This number accompanies every write. The receivers reject any write bearing an old number.

Here's how it works concretely with etcd:

At election time, the new leader reads the etcd key's revision (or create_revision of the lease, or a separate monotonic counter incremented in the same transaction). This revision is globally ordered and guaranteed to increase with each leader change:

# Node B wins the election
txn_response = etcd.transaction(
    compare = [key_not_exists("/services/mydb/shard-1/leader")],
    success = [put("/services/mydb/shard-1/leader", node_b_info, lease=...)],
    failure = []
)
# The revision from this transaction is the fencing token
fencing_token = txn_response.header.revision
# e.g., fencing_token = 42

The leader includes the token with every write to followers:

# Every replication message carries the token
replicate_to_followers(
    operation = write_W,
    fencing_token = 42
)

The followers enforce the fence:

# On each follower:
def handle_replication(request):
    if request.fencing_token < self.highest_seen_token:
        reject(request)  # stale leader, ignore
        return
    self.highest_seen_token = request.fencing_token
    apply(request.operation)

Now replay the failure scenario:

Node A (token=41)                etcd              Node B (token=42)

1. renew lease → OK
2. begin write W (token=41)
3. ──── GC pause ────
                             4. lease expires
                                                  5. wins election, token=42
                                                  6. writes to followers (token=42)
                                                     followers update:
                                                     highest_seen = 42
   ──── GC pause ends ────
7. sends write W to follower
   with token=41
                    ↓
   follower: 41 < 42 → REJECTED

The old leader's write is rejected regardless of what it believes about its own status. The enforcement happens at the point that matters — the storage layer — not inside the potentially-paused leader process.

Where this pattern appears in real systems

Raft's term number is exactly a fencing token. Every message in Raft carries a term. If a follower receives an AppendEntries RPC with a term lower than what it has already seen, it rejects it. This is why Raft-based systems (etcd itself, CockroachDB, TiKV) are inherently protected against this scenario — the protocol has fencing built into its core.

ZooKeeper's zxid serves a similar role. Chubby (Google's lock service, the predecessor to ZooKeeper) called these sequencers in the original paper.

Patroni (PostgreSQL HA with etcd) handles this at the PostgreSQL level. When a new leader is promoted, the timeline ID in PostgreSQL's WAL increments. A stale leader writing to an old timeline won't have its WAL accepted by followers streaming from the new timeline.

The broader principle

The lesson generalizes beyond leader election: any time safety depends on one actor's exclusive access to a resource, enforcement must live at the resource, not at the actor. A lock holder can be paused, partitioned, or confused. The storage, the follower, the file system — whatever receives the writes — is the only party that can reliably say "your token is stale, go away." This is essentially what Kleppmann argues in chapter 8 of Designing Data-Intensive Applications, and it's one of those ideas that seems obvious in retrospect but is surprisingly easy to get wrong when designing a system.

Q: Above you wrote: "The fix is to move enforcement away from the leader and onto the receivers of the writes — the followers or the storage layer. The mechanism is called a fencing token (also known as an epoch number, generation number, or term number).

The idea is straightforward. Every time a new leader is elected, it gets a monotonically increasing number. This number accompanies every write. The receivers reject any write bearing an old number."

But this still doesn't solve this case: if the old leader with the old lease and old sequence number (Aka fence token) made its write before the new leader gets to do the same, the target store (the resource) would allow it as it wouldn't have yet seen that the new leader has been elected.. So the split brain still would exist.

A: You're right again, and this is an important further refinement. Let me trace through the exact scenario to make sure we're on the same page, then explain what actually closes the gap.

The scenario that breaks fencing tokens alone
Node A (token=41)              etcd              Node B

1. renew lease → OK
2. ── GC pause starts ──
                            3. lease expires
                                                4. wins election, gets token=42
                                                   (has NOT yet written to
                                                    followers)
   ── GC pause ends ──
5. sends write W (token=41)
   to follower
   
   Follower thinks:
   "highest token I've seen = 41,
    incoming token = 41,
    41 ≥ 41 → ACCEPT"          ← stale write goes through

                                                6. sends first write (token=42)
                                                   Follower accepts this too

The follower accepted a write from a leader that was already deposed — it just didn't know yet. The fencing token helps only when the new leader's writes arrive first, but there's no guarantee of that ordering. You're correct that this is a real gap, not a theoretical one.

What actually solves it

The fundamental insight is that no mechanism local to a single node can prevent this. And no mechanism at the followers alone can prevent it either, if they haven't been told about the new epoch. You need a protocol-level guarantee that before the new leader can accept any writes, every node that could accept a write from the old leader has already been notified of the new epoch. There are three ways real systems accomplish this:

1. Quorum overlap (the Raft/Paxos answer)

This is the most rigorous solution and what consensus protocols actually rely on. The rule: a write is not considered committed until a majority of nodes acknowledge it. Leader election also requires a majority.

By pigeonhole, those two majorities must overlap in at least one node. That overlap node is the key — it acts as an implicit barrier.

Here's why it works. Say you have nodes A, B, C (majority = 2):

Node A (old leader, token=41)    Node B    Node C

Scenario: A is paused, B wins election

B wins election by getting votes from B and C (majority).
Before B starts, it replicates a "new term" marker to
B and C. Both now know: term = 42.

A wakes up, sends write W (token=41) to followers:
  → sends to B: B has seen term 42, rejects (41 < 42)
  → sends to C: C has seen term 42, rejects (41 < 42)
  A cannot get a majority ack → write is NOT committed

The critical detail: the election itself serves as the barrier. To win, the new leader must contact a majority. That majority now knows about the new term. The old leader also needs a majority to commit — but every possible majority overlaps with the one that elected the new leader. So the old leader will always hit at least one node that rejects its stale writes.

This is exactly how Raft works. The AppendEntries RPC carries the term number, and any node that has seen a higher term rejects messages from older terms. The new leader also doesn't accept client writes until it has committed a no-op entry in the new term to establish its authority across the majority.

2. Explicit barrier protocol (for systems not using Raft internally)

If your replicated database doesn't use a consensus protocol internally and relies on etcd purely for coordination, you need to explicitly do what Raft does implicitly. Before the new leader starts accepting writes, it must:

# Node B, immediately after winning the election:

1. Send "BEGIN EPOCH 42" to ALL followers
2. Wait for acknowledgment from ALL (or a majority of) followers
3. Only then start accepting client writes

# On each follower:
def handle_begin_epoch(new_token):
    self.highest_seen_token = new_token
    # From this point, writes with token < 42 are rejected
    ack(new_token)

This closes the window because by the time Node B starts writing, every follower already knows about epoch 42 and will reject anything from epoch 41. The downside is latency — the new leader can't serve writes until the barrier completes.

3. Storage-level fencing (I/O fencing)

In shared-storage architectures (common in traditional enterprise HA), you fence at the infrastructure layer rather than the application layer. The idea: instead of asking followers to reject stale writes, you revoke the old leader's ability to write at all.

Mechanisms include SCSI-3 persistent reservations (the new leader registers a new reservation key and preempts the old one — the storage array itself refuses I/O from the old reservation), VM-level fencing (the cluster manager powers off or isolates the old leader's VM before promoting the new one — this is what STONITH, "Shoot The Other Node In The Head," does in Pacemaker clusters), and network-level fencing (reconfigure the network to block the old leader's traffic to the storage).

This is a blunt instrument, but it gives a hard guarantee: the old leader physically cannot write, regardless of GC pauses, code bugs, or any other software-level issue.

The hierarchy of guarantees

So to put this whole thread together:

Lease-based self-check (what I initially described) prevents the common case — a leader that notices it lost its lease and steps down voluntarily. But it fails under pauses.

Fencing tokens handle the case where the old leader's write arrives after the new leader's. But they fail if the stale write arrives first.

Quorum overlap / barrier protocols close the remaining gap by ensuring that by the time any new write is possible, a sufficient number of nodes already know about the new epoch to block stale writes from being committed.

Each layer catches what the previous layer misses. Production systems that take correctness seriously (as opposed to "mostly correct, rarely fails") use quorum-based approaches or explicit barriers. This is why you'll find that every system that truly claims linearizable writes — Raft-based systems, Multi-Paxos systems, ZAB (ZooKeeper's protocol) — has quorum overlap baked into the core protocol, not as an afterthought.

Q: But, regardless of the mechanism, technically, the "leader bifurcation" still _physically_ takes_ place - not matter how we choose to fence/cut it off, the old leader still may exist (as a live process) that thinks that it's still the leader. What we do instead is try to keep it from doing side-effects (e.g. doing writes).
And even in that case we can't absolutely guarantee that a node deemed an old leader (e.g. sue to the heartbeat timeout) won't be able to make a single write right after, say, etcd considers it's dead. The reason: signle propagation latency. As the "cut off signal" - be it a new version, or an update to the SCSI-3 interface - takes time still. And if that signal was issues even a momnet later that the old leader was able to initiate its write, the write will go through.

Given that, I think it's more correst to frame those mechanisms as not "split-brain prevention", but rather "cancelling out inconsistencies due to split brain" - as even given that write from the old leader did go through, it doesn't mean anything is corrupted (from the app logic/invariants perspective) - after all as lomng as the version number it carries greater or equal to the last seen by the given storage. Or did  miss something?

A: This is a really sharp observation, and you're mostly right — but I think there's one important distinction worth drawing that changes the framing slightly.

Where you're exactly right

You cannot prevent the mental state of split brain. In an asynchronous system, there is always some window where two processes both believe they are the leader. This is a direct consequence of the impossibility of instantaneous state propagation. No protocol, no matter how clever, can make one process instantly aware that another process has decided something. That's just physics.

And you're right that every "cut-off signal" has propagation latency. Whether it's a new term number, a SCSI reservation preempt, or a network rule — there is always a nonzero interval between "the system decides the old leader is deposed" and "the old leader is physically unable to act." During that interval, the old leader can initiate writes.

Where I'd refine the framing

The distinction I'd push on is between a write being physically sent and a write being committed — meaning durably accepted as part of the system's state and potentially visible to clients.

In a quorum-based protocol, those are very different things. Consider Raft with nodes A, B, C (majority = 2):

Node A (old leader, term 41):

  1. A sends AppendEntries(term=41, write W) to B and C
  2. Suppose it reaches B before B has heard about the election

  B accepts it into its local log. The physical write happened.

  But A needs acks from 2 nodes (itself + one follower) to tell the
  client "write committed." Meanwhile:

  3. C already voted for Node B's election (saw term 42)
     C rejects A's message (41 < 42)

  A has: itself + B = 2? Maybe. But...

  4. Node B (new leader, term 42) sends its first AppendEntries to B
     B sees term 42 > 41, updates its term, and when the new leader
     replicates its log, W gets overwritten in B's log because it
     was never committed — it's in a suffix that the new leader
     doesn't have.

The crucial property: write W was physically received by a follower, but it was never committed. It sits in an uncommitted log suffix that the new leader will overwrite. No client was ever told "write succeeded." From the perspective of the system's linearizable history, write W never happened.

So in quorum-based systems I'd argue it's not quite "cancelling out inconsistencies due to split brain." It's more precisely: the system's definition of "committed" is designed so that a deposed leader structurally cannot meet the commit criteria, even if it can still physically send messages. The write doesn't go through and then get cancelled — it goes through at a physical layer that doesn't constitute commitment. The distinction between "received by a node" and "committed by the system" is doing all the heavy lifting.

Where your framing fits perfectly

For systems that don't use quorum-based replication — say, a single-follower synchronous replication setup, or a primary-backup system that relies on external coordination — your framing is exactly right. If the old leader writes to the single follower before the fencing signal arrives, that write genuinely went through. It's in the follower's durable state. And the system's only option is to ensure that this doesn't violate application invariants. That's where your "cancelling out" framing is spot on — the new leader's writes must either be compatible with or supersede the stale write.

So what's the general principle?

I think the cleanest way to put it is your core insight plus the commit distinction:

You can never prevent a deposed leader from believing it's the leader or from attempting writes. What you can control is the definition of "committed." A well-designed protocol defines commitment such that a deposed leader's writes structurally cannot meet the commitment threshold, regardless of timing. The "split brain" physically exists in the sense that two processes hold the leader belief simultaneously, but it never manifests in the committed state of the system.

In quorum systems, the quorum intersection property is what makes this structural. In non-quorum systems, you're right that it degrades to "best-effort damage control" — fencing tokens, epoch barriers, I/O fencing — and the guarantees are weaker, closer to your "cancelling out inconsistencies" framing.

The FLP impossibility result lurks behind all of this: in an asynchronous system you cannot simultaneously guarantee agreement, termination, and fault tolerance. Every real system makes a trade-off, and the "split brain window" you're identifying is one manifestation of that fundamental limit.

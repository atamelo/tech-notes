# Leader Election, etcd & Fencing

*A conversation with ChatGPT: how writers find a new leader after failover, how etcd's leader election works, and why fencing can only be as strong as the resource enforcing it.*

## Contents

1. [How writers learn the new leader address](#q1--how-writers-learn-the-new-leader-address)
2. [Isn't a stable endpoint just kicking the can?](#q2--isnt-a-stable-endpoint-just-kicking-the-can)
3. [The stable endpoint is itself the writer](#q3--the-stable-endpoint-is-itself-the-writer)
4. [Can etcd help?](#q4--can-etcd-help)
5. [How etcd leader election works](#q5--how-etcd-leader-election-works)
6. [GC pauses and lease-based fencing](#q6--gc-pauses-and-lease-based-fencing)
7. [Fencing tokens and the epoch-install barrier](#q7--fencing-tokens-and-the-epoch-install-barrier)

---

## Q1 — How writers learn the new leader address

> [!NOTE]
> **Question:** Suppose I have a distributed system with replication (say, factor 3) with a leader shard and two followers. All the operations go through the leader. So the writing party needs to know leader's address. In case the leader changes (failover), its address changes too. What are the most common ways (used in practice) to let the writing parties know of this new address?

In practice, systems usually solve this with one of a few patterns. A key observation is that clients often don't need to learn the leader's address proactively; they can discover it lazily when they try to write.

| Pattern                               | How it works                                                                  | Typical use                              |
| ------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------- |
| **Stable proxy / load balancer**      | Client always talks to one stable address; proxy routes to current leader     | Databases, managed services              |
| **Client-side discovery**             | Client asks a registry / cluster metadata service who the leader is           | etcd, ZooKeeper-style systems            |
| **Redirect / `NOT_LEADER` response**  | Client contacts any replica; replica replies “leader is X”; client caches X   | Raft-based systems, Kafka-like protocols |
| **DNS / VIP movement**                | A stable hostname or virtual IP is repointed to the new leader                | Simpler deployments                      |
| **Client maintains cluster metadata** | Client knows several nodes and refreshes topology/leader metadata when needed | Kafka, Cassandra-like client libraries   |

The most common scalable design is actually a combination of **bootstrap + redirect + caching**.

For example, suppose replicas are:

```text
A
B
C
```

and initially:

```text
A = leader
B = follower
C = follower
```

The client is configured with something like:

```text
bootstrap_nodes = [A, B, C]
```

It doesn't necessarily know which one is leader.

It sends a write to B:

```text
Client -> B

B -> Client:
    NOT_LEADER
    leader = A
    leader_epoch = 42
```

The client remembers:

```text
leader = A
epoch = 42
```

and writes directly:

```text
Client -> A
```

Later A fails and C becomes leader.

The next request to A fails, or A responds that it is no longer leader. The client contacts another known node:

```text
Client -> B

B -> Client:
    NOT_LEADER
    leader = C
    leader_epoch = 43
```

Then:

```text
Client -> C
```

This has a useful property: you don't need some external mechanism pushing leader changes to every writer. Clients discover changes as necessary.

### 1. Stable frontend / proxy

Another very common architecture is:

```text
             +----------+
Clients ---> | Proxy/LB |
             +----+-----+
                  |
                  v
                Leader
```

The client always connects to:

```text
db.example.com:1234
```

The proxy knows:

```text
current leader = C
```

If leadership changes:

```text
A -> C
```

only the proxy's routing changes. Clients don't care.

This greatly simplifies clients, but introduces another component into the write path. The proxy itself therefore usually needs redundancy.

Managed database products often expose something resembling this model: a stable endpoint remains constant while the backend role changes.

### 2. Coordination/discovery service

You can store the current leader in something strongly consistent such as:

```text
/cluster/foo/leader -> 10.1.2.7:9000
```

in:

- etcd
- ZooKeeper
- Consul
- another consensus-backed metadata service

Clients can either query it when necessary or watch the key for changes:

```text
watch /cluster/foo/leader

A -> C
```

and receive an update.

This works, but there's a subtle architectural issue: if you're already implementing a replicated consensus system, introducing another consensus system just to discover your consensus system's leader can be unnecessary complexity.

Consequently, many systems instead let the replicas themselves provide this metadata.

### 3. DNS or virtual IP

You can have:

```text
leader.myservice.internal
```

point to the current leader.

On failover:

```text
leader.myservice.internal
A record: A -> C
```

or move a virtual IP between machines.

This is operationally simple, but DNS in particular isn't ideal for very rapid leader changes because of caches, TTLs, connection pooling, and resolver behavior.

VIP failover can be much faster, but is generally tied more closely to the networking environment.

### An important detail: leader address usually doesn't really change

A particularly useful design distinction is:

```text
node A = 10.0.0.1
node B = 10.0.0.2
node C = 10.0.0.3
```

Those addresses remain stable.

What changes is only:

```text
leader_id = A
```

becoming:

```text
leader_id = C
```

So instead of thinking:

> The address of the service changed.

it is often better to think:

> The mapping (shard -> leader node) changed.

For a sharded system you might have:

```text
Shard 1 -> node A
Shard 2 -> node F
Shard 3 -> node J
```

and clients maintain this metadata.

That's roughly the architecture used by many high-performance distributed systems because it avoids routing every request through an extra proxy.

### What I would consider the canonical design

For a high-throughput distributed storage system, a very common approach is:

```text
Client
  |
  | bootstrap / metadata request
  v
Any cluster node
  |
  | leader = B, epoch = 27
  v
Client caches:
  Shard 17 -> B
```

Then:

```text
Client ----------> B
                (leader)
```

After a failover:

```text
Client ----------> B
                   |
                   +--> NOT_LEADER
                        leader=C
                        epoch=28

Client ----------> C
```

Clients maintain a **leader cache**, not a permanently authoritative mapping.

And the epoch / term is important. If the old leader comes back or stale metadata propagates, everyone can distinguish:

```text
leader=A, term=27   # stale
leader=C, term=28   # current
```

This fits naturally with Raft/Paxos-style consensus because they already maintain monotonically increasing terms/epochs.

One additional production concern is retries. A failover can create the sequence:

```text
client sends write
leader commits write
leader dies
client never receives response
client retries against new leader
```

So writes should generally carry an idempotency/request ID such as:

```text
(client_id, request_sequence_number)
```

allowing the new leader to recognize duplicate requests.

So the practical answer is: either hide leadership behind a stable endpoint, or let clients discover the leader from any replica and cache that information. For high-performance distributed storage, the latter—bootstrap nodes + metadata/redirect + cached leader mapping + epoch/term—is probably the most characteristic architecture.

---

## Q2 — Isn't a stable endpoint just kicking the can?

> [!NOTE]
> **Question:** Above you wrote: "either hide leadership behind a stable endpoint" but that "stable endpoint" in its turn will need to know what the leader's address is. So it's just like kicking the can down the road.

Exactly. A “stable endpoint” does not eliminate leader discovery; it centralizes it.

The real difference is who has to track leadership changes. Without a stable endpoint, potentially thousands of writers need to maintain `shard -> leader` mappings. With one, only a small routing layer needs to know. That router can learn the leader in essentially the same ways clients would: query any replica, receive `NOT_LEADER` / `leader=X`, watch consensus-backed metadata, or in some systems participate directly in the cluster/control plane.

So you can think of it as:

```text
without proxy:
    10,000 clients × leader-discovery logic

with proxy:
    10,000 clients -> stable address
                            |
                            v
                     3 proxies × leader-discovery logic
```

You are indeed “kicking the can down the road,” but deliberately: you move volatile topology knowledge from every application into a small infrastructure component.

There are only a few fundamental possibilities. Somebody somewhere must know or discover the current leader:

1. Clients discover it.
2. A proxy/router discovers it.
3. A naming/network layer is updated when leadership changes.
4. Any node accepts the request and internally forwards it to the leader.

Even case 4 is just another form of indirection:

```text
Client -> follower B -> leader C
```

Now B needs to know that C is leader.

So there is no magic mechanism that avoids leader-location knowledge entirely. The design choice is where you place that knowledge and how many components must maintain it.

In consensus-based systems, a particularly clean arrangement is that the replicas themselves are the authoritative source. A client/router only needs a few stable bootstrap addresses. It asks any reachable node, “who is leader for shard X?” and then caches the result until it becomes stale.

That gives you a hierarchy like:

```text
stable knowledge:
    addresses of some cluster members

dynamic knowledge:
    shard 42 -> leader node C, term 91
```

The important trick is that the bootstrap information changes much less frequently than leadership. You don't need a permanently correct leader address; you only need some reachable cluster member from which the current leader can be rediscovered.

That's why systems often configure clients with multiple bootstrap nodes rather than a leader address. The bootstrap set is not authoritative and need not contain the leader—it just gives you an entry point into the cluster.

---

## Q3 — The stable endpoint is itself the writer

> [!NOTE]
> **Question:** Yeah, but my initial question specifically wasn't about "clients", but rather a "writing party" - in this case being the "stable endpoint".

Right — in your terminology, the “stable endpoint” is itself a writing party, so it has exactly the same problem your original question asked about.

For that component, the common production mechanisms are basically:

1. **Ask the replica group directly.** The endpoint keeps addresses of the 3 replicas, sends to one, and gets either success or `NOT_LEADER, leader=C`. It caches C until that becomes stale.
2. **Read/watch authoritative metadata.** The consensus group, or a separate control-plane service such as etcd/ZooKeeper/Consul, maintains something like `shard-17 -> node C, term 42`. The writer watches that record and updates immediately on failover.
3. **Have the election mechanism publish the new leader.** When C wins leadership, it announces itself to registered routers/writers or updates shared metadata. In practice this is usually implemented as the previous “watch metadata” pattern rather than bespoke push notifications.
4. **Send to any replica and let replicas forward internally.** Then the stable endpoint does not need to know the leader at all; whichever replica it reaches does. The leader-location problem has moved into the replica group.

For a router/stable endpoint specifically, the most common shape is often:

```text
Router has:
    replica addresses = [A, B, C]
    cached leader = A
    cached term = 91

write -> A
```

After failover:

```text
A unavailable
or
A -> NOT_LEADER, leader=C, term=92

Router updates cache:
    leader = C
    term = 92

write -> C
```

Or, with a metadata service:

```text
Consensus group elects C
        |
        v
metadata:
    shard-17 -> C, term=92
        |
        v
router watch fires
        |
        v
router now routes writes to C
```

The key distinction I should have made earlier is that a stable endpoint solves **address stability for its callers**, not **leader discovery for itself**. For the stable endpoint, the canonical choices are still either discover on demand from the replicas or subscribe to authoritative leader metadata.

---

## Q4 — Can etcd help?

> [!NOTE]
> **Question:** Can systems like etcd help here?

Yes. etcd is very well suited to exactly this coordination/discovery problem. It gives you a strongly consistent key-value store plus watches, leases, and election primitives.

A straightforward design is:

```text
/cluster/shard-17/leader -> {
    address: "10.0.0.3:9000",
    term: 42
}
```

Your writing party—the stable endpoint/router—does this:

```text
1. Read /cluster/shard-17/leader
2. Cache C = 10.0.0.3:9000
3. Watch that key
4. Route writes to C
```

When leadership changes:

```text
A loses leadership
C becomes leader
```

C (or your failover controller) updates etcd:

```text
/cluster/shard-17/leader:
    A, term=41
        ->
    C, term=42
```

The router's watch fires and it updates its routing table:

```text
shard-17 -> C
```

etcd watches are specifically designed to notify clients of subsequent key changes.

The more interesting question is **who is allowed to write that key**. You don't want two nodes independently announcing:

```text
A says: I'm leader
C says: I'm leader
```

because this recreates the distributed coordination problem you're trying to solve.

One robust arrangement is to use an etcd lease/election. A node acquires leadership through etcd and publishes its address as the election value. etcd's election API supports acquiring leadership, observing current leadership, and automatically losing leadership when the associated lease expires.

Conceptually:

```text
           etcd cluster
        /       |       \
      E1        E2        E3
                ^
                |
       leader coordination
                |
       +--------+--------+
       |        |        |
       A        B        C
                |
             elected
             leader
                |
        publishes address
                |
                v
           /leader = B
                |
              watch
                |
                v
              Router
```

That means etcd isn't merely answering:

> "Where is the leader?"

It can actually be the authority deciding:

> "Who currently owns the right to act as leader?"

Those are two different uses.

### If your system already has Raft/Paxos

This is where I'd be cautious.

Suppose your replica group itself already runs Raft:

```text
A ----- B ----- C
      Raft
```

Raft has already decided that:

```text
leader = C
term = 42
```

If you additionally use etcd to determine leadership:

```text
A ----- B ----- C
  Raft election

        +

E1 ---- E2 ---- E3
  etcd/Raft election
```

you now have two consensus systems whose view of leadership must agree.

That's usually not desirable.

A better arrangement in that case is:

```text
Your Raft group
      |
      | authoritative election
      v
leader = C
      |
      | publish metadata
      v
    etcd
      |
      | watch
      v
   routers
```

Here etcd is only the **distribution mechanism** for leader metadata, not the authority that determines your shard's leader.

And even then you have to deal with a timing window:

```text
t0: C becomes leader, term 42
t1: etcd still says A, term 41
t2: C updates etcd
```

So the writer should still tolerate stale metadata:

```text
Router -> A

A:
    NOT_LEADER
    leader=C
    term=42
```

In other words, I would treat etcd as an efficient discovery/cache-invalidation mechanism, not as proof that a particular node is still leader.

The authoritative truth should come from whichever consensus protocol actually owns the shard.

### Where etcd becomes especially attractive

Imagine hundreds or thousands of shards:

```text
/shards/0001/leader -> node-17
/shards/0002/leader -> node-93
/shards/0003/leader -> node-17
...
/shards/8472/leader -> node-51
```

Routers can watch a prefix:

```text
/shards/
```

and maintain an in-memory routing table:

```text
shard -> leader
```

Then leader changes propagate through watches instead of every router continually polling every replica group. etcd supports watching keys and prefixes for this sort of change stream.

So for the architecture you were originally asking about, a very plausible production design is:

```text
                 consensus groups
             /           |           \
        shard1        shard2        shard3
           |             |             |
           +-------------+-------------+
                         |
                         | leader changes
                         v
                     +------+
                     | etcd |
                     +------+
                         |
                         | watches
                         v
                +--------+--------+
                |                 |
            Router 1          Router 2
                |                 |
                +-----writers-----+
```

The important caveat is:

> etcd solves **dissemination** of the leader location very nicely. It doesn't make the underlying leader-election/fencing problem disappear.

If the replica group already elects its leader, I'd generally let that protocol remain authoritative and use etcd to distribute the resulting shard → leader + term/epoch mapping.

---

## Q5 — How etcd leader election works

> [!NOTE]
> **Question:** Let's talk a bit more about how etcd provides leader election. AFAIU, it's based on leases, right?

Yes—with an important distinction.

If you mean etcd's client-facing **Election primitive**, then yes: it is built on leases + ephemeral keys + etcd's globally ordered revisions. If you mean how the **etcd cluster itself** elects its internal Raft leader, that's different: that is ordinary Raft leader election using terms, heartbeats, votes, and quorum—not etcd leases.

For the client-facing election, the mechanism is roughly this. Suppose A, B, and C all want to become application leader for `/my-service`.

```text
A ─┐
B ─┼──> etcd
C ─┘
```

Each participant first creates its own lease and keeps it alive. In the current Go concurrency implementation, an etcd `Session` grants a lease and starts a keepalive stream; the default session TTL is 60 seconds unless changed.

Then each participant calls `Campaign`. That creates a key under the election prefix and attaches that key to its lease:

```text
/my-service/<A-lease-id> = "A"   lease=A-lease
/my-service/<B-lease-id> = "B"   lease=B-lease
/my-service/<C-lease-id> = "C"   lease=C-lease
```

Each creation gets an etcd creation revision:

```text
A key: createRevision = 100
B key: createRevision = 105
C key: createRevision = 109
```

The candidate with the **oldest surviving creation revision** is leader. The others wait for every candidate key older than theirs to disappear. That's literally what the current etcd client implementation does: `Campaign` creates a key attached to the session lease, records its creation revision, and then waits for lower-revision election keys to be deleted.

So initially:

```text
100  A   <-- leader
105  B   <-- waiting
109  C   <-- waiting
```

Now suppose A crashes.

Its lease stops receiving keepalives:

```text
A process dies
      |
      X keepalive
      |
    etcd
```

After the lease expires, etcd automatically removes keys attached to that lease. Thus:

```text
/my-service/<A-lease-id>
```

disappears. An etcd lease-attached key is deleted when its lease expires.

Now the remaining order is:

```text
105  B   <-- leader
109  C   <-- waiting
```

B's waiting watch notices that all preceding candidates are gone, and `Campaign()` returns successfully to B. B now knows it has acquired leadership. The Election API explicitly defines expiration/revocation of the leader's lease as transferring leadership to the next campaigner.

So a useful mental model is:

```text
                  etcd
                   |
       +-----------+-----------+
       |           |           |
     lease A     lease B     lease C
       |           |           |
      key A       key B       key C
     rev=100     rev=105     rev=109
       ^
       |
  smallest rev
   = leader
```

The lease does not, by itself, determine who wins. There isn't one magical "leader lease" that everybody races to acquire.

Instead:

> **The lease provides liveness. The creation revision provides ordering. The ephemeral election key represents candidacy/ownership.**

That distinction matters quite a bit.

For example, if A and B campaign almost simultaneously, etcd's serialized KV operations give their keys an unambiguous order:

```text
A: revision 4001
B: revision 4002
```

A wins. B waits for A's key to disappear.

If A gracefully resigns, etcd deletes A's election key immediately rather than waiting for the lease TTL. If A crashes or becomes disconnected long enough, its lease expiration eventually causes the same deletion. `Resign` in the implementation conditionally deletes the leader's key.

There is one subtle but very important issue for the distributed system you're describing: **lease-based election does not automatically fence the old leader from external resources.**

Imagine:

```text
   A is leader
        |
network partition
        |
        X  etcd

A still running
```

A may temporarily continue to believe it is leader. Meanwhile its lease expires in etcd:

```text
etcd:
    A's lease expires
    A's key disappears
    B becomes leader
```

You could transiently have:

```text
A: "I think I'm leader"
B: "etcd says I'm leader"
```

The Election API provides a leader key and its creation revision specifically so ownership can be checked transactionally against etcd.

And that brings us directly back to your original replicated-writer problem: leases solve failure detection/liveness, but **fencing** is the deeper correctness problem.

If you'd like, the interesting next step is to look at exactly what happens during a network partition and why a lease alone isn't sufficient to guarantee that two machines can never concurrently perform writes.

---

## Q6 — GC pauses and lease-based fencing

> [!NOTE]
> **Question:** The mechanism you have described with fences still won't give you 100% guarantee from the split brain. Consider this: assume the Node A was just able to renew its lease and then hit an unusually long GC pause (say, 20 sec). So no heartbeats to etcd were issued. etcd considered it dead and the Node B has been promoted to the leader. In the meantime the old leader still can proceed with whatever operation (write) it was in the middle of right when it hit the GC pause. So the "fencing" technique you have described doesn't work reliably against preventing the split brain case.

Correct. A lease by itself does not give you a 100% split-brain prevention guarantee in the scenario you described.

Your GC-pause example is exactly why “I still have a lease, therefore I may write” is insufficient.

Suppose:

```text
t0: A is leader, lease valid
t1: A renews lease
t2: A enters 20s GC pause
t3: A's lease expires in etcd
t4: B becomes leader
t5: B starts writing
t6: A wakes up
t7: A resumes the operation it started before the pause
```

At t7, A's local state may still say:

```text
I am leader
```

and there is no way for A itself to know that this belief became stale while it was paused.

This is the classic reason that leases are a liveness mechanism, not sufficient fencing by themselves.

The important distinction is between two things people sometimes both call “fencing.”

A **weak form** is:

```text
before writing:
    check etcd: "am I still leader?"
    if yes:
        write
```

That is not safe, because you can always get:

```text
check succeeds
GC pause / network stall
leadership changes
old leader resumes
write happens
```

There is an unavoidable TOCTOU race between the check and the actual side effect.

The **stronger form** of fencing requires the target of the write itself to reject stale leaders.

For example, every leadership acquisition gets a monotonically increasing fencing token:

```text
A leader token = 41
B leader token = 42
```

Every write carries the token:

```text
WRITE(key=X, value=Y, fencing_token=41)
```

and the destination resource tracks the highest token it has accepted:

```text
highest_token_seen = 42
```

Then after A wakes up:

```text
A -> storage:
    WRITE(..., token=41)

storage:
    reject, because 41 < 42
```

That is real fencing.

The crucial point is:

> **The stale leader must not be trusted to fence itself. The receiver of the side effect must enforce the fence.**

So your timeline becomes:

```text
A gets token 41
A pauses

lease expires

B gets token 42
B writes with token 42
storage remembers 42

A wakes up
A writes with token 41
storage rejects it
```

Now the GC pause no longer matters.

This is also why just putting an epoch or term in etcd doesn't solve the problem unless the storage system being modified participates in checking that epoch.

If your writes are going into your own replicated storage system, this is usually handled naturally by the consensus protocol. In Raft, for example, terms and quorum rules prevent an old leader from successfully committing new log entries once a newer-term leader has taken over. The old process may still think it's leader for a while, but it cannot get a majority to commit writes.

That's a key distinction:

```text
split-brain belief:
    possible temporarily

split-brain committed writes:
    prevented
```

Consensus protocols generally don't guarantee that only one process in the universe ever believes “I am leader” at every instant. They guarantee that conflicting operations cannot both become committed according to the protocol's safety rules.

For external resources, though, things get harder.

Suppose your leader performs:

```text
charge credit card
send email
write to S3
call external HTTP API
```

and that external system does not understand your fencing token.

Then etcd cannot magically make that side effect safe.

A can wake up after its lease expires and still perform:

```text
POST /charge-card
```

If the remote system has no mechanism like:

```text
epoch=42
reject epochs < 42
```

then there is no general way for etcd to prevent the stale leader from doing it.

So the stronger statement is:

> **Perfect fencing requires cooperation from the resource being accessed.**

The resource must support one of several equivalent mechanisms: a monotonically increasing fencing token, compare-and-swap against an epoch/version, transactions that verify ownership and modify state atomically, or routing all operations through something that itself serializes access.

This is why etcd's own transactional primitives are safe when you modify etcd itself. You can atomically say something conceptually like:

```text
IF leadership_key == my_revision
THEN write data
```

because both the leadership check and the write happen inside the same linearizable etcd transaction.

There is no race like:

```text
check etcd
pause
write somewhere else
```

Instead:

```text
  one atomic operation
          |
          v
etcd:
    check leadership
    +
    perform write
```

But once the actual write goes to some different system:

```text
etcd                 database
  |                     |
check leadership        |
  |                     |
GC PAUSE                |
  |                     |
                      write
```

you've lost atomicity.

So your objection is exactly right, with one refinement: the fencing technique isn't inherently broken; **fencing enforced by the old leader is broken**. Proper fencing is enforced downstream by the resource receiving the write.

That's the part that makes fencing tokens materially different from leases.

---

## Q7 — Fencing tokens and the epoch-install barrier

> [!NOTE]
> **Question:** You kinda twisted my words) Above I never said that I was expecting the lease alone to solve the split-brain, I was specifically saying the "fence token still doesn't give you 100% guarantee from split-brain". And here is what I mean: if the old leader with the old fence token was able to make its write before the new leader gets to do the same (using the new fence token), the target store (the resource) would allow it, as it wouldn't have yet seen that the new leader has been elected (and its new token wouldn't have ever hit the store yet at that point). So the split brain still would exist.

Yes — that’s the more precise objection, and you're right.

A fencing token does not retroactively invalidate an old leader's write. If the resource has only seen token 41, then an old leader holding 41 can still get a write accepted even after some external system has already elected leader 42.

For example:

```text
A has token 41

t1  A pauses
t2  A's lease expires
t3  etcd elects B, token 42
t4  A wakes up
t5  A -> store: WRITE(..., token=41)
t6  store has only seen 41, so it ACCEPTS
t7  B -> store: WRITE(..., token=42)
```

Exactly. A naive rule such as:

```text
accept token >= max_token_seen
```

does not prevent the write at t5.

The important missing piece is that a proper fencing protocol normally needs a **fence-installation barrier** at the resource.

The new leader must do:

```text
1. acquire epoch/token 42
2. install epoch 42 at the target resource
3. wait until the resource acknowledges it
4. only then consider itself active leader
5. start performing normal writes
```

So:

```text
etcd                      store

A leader, epoch 41
   |
   | GC pause
   X

B elected, epoch 42
   |
   |--------------------> INSTALL_EPOCH(42)
   |                      max_epoch = 42
   |<-------------------- ACK
   |
B is now ACTIVE
```

After that barrier:

```text
A -> WRITE(token=41)
store -> REJECT
```

Now consider the race you described:

```text
B elected 42

A -> WRITE(41)
```

If that write reaches the resource before `INSTALL_EPOCH(42)`, then yes, it can succeed.

But the protocol defines that point as being **before B has actually acquired write ownership** of that resource.

So the ordering becomes:

```text
A write(41)         accepted
INSTALL_EPOCH(42)   accepted
B becomes active
B write(42)         accepted
```

There is a clean serialization point:

```text
                fence barrier
                      |
A operations ---------|--------- B operations
            epoch 41  |    epoch 42
```

There is no moment from the resource's perspective where operations from both epochs can be accepted on opposite sides of that barrier.

That distinction is crucial.

### Election and leadership activation aren't the same event

This is really the conceptual mistake in many simplified descriptions of leader election.

You shouldn't model it as:

```text
etcd says B is leader
=> B is immediately leader everywhere
```

Instead:

```text
B wins election
      |
      v
B acquires fencing token 42
      |
      v
B establishes token 42 at resource(s)
      |
      v
B may now serve requests
```

So you might distinguish:

```text
ELECTED
    !=
ACTIVE
```

B is elected at t3, but doesn't become the active writer until t7:

```text
t1 A active, token 41
t2 A pauses
t3 lease expires
t4 B wins election, token 42
t5 A sneaks in one last write(41)   <-- possible
t6 store installs epoch 42
t7 B becomes active
t8 A write(41)                      <-- rejected
t9 B write(42)                      <-- accepted
```

If your safety condition is:

> *Once B is elected in etcd, no write from A may ever succeed again,*

then etcd + fencing tokens cannot provide that guarantee by themselves.

There's an unavoidable information-propagation delay:

```text
etcd knows 42
      |
      | network
      v
store eventually knows 42
```

During that interval, the store cannot magically know that token 41 has become stale.

### And this points to a fundamental distributed-systems constraint

If these are two independent systems:

```text
+------+              +---------+
| etcd |              | storage |
+------+              +---------+
```

there is no atomic event that simultaneously means:

```text
etcd:    B is leader
storage: reject A
```

unless the two systems participate in some common transaction/consensus mechanism.

There will always potentially be:

```text
etcd committed new leader
           |
           | propagation window
           |
storage hasn't learned yet
```

You can move the linearization point to the storage system using the fence-installation barrier, but you cannot make two independent systems change state at precisely the same logical instant.

### The strongest design

For strict correctness, define the authoritative transition as something the resource itself participates in:

```text
             resource
                |
      epoch = 41, owner = A

B wins election
      |
      v
CAS(epoch=41 -> epoch=42, owner=B)
      |
      +---- succeeds
      |
      v
             resource
                |
      epoch = 42, owner = B
```

Every operation is then:

```text
WRITE(value, expected_epoch=42)
```

and internally the resource performs something equivalent to:

```text
atomic:
    if epoch != request.epoch:
        reject
    modify data
```

The transition:

```text
epoch 41 -> epoch 42
```

and ordinary writes are serialized by the same resource.

Then you can prove:

```text
                   epoch bump
                        |
A write ----------------|---------------- B write
epoch=41                |                 epoch=42
                        |
               linearization point
```

A write racing the transition lands on exactly one side:

```text
before epoch bump -> may succeed
after epoch bump  -> must fail
```

There is no ambiguous middle state.

And that's about as strong as you can make fencing.

So your point is an important correction to the often-heard statement:

> “Monotonic fencing tokens prevent stale leaders from writing.”

More precisely:

> **Monotonic fencing tokens prevent stale leaders from writing *after the resource has advanced to the newer fencing epoch*.**

If you need a strict handoff, the new leader must first establish that newer epoch at the resource and wait for acknowledgement before becoming active.

And if the resource can't participate in that protocol, then you cannot obtain an absolute no-split-brain guarantee merely by having etcd hand out ever-increasing tokens.

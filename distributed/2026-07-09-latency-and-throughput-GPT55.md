Q: How are latency and throughput are connected?

A: Latency and throughput are related, but they are **not generally inverses**.

- **Latency**: how long one operation takes.
- **Throughput**: how many operations complete per unit time.

The key connection is:

\[
\boxed{\text{Concurrency} = \text{Throughput} \times \text{Latency}}
\]

This is Little’s Law, usually written:

\[
L = \lambda W
\]

where:

- \(L\) = average number of operations in the system
- \(\lambda\) = throughput, operations per second
- \(W\) = average latency, seconds per operation

## Example

Suppose a service completes:

\[
1000 \text{ requests/second}
\]

and average request latency is:

\[
0.2 \text{ seconds}
\]

Then, on average:

\[
1000 \times 0.2 = 200
\]

requests are in flight.

So high throughput can coexist with relatively high latency, provided enough operations run concurrently.

---

## Why latency and throughput are not simply reciprocals

The formula

\[
\text{throughput} = \frac{1}{\text{latency}}
\]

is only true in the special case of a **single sequential worker** processing one operation at a time.

For example, if one worker takes 10 ms per request:

\[
\frac{1}{0.010} = 100 \text{ requests/second}
\]

But with 20 independent workers, each taking 10 ms:

\[
20 \times 100 = 2000 \text{ requests/second}
\]

The latency remains approximately 10 ms, while throughput becomes 2000 requests/s.

More generally:

\[
\text{Throughput} \approx
\frac{\text{effective parallelism}}{\text{latency}}
\]

provided there is enough demand and no shared bottleneck.

---

## Latency under increasing load

The most important practical relationship appears near system saturation.

At low load:

- requests rarely wait
- latency is close to actual processing time
- throughput increases roughly with incoming load

As load approaches capacity:

- workers, CPU cores, disks, connections, or other resources become busy
- requests begin waiting in queues
- latency rises

Past capacity:

- throughput stops increasing meaningfully
- queues continue growing
- latency may grow without bound
- timeouts and retries may reduce useful throughput

Conceptually:

```text
Throughput
    ^
    |                 ───────── capacity
    |              /
    |           /
    |        /
    |_____ /________________________> offered load
```

```text
Latency
    ^
    |                         /
    |                      /
    |                   /
    |_________________/___________> offered load
                    saturation
```

So around saturation:

\[
\boxed{\text{More load} \rightarrow \text{more queueing}
\rightarrow \text{higher latency, but little or no extra throughput}}
\]

---

## Processing time versus queueing time

Observed latency can be decomposed as:

\[
\text{Latency}
=
\text{Queueing time}
+
\text{Service time}
\]

Service time is the time the resource actually spends processing the request.

Queueing time is the time waiting for that resource.

Before saturation, queueing time may be negligible. Near saturation, queueing time often dominates total latency.

For example:

```text
Actual CPU/database work:     5 ms
Waiting in queue:           195 ms
Observed latency:           200 ms
```

The system has not necessarily become slower at performing the work. It has become busier, so each request waits longer before being served.

---

## A concrete example

Assume a database can execute 100 queries concurrently, and each query takes 20 ms once execution starts.

Its idealized maximum throughput is:

\[
\frac{100}{0.020}
=
5000 \text{ queries/second}
\]

At 1000 queries/s:

\[
L = 1000 \times 0.020 = 20
\]

Only about 20 queries are concurrently active, so the database is below capacity.

At 5000 queries/s:

\[
L = 5000 \times 0.020 = 100
\]

All 100 execution slots are occupied.

If clients now send 6000 queries/s, the database cannot sustainably complete 6000/s. The extra requests queue:

```text
Arrival rate:       6000/s
Completion rate:    5000/s
Queue growth:       1000 requests/s
```

Throughput stays near 5000/s, while latency continually increases.

---

## Improving latency may or may not improve throughput

Reducing service time often improves maximum throughput.

Suppose one worker takes 10 ms:

\[
100 \text{ operations/s}
\]

If optimized to 5 ms:

\[
200 \text{ operations/s}
\]

So halving service time can double single-worker capacity.

But reducing some component of end-to-end latency may not affect throughput. For example:

- reducing network round-trip time may improve client-visible latency
- but if the server’s CPU is still the throughput bottleneck, server capacity may remain unchanged

Similarly, batching may:

- increase throughput
- while increasing latency because an item waits for the batch to fill

That is a common throughput–latency tradeoff.

---

## Summary

The cleanest mental model is:

\[
\boxed{
\text{Throughput}
=
\frac{\text{average concurrency}}{\text{average latency}}
}
\]

But causally:

1. **Service time and parallelism determine capacity.**
2. **Offered load determines how much of that capacity is used.**
3. **When offered load approaches capacity, queueing raises latency sharply.**
4. **Once saturated, additional load mostly increases latency rather than throughput.**

Therefore, latency and throughput are connected through **concurrency, capacity, and queueing**, not by a universal inverse relationship.

Q: Above you said that when a system runs at/above its capacity, its througput flattens while its latency can grow unboundedly. Can you explan such a behavior of latency (i.e. it's clear why it'll grow, but why unboundedly?)?

A: “Unbounded” means there is **no finite steady-state upper bound** on queueing delay under the stated assumptions—not that every individual request literally waits forever.

Assume:

- requests arrive at rate \(\lambda\)
- the system can complete them at maximum rate \(\mu\)
- excess requests are allowed to wait in an effectively unlimited queue
- there are no timeouts, drops, or admission control

Observed latency is:

\[
W = W_{\text{queue}} + W_{\text{service}}
\]

The service time may remain roughly constant. It is the queueing component that can grow without bound.

## Above capacity: \(\lambda > \mu\)

Suppose:

\[
\lambda = 1100\ \text{requests/s}
\]
\[
\mu = 1000\ \text{requests/s}
\]

Every second:

- 1100 requests arrive
- only 1000 complete
- backlog increases by 100

Therefore, after \(t\) seconds:

\[
Q(t) \approx Q(0) + (\lambda-\mu)t
\]

So here:

\[
Q(t) \approx Q(0)+100t
\]

A newly arriving request must wait for approximately \(Q(t)\) requests ahead of it. Its queueing delay is roughly:

\[
W_{\text{queue}}(t) \approx \frac{Q(t)}{\mu}
\]

Therefore:

\[
W_{\text{queue}}(t)
\approx
\frac{Q(0)}{\mu}
+
\frac{\lambda-\mu}{\mu}t
\]

Since \(t\) can keep increasing, there is no finite maximum:

\[
\boxed{\lambda>\mu \implies Q(t)\to\infty \implies W(t)\to\infty}
\]

For the example:

```text
Time under overload     Backlog       Approximate queueing delay
0 s                     0             0 s
10 s                    1,000         1 s
60 s                    6,000         6 s
10 min                  60,000        60 s
1 hour                  360,000       360 s
```

Throughput remains about \(1000/s\), because that is all the system can process. The extra \(100/s\) becomes accumulated waiting work.

## Why throughput does not need to grow with the queue

The queue contains **demand**, not additional processing capacity.

```text
                    capacity = 1000/s
1100/s arrivals ──> [ growing queue ] ──> 1000/s completions
```

A queue can hold more and more requests, but it cannot make the CPU, database, disk, or worker pool process them faster.

Thus:

\[
\text{incoming work}
=
\text{completed work}
+
\text{growth of backlog}
\]

More precisely:

\[
\lambda = \mu + \frac{dQ}{dt}
\]

when the system is saturated. If \(\lambda>\mu\), the difference goes into queue growth.

---

## What about exactly at capacity: \(\lambda=\mu\)?

This case needs more nuance.

In a perfectly deterministic hypothetical system:

- exactly one request arrives every millisecond
- exactly one request finishes every millisecond
- arrivals and completions are perfectly aligned

the queue could remain bounded.

Real systems are variable:

- arrivals come in bursts
- request service times differ
- schedulers pause
- garbage collection happens
- storage occasionally takes longer
- network traffic is irregular

At exactly \(100\%\) average utilization, the system has no spare capacity to drain a queue created by one of these fluctuations.

For example:

1. A short burst creates a backlog of 100 requests.
2. After the burst, arrivals continue at exactly the system’s capacity.
3. The system completes requests as fast as new ones arrive.
4. It therefore never catches up.

To reduce the queue, the system needs a period where:

\[
\lambda < \mu
\]

At exactly equal average rates, random fluctuations push the queue upward and there is no systematic downward tendency. Queueing theory calls such a queue **unstable** or, at the boundary, non-positive-recurrent: it has no finite stationary queue-length distribution.

So the precise statement is:

\[
\boxed{
\text{For a variable queueing system, stable steady state generally requires }
\lambda < \mu
}
\]

Not merely:

\[
\lambda \leq \mu
\]

---

## Approaching capacity from below

Even before the arrival rate reaches capacity, average latency can become extremely large.

For the basic \(M/M/1\) queue:

\[
W = \frac{1}{\mu-\lambda}
\]

Suppose capacity is:

\[
\mu=1000/s
\]

Then:

| Arrival rate \(\lambda\) | Utilization | Mean latency \(W\) |
|---:|---:|---:|
| 500/s | 50% | 2 ms |
| 900/s | 90% | 10 ms |
| 990/s | 99% | 100 ms |
| 999/s | 99.9% | 1 s |
| 999.9/s | 99.99% | 10 s |

As:

\[
\lambda \to \mu^{-}
\]

we get:

\[
\mu-\lambda \to 0
\]

and therefore:

\[
W \to \infty
\]

This is a mathematical statement about the long-run average in that model. It captures the fact that, close to full utilization, the small amount of spare capacity available to clear random backlogs becomes vanishingly small.

## A useful “backlog draining” view

Suppose capacity is \(1000/s\), and a burst creates a backlog of 10,000 requests.

If normal arrivals afterward are \(900/s\), spare capacity is:

\[
1000-900=100/s
\]

Time to clear the backlog:

\[
\frac{10{,}000}{100}=100\text{ seconds}
\]

If normal arrivals are \(990/s\), spare capacity is only \(10/s\):

\[
\frac{10{,}000}{10}=1000\text{ seconds}
\]

At \(999/s\):

\[
\frac{10{,}000}{1}=10{,}000\text{ seconds}
\]

At exactly \(1000/s\), spare capacity is zero:

\[
\frac{10{,}000}{0}
\]

The backlog never drains.

That is the intuitive origin of the divergence.

## Real systems usually do not allow literal infinite latency

Actual systems usually have finite limits:

- bounded request queues
- client or server timeouts
- load shedding
- rejected requests
- connection limits
- circuit breakers
- backpressure
- autoscaling
- crashes or resource exhaustion

In those systems, latency does not necessarily increase forever. Instead, some other metric deteriorates:

```text
Unlimited queue:
    latency grows without bound

Bounded queue:
    queue fills, then requests are rejected or dropped

Timeouts:
    apparent latency caps near timeout, but failures increase

Backpressure:
    callers are slowed or prevented from submitting more work
```

For example, with a 30-second timeout, latency may appear “bounded” at 30 seconds. But that does not mean the overloaded system is healthy. It means requests waiting longer than 30 seconds are converted into failures.

Thus a more exact statement than the earlier one is:

> If sustained offered load meets or exceeds service capacity, and requests are queued without a finite bound, the queue has no stable steady state and queueing latency can grow without limit. With finite queues or timeouts, latency is capped by policy, but rejection, timeout, or loss rates rise instead.

Q: What is the industry-accepted practical higher bound of the system capacity to run at?

A: There is **no universal industry-standard percentage**, because the safe limit depends on burstiness, latency SLOs, scaling speed, redundancy requirements, and the resource that actually bottlenecks.

However, for a latency-sensitive online service, a widely used practical rule is:

\[
\boxed{\text{Target sustained peak load at roughly }60\%-70\%\text{ of tested capacity}}
\]

Treat sustained utilization above approximately **80%** as risky unless load is exceptionally smooth and the system has been specifically validated there.

Google Cloud gives 70% CPU as an example autoscaling trigger, warning that a 90% trigger may not leave enough time to handle bursts. Azure SQL recommends average CPU below 70%, although brief spikes to 90% may be acceptable. Google SRE emphasizes deriving capacity through load testing and retaining enough capacity for outages rather than relying on a universal utilization number. :chatgpt-content-reference{index="0"}

## Practical interpretation

| Sustained load relative to tested capacity | Typical interpretation |
|---:|---|
| Below 50% | Conservative; appropriate when failure headroom or extreme burst tolerance is required |
| 50–70% | Normal target range for latency-sensitive production services |
| 70–80% | Aggressive but sometimes acceptable with smooth load and rapid autoscaling |
| 80–90% | Little headroom; tail latency and overload risk usually become significant |
| 90–100% | Generally unsuitable as a steady operating point for an interactive service |
| Near 100% | Capacity test territory, not normal production operation |

Brief spikes are different from sustained operation. A system may reach 90–100% momentarily without trouble if queues remain short and it quickly returns to a lower utilization.

## Why approximately 70%?

Because the remaining 30% is needed for several different things:

- random traffic bursts;
- uneven load distribution and hotspots;
- slow or unusually expensive requests;
- garbage collection, compaction, maintenance, and background work;
- autoscaling delay;
- machine or zone failures;
- estimation error in the measured capacity.

It is not just a “safety margin.” It is also what lets the system **drain temporary queues**.

Suppose capacity is 1,000 requests/s and a burst creates a backlog of 10,000 requests.

At a normal load of 700 requests/s:

\[
\text{spare capacity}=1000-700=300/s
\]

The backlog drains in approximately:

\[
\frac{10{,}000}{300}\approx 33\text{ seconds}
\]

At 900 requests/s:

\[
\frac{10{,}000}{100}=100\text{ seconds}
\]

At 990 requests/s:

\[
\frac{10{,}000}{10}=1000\text{ seconds}
\]

So even though 90% or 99% is technically below capacity, temporary congestion can persist for a very long time.

## Queueing-theory illustration

For the simplified \(M/M/1\) model:

\[
W=\frac{S}{1-\rho}
\]

where:

- \(S\) is average service time with no queue;
- \(\rho\) is utilization;
- \(W\) is average total latency.

Therefore:

| Utilization \(\rho\) | Average latency relative to service time |
|---:|---:|
| 50% | \(2S\) |
| 70% | \(3.33S\) |
| 80% | \(5S\) |
| 90% | \(10S\) |
| 95% | \(20S\) |
| 99% | \(100S\) |

The exact numbers rarely apply directly to a real distributed system, but the shape is important:

\[
\boxed{\text{Latency rises nonlinearly as utilization approaches 100%}}
\]

This is why 80% is not merely “10 percentage points worse” than 70%. It can be substantially less stable under variable arrivals.

## Failure tolerance may require a much lower limit

Suppose a cluster has \(N\) equal nodes and must survive the loss of \(f\) nodes without exceeding some acceptable post-failure utilization \(U_{\max}\).

Its normal-load utilization should satisfy approximately:

\[
U_{\text{normal}}
\leq
U_{\max}\frac{N-f}{N}
\]

Example: a four-node cluster must survive one node failure, and you do not want the surviving nodes to exceed 80%:

\[
U_{\text{normal}}
\leq
0.8\frac{3}{4}
=
0.6
\]

So the cluster should normally run at no more than approximately **60% of total installed capacity**.

Likewise, two equally sized active regions that must each absorb the other region’s entire load generally cannot both run much above 50% before failover—unless degraded service or emergency scaling is acceptable. Google SRE recommends explicitly provisioning for simultaneous planned and unplanned outages, illustrating why redundancy requirements often dominate the percentage target. :chatgpt-content-reference{index="1"}

## Important exception: batch systems

A batch-processing system may intentionally run CPU at 90–100% because:

- individual-job latency is not important;
- backlog is expected;
- maximum throughput is the objective;
- failure recovery does not require immediate spare capacity.

For an interactive API, database, queue consumer, or user-facing service, that same operating point is usually dangerous. Google SRE documents that insufficient steady-state capacity causes queues and latency to grow and can trigger retries, resource exhaustion, and cascading failure. :chatgpt-content-reference{index="2"}

## The most defensible rule

For a typical latency-sensitive service:

\[
\boxed{
\begin{aligned}
&\text{Design target: } 60\%-70\%\\
&\text{Aggressive steady-state ceiling: } \sim80\%\\
&\text{Brief tested spikes: } 80\%-90\%\\
&\text{Normal operation near }100\%: \text{avoid}
\end{aligned}}
\]

But “capacity” must mean **maximum sustainable throughput while still meeting the latency and error-rate SLO**, established through representative load testing—not the absolute point at which the system stops completing additional work.

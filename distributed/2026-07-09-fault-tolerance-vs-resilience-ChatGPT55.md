# Fault Tolerance vs. Resiliency vs. Availability (ChatGPT 5.5)

*A conversation with ChatGPT 5.5: fault tolerance as a mechanism, availability as an observed service property, resilience as the broader ability to handle adversity - how they relate, concrete examples, and "fault model" vs. "failure model".*

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; What is the difference between:
<br>
1. Fault tolerance
<br>
2. Resiliency
<br>
3. Availability
<br><br>
?
<br><br>
It feels like those terms are defined somewhat differently sometimes with quite a bit of overlap.. Also, those seem to be interconnected in many ways. Can you try and untangle it using most adopted industry-accepted definitions?
</td></tr>
</table>

You are right: the terms overlap because they describe **different aspects of system behavior**, not three mutually exclusive categories.

The cleanest industry-oriented distinction is:

> **Fault tolerance is a capability/mechanism.**  
> **Availability is an externally observed service property.**  
> **Resilience is the broader ability to handle adversity over time.**

### 1. Fault tolerance

**Fault tolerance is the ability of a system to continue providing correct service despite specified faults.**

NIST defines it as a system property allowing proper operation even when components fail, or continued correct execution in the presence of hardware or software faults.

The words **specified faults** are important. A system is never simply “fault tolerant” without qualification:

- tolerant of one server crash;
- tolerant of two disk failures;
- tolerant of network packet loss;
- tolerant of one Byzantine replica;
- tolerant of an Availability Zone failure.

It may still fail under faults outside that model.

#### Fault → error → failure

The standard dependability terminology distinguishes:

```text
Fault
  ↓ causes
Error: incorrect internal system state
  ↓ may cause
Failure: externally visible incorrect service
```

For example:

```text
Disk develops a bad sector       ← fault
A database page becomes corrupt  ← error
A query returns incorrect data   ← failure
```

Fault tolerance attempts to prevent the fault or resulting error from becoming an externally visible failure - for example through replication, voting, error-correcting codes, retries, failover, or reconfiguration. This fault/error/failure distinction comes from the classic IEEE dependability taxonomy.

#### Key characteristic

Fault tolerance usually tries to **mask or contain faults while operation continues**.

```text
Component fault
      ↓
Detection / masking / failover
      ↓
Service continues correctly
```

There may be a small latency spike, but ideally the client sees no failed operation.

---

### 2. Availability

**Availability is whether the service is operational, accessible, and usable when required.**

ISO/IEC 25010 defines availability as the capability of a product to be operational and accessible when required for use. NIST similarly defines it as being accessible and usable on demand.

Availability is generally an **observable outcome**, often measured as:

$$
Availability =
\frac{\text{time service satisfies its availability requirement}}
{\text{total relevant time}}
$$

A simplified repairable-system approximation is:

$$
A \approx \frac{MTBF}{MTBF + MTTR}
$$

where:

- **MTBF** = mean time between failures;
- **MTTR** = mean time to restore or repair.

#### Availability does not specify how it is achieved

A system can achieve high availability through:

- fault tolerance;
- fast automatic restart;
- rapid failover;
- short repair times;
- operational procedures;
- redundancy;
- reduced deployment downtime.

Therefore:

> **Fault tolerance is one way to improve availability, but availability does not imply fault tolerance.**

For example, a single process that crashes once a year and restarts in one second may have excellent measured availability, but it is not fault tolerant: the process really does fail, even though recovery is fast.

Conversely, a system may tolerate one disk failure perfectly but still have poor availability because deployments, overloads, operator errors, or dependency failures frequently take it down.

#### Availability requires a service definition

“99.99% available” is incomplete unless you define what counts as available:

```text
HTTP endpoint responds?
HTTP endpoint returns non-5xx?
99% of requests finish within 500 ms?
Reads and writes both work?
Only critical functions work?
Responses are semantically correct?
```

A server returning `200 OK` with incorrect data should not normally count as providing correct service, but a poorly designed availability metric might count it. Availability is separate from integrity, consistency, and correctness requirements.

---

### 3. Resilience

**Resilience is the broader capability to prepare for, withstand, recover from, and adapt to adversity while preserving required or critical capabilities.**

NIST gives several closely related definitions, including maintaining required capability in the face of adversity and continuing essential operations - possibly in a degraded state - then recovering within a mission-appropriate time.

A useful lifecycle model is:

```text
Anticipate / prepare
        ↓
Withstand / absorb
        ↓
Continue, possibly degraded
        ↓
Recover
        ↓
Adapt / improve
```

Resilience is broader than fault tolerance in several ways.

#### Broader range of adverse events

Fault tolerance normally focuses on faults covered by an engineering fault model.

Resilience may cover:

- hardware and software failures;
- network partitions;
- overload and traffic spikes;
- cyberattacks;
- operator mistakes;
- data corruption;
- cloud-region outages;
- third-party dependency failures;
- natural disasters;
- unexpected or previously unmodeled conditions.

NIST’s cyber-resiliency formulation explicitly includes anticipating, withstanding, recovering from, and adapting to adverse conditions, stresses, attacks, and compromises.

#### Degraded operation can still be resilient

A resilient system does not necessarily maintain full functionality.

For example, during a major failure:

```text
Normal mode:
    reads + writes + reporting + recommendations

Degraded mode:
    reads only; critical transactions preserved

Recovered mode:
    full operation restored
```

This can be a successful resilience response even though full availability temporarily decreased.

#### Recovery is part of resilience

Fault tolerance emphasizes continuing despite faults. Resilience also includes what happens when continuation is impossible:

- limiting the blast radius;
- preserving critical data;
- restoring essential service;
- rebuilding redundancy;
- learning or adapting afterward.

---

## The relationship

A useful conceptual picture is:

```text
                         RESILIENCE
        ┌───────────────────────────────────────┐
        │ Prepare                               │
        │                                       │
        │   ┌──────── Fault tolerance ───────┐  │
        │   │ Mask/contain specified faults  │  │
        │   │ and continue correct service   │  │
        │   └────────────────────────────────┘  │
        │                                       │
        │ Degrade safely                        │
        │ Recover                               │
        │ Rebuild                               │
        │ Adapt                                 │
        └───────────────────────────────────────┘

Availability = externally observed result:
“How often was the required service actually usable?”
```

It is not quite correct to draw availability simply as a subset of resilience. Availability is better viewed as a **measurable service attribute or outcome**, while resilience and fault tolerance describe capabilities that influence it.

### Comparison

| Concept | Main question | Scope | Failure may become visible? | Usually measured as |
|---|---|---|---:|---|
| **Fault tolerance** | Can the system continue correctly despite particular faults? | Specific fault model and mechanisms | Ideally no | Number/type of faults tolerated |
| **Availability** | Is the required service usable when needed? | Externally observed service | Yes, reflected as downtime or failed requests | Percentage, SLI/SLO, downtime |
| **Resilience** | Can the system preserve critical capability, limit damage, recover, and adapt under adversity? | End-to-end system, organization, and lifecycle | Possibly; degraded service or outage may occur | Impact, degradation, recovery time, data loss, adaptability |

## Concrete examples

### Example 1: replicated database loses one replica

Suppose a database has three replicas and can serve requests using two.

```text
Replica A: alive
Replica B: alive
Replica C: crashes
```

- **Fault tolerant:** Yes, if it is designed to tolerate one replica crash and still serve correct results.
- **Available:** Yes, while the remaining replicas serve requests within the defined SLO.
- **Resilient:** More likely yes if it also detects the loss, creates a replacement replica, restores redundancy, handles reduced capacity, and avoids recurrence or uncontrolled cascading failure.

Fault tolerance explains why the immediate crash was masked. Resilience describes the broader reaction and restoration process.

### Example 2: backup restoration after a ransomware incident

A system is taken offline, rebuilt, and restored from immutable backups after four hours.

- **Fault tolerant:** No. The failure was not masked; service stopped.
- **Available:** Unavailable for four hours.
- **Resilient:** Potentially yes, if critical data was preserved and service was restored within the organization’s recovery objectives.

This is the clearest demonstration that resilience does not require uninterrupted availability.

### Example 3: extremely reliable single server

A single server runs for three years without failing.

- **Available:** Historically, yes - possibly extremely available.
- **Fault tolerant:** No; one server failure can stop the service.
- **Resilient:** Probably weak, unless there are effective restoration, backup, disaster-recovery, and operational capabilities.

Good historical availability does not prove that the architecture can withstand future faults.

### Example 4: graceful degradation during overload

An online store disables recommendations and analytics so checkout remains operational.

- Some noncritical functions are unavailable.
- The most important business capability remains available.
- The system is demonstrating resilience through load shedding and graceful degradation.
- This is not necessarily fault tolerance because overload is not always modeled as a component fault.

## Why industry usage becomes confusing

There is no single universally binding definition covering every industry and context. NIST itself warns that glossary definitions vary by source and context.

Three common sources of confusion are:

1. **“Resilient” is used as marketing shorthand for “highly available.”**  
   Technically, resilience is broader.

2. **“Fault tolerant” is sometimes used to mean “automatically recovers.”**  
   Strictly, restart-after-failure is recovery. Fault tolerance usually means preventing a covered fault from causing service failure, although some definitions include recovery techniques within the broader fault-tolerance process.

3. **Availability is sometimes treated as an architectural mechanism.**  
   “High availability” architectures do use mechanisms such as redundancy and failover, but availability itself is the resulting service property.

Also, **resilience** is generally the more standard noun in engineering literature. **Resiliency** is commonly used, especially in business and cybersecurity, and normally means the same thing.

## A precise working vocabulary

For architecture discussions, I would use the terms this way:

> **Fault tolerance:** The system continues to provide correct service despite a declared class and number of faults.

> **Availability:** The proportion of required occasions or time during which the defined service is operational and usable within its service-level requirements.

> **Resilience:** The system’s broader ability to anticipate adversity, limit its impact, preserve critical capabilities - possibly in degraded form - recover within required limits, and adapt afterward.

And the central relationship is:

```text
Fault tolerance
      ↓ contributes to
Availability
      ↑ also affected by recovery, repair, operations, deployments, etc.

Fault tolerance + graceful degradation + containment
+ recovery + adaptation + operational preparedness
      ↓
Resilience
```

Thus, a compact but accurate summary is:

> **Fault tolerance is about continuing through faults. Availability is about whether users can obtain the service. Resilience is about surviving the complete adverse event - including preparation, impact, degraded operation, recovery, and adaptation.**

<table style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;">
<tr><td style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;">
<strong style="font-size: 1.4em;">Q:</strong>&nbsp; "fault model" or "failure model" - which one is the correct term?
</td></tr>
</table>

**Both terms are used, but they are not perfectly synonymous under strict dependability terminology.**

### Precise distinction

The standard fault–error–failure chain is:

```text
fault → error → failure
```

- **Fault:** the underlying cause of a problem.
- **Error:** an incorrect internal system state.
- **Failure:** externally observable deviation from the specified service.

Therefore:

#### Fault model

A **fault model** specifies the faults the system assumes may occur, including their nature and bounds:

- hardware defects or bit flips;
- software bugs;
- maliciously controlled nodes;
- transient versus permanent faults;
- at most $f$ faulty replicas;
- faults occurring independently versus correlated faults.

It answers:

> **What can go wrong internally, and under what assumptions?**

#### Failure model

A **failure model** specifies how a component is allowed to behave incorrectly at its externally visible interface:

- crash-stop;
- crash-recovery;
- send omission;
- receive omission;
- timing failure;
- arbitrary or Byzantine behavior.

It answers:

> **How may an affected process or component deviate from its specification?**

For example:

```text
Underlying fault:
    power-supply defect

Resulting failure behavior:
    process stops permanently
```

The fault belongs to the **fault model**; “stops permanently” belongs to a **crash-stop failure model**.

### Why the distinction is blurry in distributed-systems literature

Distributed algorithms generally cannot observe the underlying physical fault. They can only reason about its resulting behavior:

```text
Did the process stop?
Did it omit messages?
Did it send contradictory messages?
```

Consequently, literature often calls behavioral assumptions **fault models** anyway:

- crash fault model;
- omission fault model;
- Byzantine fault model.

This usage is deeply established. Research papers routinely use phrases such as “Byzantine fault model” and “crash fault model,” even though those models primarily describe the behavior of faulty processes.

So this is not simply incorrect terminology - it is a conventional abstraction:

> A process is *faulty* if its behavior does not conform to the algorithm; the **fault model** describes the deviations that a faulty process may exhibit.

### Best terminology in practice

When discussing a distributed algorithm, I would normally write:

> **The algorithm assumes a crash-stop failure model and tolerates at most $f$ faulty processes.**

That is maximally precise:

- **failure model** describes the externally observable behavior;
- **faulty processes** identifies which participants violate the specification;
- $f$ specifies the tolerated bound.

But this is also completely conventional:

> **The algorithm operates under a crash fault model.**

For the statement from my previous answer -

> “A system is fault tolerant only relative to a specified fault model”

\- **fault model** was the appropriate broader term. It may include both:

1. which faults may occur and how many; and
2. what behavior those faults may cause.

The cleanest summary is:

```text
Fault model:
    assumptions about what can go wrong

Failure model:
    assumptions about how the resulting incorrect
    behavior may appear externally
```

In distributed-systems writing, however, **“fault model” is frequently used as the umbrella term for both**.

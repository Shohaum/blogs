---
title: "Designing a Unique OTP System on a Single Server Part 1: Requirements & Capacity"
description: "A deep dive into designing a secure, efficient, and scalable One-Time Password (OTP) system."
date: "2026-11-15"
tags:
  - System Design
---

| # | Ambiguity | Assumption | Why this matters |
|---|---|---|---|
| A1 | "Unique OTP" could mean unique across all time, or unique among currently active OTPs. | **Unique among currently active OTPs.** An OTP value may be reissued after the previous holder's OTP has been consumed, expired, or revoked. | A 6-digit space has only $10^6$ values. Global-forever uniqueness is impossible for any real service. |
| A2 | What is an "authentication context"? | A **subject** is the pair `(user_id, purpose)`, for example `(u_42, "login")` or `(u_42, "password_reset")`. | The same user may legitimately have a login OTP and a reset OTP at once. |
| A3 | How many active OTPs per subject? | **Exactly one.** Requesting a new one revokes the previous one. | Simplifies invariants and shrinks the attack surface. |
| A4 | Numeric or alphanumeric? | **6-digit numeric by default** (human-friendly on a phone keypad). The design also works for 8 digits. | Search-space size drives every probability in this guide. |
| A5 | Who calls this service? | A trusted **backend or API gateway** that has already authenticated the *caller*. The user is not yet authenticated (that is the point of an OTP). | Decides which endpoints need which credentials. |
| A6 | Durability across restarts? | **Not required.** If the process restarts, all active OTPs are lost and users simply request a new one. | Allows an in-memory design. |
| A7 | Delivery channel | SMS or email through an **external provider** over HTTPS. The provider is slow (50 ms to several seconds) and can fail. | Delivery latency must never be paid while holding a lock. |
| A8 | Hardware | One VM: **8 vCPU, 16 GiB RAM, NVMe disk, 1 Gbps NIC**. A reverse proxy (for example nginx) terminates TLS on the same machine. | Capacity math needs concrete numbers. All are labelled *hypothetical*. |
| A9 | Time | OTP lifetime $T = 300$ s (5 minutes), configurable. | Drives the number of active OTPs. |
| A10 | Process model | **One application process, many threads (or one async event loop).** | This is the single most consequential structural choice. |

### Requirements Analysis

Imagine a hotel with 1,000,000 possible room-key codes. When a guest checks in, the front desk hands over a code. Two things could go wrong:

1. **Two guests are handed the same code on the same night.** Either guest can now open the other's room.
2. **A code is correct, but is used on the wrong door.** Guest A learns a code that is valid for Guest B's room, and tries it on B's door.

These are different failures with different fixes. The first is a **uniqueness** failure at the front desk. The second is a **binding** failure at the door. The brief asks for both to be impossible, and a recurring theme of this guide is that **you need both**, because each one alone leaves a hole.

**Two properties that can confuse**

| Property | Question it answers | Enforced where | Failure if missing |
|---|---|---|---|
| **Uniqueness** | "Can two active challenges ever hold the same OTP value?" | At **generation** time (the reservation step) | Two users receive the same code. A user who learns another's code may collide with it. |
| **Binding** | "Is this OTP acceptable *for this specific user and purpose*?" | At **verification** time (the lookup step) | A code issued to B can be replayed into A's session if the verifier looks up by code alone. |

**Uniqueness does not give you security.** Even with perfect uniqueness, a verifier that asks "is this code currently valid for anyone?" can be attacked by random guessing against *all* active codes simultaneously. With $N$ active codes among $S$ possible values, one random guess succeeds with probability $N/S$. Binding turns that into $1/S$ per guess *against a single challenge*, which can then be capped by an attempt limit.

**Binding does not give you uniqueness.** If two users hold the same code but verification is strictly bound to the challenge, nothing breaks cryptographically. But two things get worse: (a) auditing and support become confusing, and (b) the brief explicitly lists uniqueness as a requirement and an invariant that defends against implementation bugs, such as someone later adding a "lookup by code" convenience path.

So we implement **both** and treat uniqueness as a *defence in depth* invariant, not a substitute for binding.

#### Functional requirements

| ID | Requirement |
|---|---|
| FR1 | Generate an OTP for a subject `(user_id, purpose)` and return an opaque `challenge_id`. |
| FR2 | Deliver the OTP out of band (SMS or email). The API response must **never** contain the OTP. |
| FR3 | Verify an OTP against a specific `challenge_id` and `user_id`. |
| FR4 | Consume the OTP on first successful verification. A second success with the same OTP is impossible. |
| FR5 | Expire an OTP after a configurable lifetime $T$. |
| FR6 | Allow regeneration: requesting a new OTP revokes the previous OTP of the same subject. |
| FR7 | Allow explicit invalidation (for example the user cancels, or a password change invalidates pending resets). |
| FR8 | Limit verification attempts per challenge. |

#### Non-functional requirements, grouped

**Security requirements**

| ID | Requirement |
|---|---|
| SR1 | An OTP value is held by at most one active challenge at any instant. |
| SR2 | An OTP is only accepted for the exact `(user_id, purpose, challenge_id)` it was issued for. |
| SR3 | Resistance to brute force: attacker success probability per challenge is bounded by $A/S$ ($A$ = attempt limit, $S$ = space size). |
| SR4 | Resistance to replay: a consumed, expired, or revoked OTP is never accepted. |
| SR5 | Resistance to enumeration: responses must not reveal whether a user exists, whether an OTP exists, or which part of a check failed. |
| SR6 | OTP values never appear in logs, metrics, error messages, or API responses. |
| SR7 | Timing side channels must not reveal how close a guess is. |

**Performance requirements**

| ID | Requirement |
|---|---|
| PR1 | p99 latency for generate (excluding delivery) under 50 ms at the sustainable load. |
| PR2 | p99 latency for verify under 20 ms at the sustainable load. |
| PR3 | Predictable behaviour at and above capacity: bounded queues, explicit rejection instead of unbounded latency growth. |

**Concurrency requirements**

| ID | Requirement |
|---|---|
| CR1 | Correct under arbitrary interleaving of any number of simultaneous requests. |
| CR2 | Multi-step operations (check-then-reserve, check-then-consume) must be atomic as a *whole*. |
| CR3 | Background cleanup must never corrupt or delete state belonging to a newer generation. |

**Operational requirements**

| ID | Requirement |
|---|---|
| OR1 | Configurable TTL, attempt limit, code length, and capacity cap. |
| OR2 | Metrics: active count, free-pool size, reservation latency, lock wait time, rejection counts. |
| OR3 | Graceful shutdown: stop accepting new work, drain in-flight requests. |
| OR4 | Behaviour on restart is defined and safe (all OTPs void). |

#### Invariants

An **invariant** is a statement that must be true at *every instant*, not only at the end of a request. Concurrency bugs are precisely moments where an invariant is briefly or permanently false. Writing them down first gives us something to test the design against.

Let $\mathcal{A}$ be the set of OTP values currently held by active challenges, and $\mathcal{F}$ the set of values currently available for issue. Let $\mathcal{S}$ be the full space of $S$ values.

| ID | Invariant | Meaning |
|---|---|---|
| **I1** | $\mathcal{A} \cap \mathcal{F} = \emptyset$ and $\mathcal{A} \cup \mathcal{F} = \mathcal{S}$ (ignoring quarantined values, introduced later) | Every value is either active or free, never both. This is the **uniqueness** invariant. |
| **I2** | The map `value -> challenge` is a function on $\mathcal{A}$ (each active value has exactly one owner). | No two challenges share a value. |
| **I3** | Each subject has at most one challenge in state `ACTIVE`. | Regeneration revokes before it replaces. |
| **I4** | A challenge in state `ACTIVE` has `now < expires_at`, or it is about to be reclaimed. A verify call **never** accepts when `now >= expires_at`, regardless of whether cleanup has run. | Correctness must not depend on the cleaner's timing. |
| **I5** | A challenge transitions `ACTIVE -> CONSUMED` at most once, and only on a correct, in-time, in-limit verify. | No double-spend. |
| **I6** | Once a challenge leaves `ACTIVE`, it never returns to `ACTIVE`. | Terminal states are terminal. |
| **I7** | `attempts` is monotonically non-decreasing and never exceeds the attempt limit plus the number of in-flight concurrent attempts, which the lock reduces to exactly the limit. | Brute-force cap holds under concurrency. |
| **I8** | A value returns to $\mathcal{F}$ at most once per issuance. | No duplicate free-list entries (which would later cause double issue). |

### Capacity Estimation

A coffee shop has one barista and a till. A queue forms. If customers arrive more slowly than the barista works, the line stays short. If they arrive slightly faster, the line grows forever. And as the barista approaches 100% busy, the wait time does not grow gently: it **explodes**.

Our server is the shop. Before designing anything we need to know **what is the barista's real speed, and what else (not the barista) limits the shop?** The most common mistake in capacity planning is to assume the CPU is the limit when something else (here, the SMS provider and the key space) hits first.

#### Three things people call "load"

| Term | Definition | Unit | Example |
|---|---|---|---|
| **Concurrent users** | People currently somewhere in an OTP flow (waiting for the SMS, typing the code). Most are doing nothing server-side. | users | 12,000 |
| **Concurrent requests** | HTTP requests being processed by the server *at this instant*. | requests in flight | ~5 |
| **Active OTPs** | OTPs currently reserved (state `ACTIVE`). Each holds one value out of $S$. | OTPs | ~26,000 |

They are linked by **Little's Law**:

$$L = \lambda \cdot W$$

where $L$ is the average number of items *in the system*, $\lambda$ the arrival rate, and $W$ the average time each item *stays*. It holds for any stable system, whatever the arrival pattern, which is why it is such a powerful sanity tool.

Intuition: if 10 customers arrive per minute and each stays 3 minutes, there are on average 30 in the shop.

#### Hypothetical workload assumptions

> Every number in this table is an **assumption** chosen to be plausible, not measured. Replace with your real traffic. The method matters more than the numbers.

| Symbol | Meaning | Value | Basis |
|---|---|---|---|
| $\lambda_{gen}$ | Peak OTP generation rate | 200 /s | Equal to the SMS provider contract limit (assumed 200 messages/s) |
| $v$ | Average verify attempts per generated OTP | 1.3 | Most users succeed first time, some mistype |
| $\lambda_{ver}$ | Peak verification rate | $200 \times 1.3 = 260$ /s | |
| $T$ | OTP lifetime | 300 s | A9 |
| $p_c$ | Fraction consumed early (user verifies) | 0.70 | |
| $t_c$ | Mean time until consume, for those who do | 60 s | Time to read the SMS and type it |
| $S$ | Key space | $10^6$ (6 digits) | A4 |
| $c_{req}$ | CPU time per request, all layers (proxy, TLS, parse, handler, serialization, logging) | 300 µs | Conservative for a compiled or well-tuned runtime |
| $c_{cs,gen}$ | Lock critical-section time, generation | 2 µs | Random index, swap-remove, three table writes |
| $c_{cs,ver}$ | Lock critical-section time, verification | 1 µs | One lookup, one compare, one state write |
| $m_{rec}$ | Memory per active record | 96 B (compact) to 500 B (high-level language dict) | |

#### Mean OTP lifetime in the system

An OTP either gets consumed early (probability $p_c$, after $t_c$) or sits until it expires (probability $1-p_c$, after $T$):

$$
\bar{W} = p_c\, t_c + (1-p_c)\,T = 0.70 \cdot 60 + 0.30 \cdot 300 = 42 + 90 = 132 \text{ s}
$$

#### Active OTPs (Little's Law)

$$
N_{typ} = \lambda_{gen}\,\bar{W} = 200 \times 132 = 26{,}400
$$

Worst case, where nobody ever verifies (for example a delivery outage makes users give up, or an attacker floods generation):

$$
N_{worst} = \lambda_{gen}\,T = 200 \times 300 = 60{,}000
$$

Load factor of the key space:

$$
\rho_{space} = \frac{N}{S}:\qquad \rho_{typ} = 2.64\%,\quad \rho_{worst} = 6\%
$$

#### Concurrent users in the flow

If a user spends about 60 s between "send me a code" and "verified" (reading, typing):

$$
L_{users} = \lambda_{gen} \times 60 = 12{,}000
$$

Twelve thousand people, yet only a handful of HTTP requests execute at any instant. This is the difference between concurrent users and concurrent requests.

#### Concurrent requests (in flight)

If delivery is **asynchronous** (the request returns after reservation and enqueue), a request takes about 5 ms of wall-clock:

$$
L_{req} = (\lambda_{gen} + \lambda_{ver}) \times W_{req} = 460 \times 0.005 \approx 2.3
$$

If delivery were **synchronous** (the request waits for the provider, about 400 ms):

$$
L_{req,sync} = 460 \times 0.4 \approx 184
$$

Hence a hidden consequence: synchronous delivery multiplies in-flight requests by about 80, tying up threads and connections for no useful work. This is one reason why we design delivery to be asynchronous.

> The in-flight count for generate requests alone is $200 \times 0.4 = 80$ threads blocked on the SMS provider. If the provider degrades to 5 s latency, how many threads do we need? ($200 \times 5 = 1000$.) What happens to the server if we only have 200 threads? This is a **cascading failure** and capacity planning must anticipate it.

#### Finding the real bottleneck

A system's sustainable throughput is the **minimum** across all of its resources. Compute the ceiling of each.

| Resource | Calculation | Ceiling |
|---|---|---|
| **CPU** | Usable cores 6 (2 reserved for proxy, OS, log shipper). $6 / 300\,\mu s = 20{,}000$ rps at 100%. Plan for 70% utilization: **14,000 rps** | 14,000 req/s |
| **Lock (single global mutex)** | $1 / 2\,\mu s = 500{,}000$ ops/s | 500,000 ops/s |
| **Network** | ~1.5 KB per request/response pair. $14{,}000 \times 1.5\text{ KB} = 21$ MB/s = 168 Mbps of 1 Gbps | not binding |
| **Memory** | Discussed below | not binding |
| **Key space (policy cap)** | Cap $N_{cap} = 200{,}000$ active (20% load). $N_{cap}/T = 666$ generations/s if none consumed early | 666 gen/s |
| **SMS provider** (assumed) | 200 messages/s | **200 gen/s** |

**The bottleneck is the SMS provider, not our server.**

Our CPU could in principle process 14,000 requests per second, but only 200 per second of those can be OTP generations that actually get delivered. So **the maximum sustainable OTP generation rate is 200/s**, and the question "how many concurrent users can this server handle" has the following honest answer:

* In the OTP flow: about 12,000 users concurrently, bounded by delivery rate and flow duration, not by CPU.
* Pure verification traffic (replaying already-delivered codes): far higher, up to the CPU ceiling.

> If you replace SMS with a faster email provider (say 2,000/s), the ceiling moves to $\min(14{,}000\text{ CPU},\ 666\text{ key space}) = 666$ gen/s, and the key space cap becomes the binding one. That is the signal to move to 8 digits.

#### Memory

Per active record:

| Field | Bytes |
|---|---|
| `challenge_id` (128-bit random) | 16 |
| `user_id` (64-bit) | 8 |
| `purpose` (enum) | 1 |
| `code` (uint32) | 4 |
| `state` | 1 |
| `attempts` | 1 |
| `created_at` (monotonic ms) | 8 |
| `expires_at` | 8 |
| `generation` (uint64) | 8 |
| padding and hash-table overhead (about 2x) | ~41 |
| **Total** | **~96** |

* Typical: $26{,}400 \times 96\,\text{B} \approx 2.5$ MB.
* At cap: $200{,}000 \times 96\,\text{B} \approx 19$ MB.
* In a high-level language (Python dicts and objects, about 500 B per record): at cap about 100 MB.

Fixed structures for a 6-digit space:

* Free pool array: $10^6 \times 4\ \text{B} = 4$ MB.
* Optional `owner[code]` index: $10^6 \times 4\ \text{B} = 4$ MB.

**Conclusion: memory is a non-issue.** A 16 GiB machine could hold the structures a thousand times over. The cost that matters is not *size* but *how often we touch it under a lock*.

#### Garbage collection and allocation overhead

* The live heap is tiny (under 130 MB even in a dynamic language). A modern collector will handle this with sub-millisecond pauses in a compiled managed runtime.
* In a language with a global interpreter lock, GC is not the issue but **the GIL** is.
* Design rule: **pre-allocate** the pool and index arrays at startup. The hot path should allocate nothing beyond the response buffer.

### Lock contention estimate

Model the global mutex as a single-server queue. Utilization is:

$$
\rho_{lock} = \lambda_{lock}\cdot s
= (200 \times 2\,\mu s) + (260 \times 1\,\mu s)
= 0.00066
= 0.066\%
$$

Even at the *full CPU-bound* rate of 14,000 requests/s, with about 2 µs each, $\rho_{lock} = 2.8\%$.

Intuition: the lock is held for 2 µs out of every 5,000 µs, so a thread that wants it finds it busy about 1 time in 36. Even then it waits at most a couple of microseconds. **A single global lock is not a bottleneck here**, and that conclusion, not preference, is what justifies choosing it. The lock would only become a problem if someone puts *slow work* (I/O, hashing, logging, allocation) inside it.

### What happens at and above maximum capacity

#### Why waiting time explodes

For a simple single-server queue, the mean time in system is:

$$
W = \frac{1}{\mu - \lambda} = \frac{1}{\mu(1-\rho)}
$$

| Utilization $\rho$ | Wait multiplier vs. idle ($1/(1-\rho)$) |
|---|---|
| 0.50 | 2x |
| 0.70 | 3.3x |
| 0.90 | 10x |
| 0.95 | 20x |
| 0.99 | 100x |
| $\ge 1.0$ | unbounded (queue grows without limit) |

This is why we plan at **70% utilization**: at 90% a request that normally takes 5 ms takes 50 ms, and any burst pushes it into the steep part of the curve.

### The delivery queue is the first thing to overflow

Assume incoming generation rate $\lambda_{in} = 300$/s but provider rate $\mu_{out} = 200$/s:

$$
\frac{dQ}{dt} = \lambda_{in} - \mu_{out} = 100 \text{ per second}
$$

With a bounded outbound queue of 5,000 entries, it fills in $5000/100 = 50$ s. Worse, an OTP at the back of a full queue waits $Q/\mu_{out} = 5000/200 = 25$ s before it is even sent, eating into its 300 s lifetime, and users start requesting resends, which make the queue longer. (This is a **retry storm**.)

### The overload policy

| Layer | Limit | Behaviour when exceeded |
|---|---|---|
| In-flight request limit (semaphore) | e.g. 512 concurrent requests | Reject immediately with `503` + `Retry-After`. Never queue unbounded. |
| Per-subject and per-IP rate limits | e.g. 1 generation per 30 s per subject | `429` |
| Delivery queue | bounded (e.g. 5,000) with **age-based shedding** (drop and release OTPs older than 20 s) | New generation gets `503` once the queue is full |
| Active-OTP cap | $N_{cap} = 200{,}000$ | Pool refuses reservation. Fail **closed** with `503` |
| Shed priority | **Verification is protected over generation.** A user who already holds a code must be able to use it. | Generation is shed first |
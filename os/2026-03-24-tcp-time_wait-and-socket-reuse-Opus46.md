Q: Consider this.
Assume I have a web server listening on the following address: (server IP, 80).

Now, a client connects to that server from the following address (clientIP, 9999).

After a response is sent to the client, server initiates the tcp connection closure.

So, if I get it right the full (connected) socket representing that connection on the server side - (serverIP, 80, clientIP, 9999) - is then put into the TIME_WAIT state (according to the tcp connection state transition scheme).

Now, the second question: do I get it right that while the server's socket is still in the TIME_WAIT, the same client still can initiate a new connection with the same clien port - i.e. from the (clientIP, 9999) address? And the server will be able to accept that connection request and use the same fully connected socket again (serverIP, 80, clientIP, 9999) as long as the sequence number in the client's SYN packet is different from the sequence number used in the previous connection?

A: You've got the first part exactly right. Since the server initiates the close, the server's end of the connection enters TIME_WAIT.

The second part is mostly right, but there's an important nuance.

**The original spec (RFC 793)** says no — the entire point of TIME_WAIT (lasting 2×MSL) is to prevent reuse of the same 4-tuple, so that stale segments from the old connection can't be misinterpreted as belonging to a new one. Under a strict reading, that 4-tuple is simply off-limits for the full TIME_WAIT duration.

**However, RFC 1122 (section 4.2.2.13) relaxed this.** It says a new incoming SYN *can* be accepted on a connection currently in TIME_WAIT, provided the new SYN's sequence number is **greater than** the last sequence number seen on the old connection — not merely *different*, but strictly greater. This is enough to guarantee that any old delayed segments won't be confused with the new connection's data, which was the whole concern TIME_WAIT was designed to address.

So the correction to your mental model is subtle but important: the condition isn't "different sequence number," it's "higher sequence number." In practice, modern TCP stacks (Linux in particular) do implement this relaxation, so yes, the same client can reconnect from `(clientIP, 9999)` and the server will accept it even while the old 4-tuple is in TIME_WAIT — as long as the SYN sequence number passes that "greater than" check.

One other small clarification: the server doesn't literally reuse the same socket object. The TIME_WAIT entry gets cleared and a brand-new connected socket is created for the same 4-tuple. From the networking perspective the result is what you described, but internally it's a replacement rather than a reuse.

Q: 
1. Above you said: "The original spec (RFC 793) says no — the entire point of TIME_WAIT (lasting 2×MSL) is to prevent reuse of the same 4-tuple, so that stale segments from the old connection can't be misinterpreted as belonging to a new one. Under a strict reading, that 4-tuple is simply off-limits for the full TIME_WAIT duration." - so what that would mean  for a client? Would it be supposed to alway use a "fresh" port? Or just do a retry with a different port if the "old" port doesn't work?
2. Above you wrote: "One other small clarification: the server doesn't literally reuse the same socket object. The TIME_WAIT entry gets cleared and a brand-new connected socket is created for the same 4-tuple. From the networking perspective the result is what you described, but internally it's a replacement rather than a reuse." Can you expand on that? - it's not clear what you meant here.

A: Great questions, both of them.

**On question 1:**

Under the strict RFC 793 reading, the burden falls on *both* sides, but in practice it mostly affects the client naturally. When a client calls `connect()`, the OS picks an ephemeral port for it. The OS's port selection logic is aware of which 4-tuples are currently in TIME_WAIT and will simply avoid picking an ephemeral port that would collide with an existing TIME_WAIT entry. So the client doesn't need to "retry" — it just gets a different ephemeral port automatically.

This is almost never a problem in practice because the ephemeral port range is large (Linux uses 32768–60999 by default, giving roughly 28,000 ports), and TIME_WAIT lasts only 60 seconds on most systems. You'd need to be opening and closing connections to the *same server* at an enormous rate to exhaust that space.

Where it *does* become a real problem is a specific scenario: the client explicitly binds to a fixed source port (using `bind()` before `connect()`), and it wants to reconnect to the same server. Under strict RFC 793, it would have to wait out the full TIME_WAIT. This is rare for normal clients but comes up in certain protocol designs or NAT situations.

So to directly answer your question — no, the client wasn't expected to manually pick fresh ports or retry. The OS handles it transparently by choosing a port that doesn't conflict.

**On question 2:**

Let me be more concrete. When the server closes the connection and the 4-tuple `(serverIP, 80, clientIP, 9999)` enters TIME_WAIT, what the kernel maintains is not a full-blown socket with send/receive buffers and all the associated machinery. It's a lightweight record — essentially just the 4-tuple plus enough state (the last sequence numbers, the timer) to fulfill TIME_WAIT's purpose: rejecting stale segments and properly responding if a retransmitted FIN arrives.

Now when a new SYN arrives from `(clientIP, 9999)` and passes the sequence number check (per RFC 1122), what happens is:

1. The kernel finds the TIME_WAIT entry matching that 4-tuple.
2. It tears down that TIME_WAIT entry (cancels the timer, frees the record).
3. The SYN goes through the normal accept path on the listening socket for port 80.
4. A completely new connected socket is created — fresh sequence numbers are negotiated, new buffers are allocated, the connection starts with a clean slate.

The reason I brought this up is that your original phrasing — "use the same fully connected socket again" — could suggest the old socket gets recycled or somehow transitions from TIME_WAIT back to ESTABLISHED. That doesn't happen. The old state is discarded and a new socket is born. The 4-tuple happens to be identical, but everything else is new.

Q: Your answer to the first question above a is total bullshit:
"but in practice it mostly affects the client naturally. When a client calls `connect()`, the OS picks an ephemeral port for it. The OS's port selection logic is aware of which 4-tuples are currently in TIME_WAIT and will simply avoid picking an ephemeral port that would collide with an existing TIME_WAIT entry. So the client doesn't need to "retry" — it just gets a different ephemeral port automatically."

As I had clearly pointed out that it's the server who closes the connection and whose socket goes into the TIME_WAIT state, not the client's one!

So you point on client avoiding the ports from sockets in the TIME_WAIT state is simply not applicable here. And so, my first question still stands: 
so what that would mean  for a client? Would it be supposed to alway use a "fresh" port? Or just do a retry with a different port if the "old" port doesn't work?

A: You're right, and I apologize for that — I described the mechanism for when the *client* is the one in TIME_WAIT, which isn't your scenario at all.

Let me answer properly. In your scenario, TIME_WAIT is on the **server**. The client's socket is fully closed — clean, no TIME_WAIT, nothing. The client's OS has zero knowledge of the server's TIME_WAIT state.

So under strict RFC 793, here's what would happen: the client calls `connect()`, the OS picks an ephemeral port, and it *might* pick 9999 again — the client's OS has no reason not to. The SYN arrives at the server, the server sees it matches a 4-tuple in TIME_WAIT, and it **silently drops the SYN**. From the client's perspective, the connection attempt just hangs and eventually times out.

To answer your actual question — the client wouldn't be "supposed" to do anything special. It has no way of knowing about the server's TIME_WAIT. It would simply experience a connection failure. On retry, the OS would most likely assign a different ephemeral port (since port selection algorithms typically cycle through the range rather than reusing the same port immediately), and that attempt would succeed because it's a different 4-tuple.

So it's not a designed mechanism — it's more of a "this rarely happens in practice because the ephemeral port space is large and sequential/random allocation makes an immediate collision unlikely, and if it does happen, a retry naturally fixes it." Which is admittedly not an elegant answer, and is part of why RFC 1122 later relaxed the rule to allow the new SYN through based on sequence number comparison.

Q: OK, now getting back to the RFC 1122 relaxation you mentioned above.

That relaxation seems to assume that the client side when initiating a connection to the server from the same address tuple - e.g. (clientIP, 9999) - would need to remember to always increase SYN value to be able to use that relaxation.

Question: is my assumption correct? And if so, do real OSes remember SYN values for connections with the identical origin address?

A: Your assumption is on the right track, but it turns out the client doesn't need to explicitly *remember* anything. It works out naturally through how initial sequence numbers (ISNs) are generated.

The original RFC 793 specified a global ISN generator based on a clock that increments roughly every 4 microseconds. Because it's a monotonically advancing clock, any new connection would naturally get a higher ISN than a previous one — no per-connection memory needed.

Modern OSes moved away from that global counter for security reasons (a predictable global ISN made TCP sequence prediction attacks trivial). Linux, for example, uses a hash-based ISN generator that takes the 4-tuple plus a secret key plus a time-based counter as inputs. The critical design choice is that the time-based component ensures that for the **same 4-tuple**, the generated ISN increases over time. So even though the algorithm is now cryptographically randomized across *different* 4-tuples, it still guarantees monotonic increase for a *repeated* 4-tuple — which is exactly what the RFC 1122 relaxation requires.

So to directly answer both parts: no, the client OS doesn't keep a table of "previous sequence numbers per 4-tuple." Instead, the ISN generation algorithm is deliberately constructed so that the property comes for free from the passage of time. As long as some time has elapsed (even a small amount), a new connection with the same 4-tuple will naturally produce a higher ISN. This isn't a coincidence — the ISN schemes were designed with exactly this interaction in mind.

Q: Your previous answer essentially means that under most circumstances a linux client will be always able to establish a new tcp connection to  the same Linux server (listening on (serverIP, 80)) using the same origin address - (clientIP, 9999) - even before the tuple from the previous one - (clientIP, 9999, serverIP, 80) - is still in the TIME_WAIT. Right?

A: Yes, that's correct. Given that Linux implements the RFC 1122 relaxation *and* its ISN generator guarantees a higher sequence number for the same 4-tuple, the server will accept the new SYN even while the old entry is still in TIME_WAIT.

The only practical caveat is that the client would need to actually *use* port 9999 again. If the client is just calling `connect()` without an explicit `bind()`, the OS picks an ephemeral port and will most likely give it a different one — not because of TIME_WAIT awareness, but simply because the allocation algorithm cycles through the range. So the scenario you're describing mostly applies when the client is deliberately binding to a fixed source port.

But yes, if the client does use port 9999 again — whether by explicit `bind()`, or by coincidence, or because of NAT reassignment — modern Linux on both sides will handle it gracefully. The server won't reject the SYN, the TIME_WAIT entry gets replaced, and a fresh connection is established. TIME_WAIT in this situation is essentially transparent.

Q: Can you verify your points you made above on how exactly SYN value is picked by a Linux client (i.e. factoring in a time portion which makes it monotonically growing) in the linux kernel source code?

A: Absolutely, let me dig into the actual source code.Good, I found the source file. Let me fetch the actual code.Here's what the source code confirms. The key is in two functions in `net/core/secure_seq.c`:

**The hash function — `secure_tcp_seq`:**

```c
u32 secure_tcp_seq(__be32 saddr, __be32 daddr,
                   __be16 sport, __be16 dport)
{
    u32 hash;

    net_secret_init();
    hash = siphash_3u32((__force u32)saddr, (__force u32)daddr,
                        (__force u32)sport << 16 | (__force u32)dport,
                        &net_secret);
    return seq_scale(hash);
}
```

**The time component — `seq_scale`:**

```c
static u32 seq_scale(u32 seq)
{
    /*
     * As close as possible to RFC 793, which
     * suggests using a 250 kHz clock.
     * Further reading shows this assumes 2 Mb/s networks.
     * For 10 Mb/s Ethernet, a 1 MHz clock is appropriate.
     * For 10 Gb/s Ethernet, a 1 GHz clock should be ok, but
     * we also need to limit the resolution so that the u32 seq
     * overlaps less than one time per MSL (2 minutes).
     * Choosing a clock of 64 ns period is OK. (period of 274 s)
     */
    return seq + (ktime_get_real_ns() >> 6);
}

```

So the formula is:

**ISN = siphash(saddr, daddr, sport, dport, net_secret) + (ktime_get_real_ns() >> 6)**

This confirms the mechanism, and actually makes it clearer than what I described earlier. For the **same 4-tuple**, the siphash portion is a **constant** — `net_secret` is generated once at boot and never changes, and the four address/port inputs are identical. So the only thing that varies between two connections on the same 4-tuple is the time component `ktime_get_real_ns() >> 6`, which is a 64-nanosecond resolution clock that strictly increases with time.

That's the guarantee: for repeated connections on the same 4-tuple, the ISN will always be higher than the previous one (as long as the u32 space hasn't fully wrapped, which the comment notes takes about 274 seconds — well beyond TIME_WAIT's 60-second duration on Linux).

Across *different* 4-tuples, the siphash produces completely unrelated values (because `net_secret` is unknown to outsiders), so an attacker can't predict one connection's ISN from observing another's. But for the same 4-tuple, the hash cancels out and you're left with a pure monotonic time ramp — which is exactly the property RFC 1122's relaxation needs.

Q: Can you embed the relevant links into your answer (like chatgpt does when answering)?

A: Sure, here's the answer again with embedded links:

---

Here's what the source code confirms. The key is in two functions in [`net/core/secure_seq.c`](https://github.com/torvalds/linux/blob/9c7db5004280767566e91a33445bf93aa479ef02/net/core/secure_seq.c):

**The hash function — `secure_tcp_seq`:**

```c
u32 secure_tcp_seq(__be32 saddr, __be32 daddr,
                   __be16 sport, __be16 dport)
{
    u32 hash;

    net_secret_init();
    hash = siphash_3u32((__force u32)saddr, (__force u32)daddr,
                        (__force u32)sport << 16 | (__force u32)dport,
                        &net_secret);
    return seq_scale(hash);
}
```

**The time component — `seq_scale`:**

```c
static u32 seq_scale(u32 seq)
{
    /*
     * As close as possible to RFC 793, which
     * suggests using a 250 kHz clock.
     * Further reading shows this assumes 2 Mb/s networks.
     * For 10 Mb/s Ethernet, a 1 MHz clock is appropriate.
     * For 10 Gb/s Ethernet, a 1 GHz clock should be ok, but
     * we also need to limit the resolution so that the u32 seq
     * overlaps less than one time per MSL (2 minutes).
     * Choosing a clock of 64 ns period is OK. (period of 274 s)
     */
    return seq + (ktime_get_real_ns() >> 6);
}
```

So the formula is:

**ISN = siphash(saddr, daddr, sport, dport, net_secret) + (ktime_get_real_ns() >> 6)**

This confirms the mechanism, and actually makes it clearer than what I described earlier. For the **same 4-tuple**, the [SipHash](https://en.wikipedia.org/wiki/SipHash) portion is a **constant** — `net_secret` is generated once at boot and never changes, and the four address/port inputs are identical. So the only thing that varies between two connections on the same 4-tuple is the time component `ktime_get_real_ns() >> 6`, which is a 64-nanosecond resolution clock that strictly increases with time.

That's the guarantee: for repeated connections on the same 4-tuple, the ISN will always be higher than the previous one (as long as the u32 space hasn't fully wrapped, which the comment notes takes about 274 seconds — well beyond TIME_WAIT's typical 60-second duration on Linux).

Across *different* 4-tuples, the siphash produces completely unrelated values (because `net_secret` is unknown to outsiders), so an attacker can't predict one connection's ISN from observing another's. This design follows [RFC 6528](https://datatracker.ietf.org/doc/html/rfc6528) ("Defending Against Sequence Number Attacks"), which updated the earlier [RFC 1948](https://datatracker.ietf.org/doc/html/rfc1948) recommendation. But for the same 4-tuple, the hash cancels out and you're left with a pure monotonic time ramp — which is exactly the property [RFC 1122](https://datatracker.ietf.org/doc/html/rfc1122#section-4.2.2.13)'s TIME_WAIT relaxation needs.

For additional context, [this analysis on LWN.net](https://lwn.net/Articles/455270/) covers the history of how Linux moved from a partial MD4-based generator to the current approach, and [this BitGuard writeup](https://bitguard.wordpress.com/2019/09/03/an-analysis-of-tcp-secure-sn-generation-in-linux-and-its-privacy-issues/) discusses the privacy implications of the time-based component being detectable across connections sharing the same 4-tuple.

Q: Now, on to the next part: how can an http client have the http server close the connection when the server is done sending the response?

A: The client sends the `Connection: close` header in its HTTP request.

In HTTP/1.1, connections are persistent (keep-alive) by default — meaning the server will leave the connection open after sending the response, waiting for further requests. By including `Connection: close`, the client tells the server "don't keep this connection open, close it once you've sent your response." The server acknowledges this by echoing `Connection: close` in the response headers, sends the response body, and then initiates the TCP close — which is exactly your original scenario where the server ends up in TIME_WAIT.

In HTTP/1.0 it was the opposite default — connections were non-persistent, so the server would close after every response unless the client explicitly asked for `Connection: keep-alive`.

It's worth noting that this is also why many high-traffic HTTP servers are configured to *avoid* being the active closer when possible. Every server-initiated close means a TIME_WAIT entry consuming kernel memory on the server. If you have thousands of clients all sending `Connection: close`, you accumulate thousands of TIME_WAIT entries on the server side. That's one of the practical motivations behind HTTP keep-alive and behind HTTP/2 multiplexing — they shift connection lifecycle management in ways that reduce this overhead.

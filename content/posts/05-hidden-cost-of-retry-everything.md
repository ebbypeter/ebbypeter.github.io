---
title: "The Hidden Cost of 'Retry Everything': How Naive Retry Logic Creates a Self-Inflicted DDoS"
draft: false
date: 2026-02-03
tags:
  - distributed-systems
  - resilience
  - retry-logic
  - azure
  - backoff
  - jitter
  - circuit-breaker
  - azure-sql
  - event-hubs
  - apim
  - application-gateway
  - azure-front-door
  - sre
categories:
  - Azure
  - Architecture
  - Site Reliability Engineering
summary: >
  Retries are load, not safety. Without exponential backoff and jitter, your retry logic doesn't protect against outages, it causes them. This post covers the mechanics of retry storms, five anti-patterns found in real production code, and what correct retry design actually looks like across layered Azure architectures.
---
It's 2:17am. Your phone lights up. PagerDuty. The order service is down, latency through the roof, error rates spiking, on-call engineer frantically pulling logs. The root cause will take another twenty minutes to surface: the service isn't being hit by external traffic. It's being hit by itself. Every instance, across every region, firing retries in perfect lockstep every two seconds, generating roughly 40 times the normal request volume against a payment API that's been struggling since a routine failover thirty minutes ago.

Nobody wrote bad code. Nobody skipped code review. The engineer who wrote the retry loop was doing exactly what they'd been taught: handle failures gracefully. Add retries. Be resilient.

The advice was right. The implementation was incomplete. And that gap, between "add retries" and "add retries correctly", is where production incidents live.

This post is not an argument against retries. Retries are necessary and correct. It's an argument for understanding what retries actually are: **load**. Every retry you fire is a request to a system that is already struggling. Without both exponential backoff *and* jitter, your retry mechanism is not a safety net, it is a latent self-destruct, armed and waiting for your first real outage.

---

## Part 1: Two Patterns You Need to Distinguish

The terms "thundering herd" and "retry storm" get used interchangeably in war stories and post-mortems. They're related, but they're not the same thing, and understanding the difference helps you target the right fix.

### The Thundering Herd

The thundering herd problem has its roots in OS scheduling. When many threads are blocked waiting on a lock or condition variable, and the event fires, the OS historically woke *all* of them, even though only one could proceed. The rest thrashed, burned CPU, and went back to sleep. Modern kernels like Linux 4.5+ fixed this with the `EPOLLEXCLUSIVE` flag. Distributed systems haven't been as lucky.

In a distributed context, a thundering herd happens when many clients simultaneously target the same resource. The synchronization usually comes from a shared event: a cache entry whose TTL expires at the same second for every server in the fleet; a JWT token that all thirty microservices refresh at once; a rolling deployment that brings fifty new instances online simultaneously, each starting with a cold cache and immediately hammering the database to warm it.

The core problem is a mismatch between how systems are designed and how traffic actually arrives. A database comfortably handling 500 queries per second is not designed to absorb 5,000 queries arriving in the same millisecond. Your capacity is measured in sustained rate. A herd doesn't respect that.

### The Retry Storm

A retry storm is a thundering herd with a specific cause: retry loops. It unfolds as a positive feedback loop:

1. Service experiences degradation, elevated latency, partial errors
2. Clients hit their timeout thresholds
3. Retry logic fires
4. Load on the already-degraded service spikes
5. More requests fail and time out
6. More retries fire
7. Return to step 4

A 10× overload that triggers retries quickly becomes 20×. The 20× triggers more retries, and the system reaches 40×. At that point, the service is spending all its resources accepting requests that it will fail and time out, instead of completing any. It has entered what researchers call a death spiral, and without external intervention, it cannot exit on its own.

The sequence isn't hypothetical. Microsoft's Azure Architecture documentation explicitly names this the "Retry Storm antipattern" and describes it as one of the most common self-inflicted outages in cloud systems.

### Cascading Retry Multiplication

There's a third failure mode that gets less attention: what happens when *multiple layers* of your stack each implement their own independent retry logic.

Consider a typical modern architecture: a frontend client talks to an API Management gateway, which forwards to a backend service, which queries a database. If each layer has a retry count of 3, a single user action that encounters a database hiccup can result in:

- Client retries 3× → 3 requests hit APIM
- APIM retries each 3× → 9 requests hit the backend
- Backend retries each 3× → **27 requests hit the database**

From a single user clicking a button, 27 database queries. Microsoft's Well-Architected Framework calls this out directly: when two layers each retry three times, the downstream service sees nine attempts per original request. Add a third layer and you're at twenty-seven.

The rule that follows from this is simple but frequently violated: **only one layer in a call chain should own retry responsibility**. Internal services should fail fast and propagate the error upstream. The layer with the most context, usually the outermost, makes the retry decision. Everything else gets out of the way.

### The Latency Tax on Healthy Requests

There is one more effect worth naming because it makes retry storms especially insidious: latency inflation that affects requests that haven't failed at all.

When a service is under retry pressure, its request queues grow. Threads and connection pool slots get consumed by in-flight requests waiting to be processed. Latency rises for *every* request, including fresh ones that have never encountered an error. As latency rises, more requests exceed their timeout thresholds and join the retry queue, adding more load, pushing latency higher still.

The mathematical trap is stark. If your client timeout is five seconds and the service is taking ten seconds to process a request, every request generates at least one retry. Your load has doubled before you've debugged anything. Timeouts continue. Retries compound. The system is spending resources on work it cannot complete.

---

## Part 2: Five Anti-Patterns That Kill You

These patterns are not exotic. They appear in codebases written by experienced engineers at well-run companies. Each one looks reasonable until your first real outage.

### Anti-Pattern 1: The Infinite Loop

Here is the most direct version of the problem, essentially verbatim from Azure's architecture documentation:

```csharp
public async Task<string> GetDataFromServer()
{
    while(true)
    {
        var result = await httpClient.GetAsync(url);
        if (result.IsSuccessStatusCode) break;
    }
    // ... process result
}
```

This looks like resilience. It will retry until it succeeds. In practice, when the downstream service is degraded, this loop runs forever, adding a sustained stream of requests to a system that is already struggling. There is no backoff. There is no ceiling. There is no circuit breaker. The service can never recover, because every moment of recovery capacity is immediately absorbed by this loop re-submitting.

Microsoft's Well-Architected Framework is unambiguous: **"Never implement an endless retry mechanism."**

The fix is always a finite retry count. When retries are exhausted, fail fast and surface the error. The caller, whether that's a user, a queue processor, or a dead-letter handler, can decide what to do next. Your retry loop cannot.

### Anti-Pattern 2: Retrying Before the Previous Attempt Has Completed

Sometimes called hedging or speculative retries, this pattern involves issuing a second request before the first has returned, running both concurrently in hopes that one will complete faster.

Hedging is a legitimate technique in very specific circumstances, reducing tail latency for idempotent read operations in low-load scenarios where the cost of an extra request is acceptable. But as a default retry behavior, it is dangerous.

If the service is experiencing elevated latency, every request is slow. Hedging every slow request doubles the number of in-flight requests. If the latency is caused by resource exhaustion, you have just made the cause worse. You've transformed a latency problem into a capacity problem. The second request doesn't reduce latency, it consumes the resources that were keeping latency at 10 seconds instead of 30.

If you use hedging intentionally, it must come with explicit in-flight request budgets. Without a ceiling on concurrent in-flight requests, hedging is just load amplification with extra steps.

### Anti-Pattern 3: Retrying Non-Transient Errors

Not every error is worth retrying. Retrying a `400 Bad Request` will never succeed. The server has already examined your request and rejected it as malformed or invalid. Retrying it again is not persistence, it is noise.

The classification that should drive retry decisions:

**Always retry**, transient errors that are likely to resolve:
- Network timeouts and connection resets
- `503 Service Unavailable` (especially with a `Retry-After` header)
- `429 Too Many Requests` (back off and respect the rate limit signal)

**Never retry**, permanent failures where retrying is wasteful:
- `400 Bad Request`, your request is invalid
- `401 Unauthorized`, your credentials are wrong or expired
- `403 Forbidden`, you don't have permission
- `404 Not Found`, the resource doesn't exist
- Business logic validation failures, retry won't change the outcome

**Conditionally retry**, errors that require judgment:
- `500 Internal Server Error`, may be a transient infrastructure event or a permanent bug. Log it, monitor the rate, retry cautiously with a low count.

For Azure services specifically, don't guess. The Azure SDK's `isTransient` flag exists for exactly this purpose. For Event Hubs AMQP clients, inspect the `AmqpException.IsTransient` property before retrying. A `QuotaExceededException` caused by exceeding the maximum number of consumer groups will never succeed on retry, no amount of retrying changes the quota.

### Anti-Pattern 4: Retrying Non-Idempotent Operations Without Safeguards

This is the anti-pattern with the most direct business consequences, and it's the one most often discovered via customer support ticket rather than monitoring.

Consider a payment processing call that times out. Your retry logic fires. The original request was received and processed, the timeout occurred in the response path, not the request path. You have just charged the customer twice.

Or an email notification that times out. Retry. The user receives the same notification twice. Or three times.

Idempotency is not a nice-to-have for retry logic on mutating operations, it is a hard prerequisite. Before enabling retries on any write operation, you must be able to answer the question: *what happens if this request is processed more than once?*

There are three ways to make mutations safe for retry:

**Idempotency keys**: Include a unique request identifier (e.g., a UUID generated once per user action) in the request. The server stores processed request IDs and deduplicates. If the same ID arrives twice, it returns the original result without reprocessing.

**Natural idempotency**: Design the operation so that running it twice produces the same result as running it once. An upsert (insert or update) is idempotent. A pure insert is not. A `PUT` that sets a value to a specific state is idempotent. A `POST` that increments a counter is not.

**Exactly-once queues**: For async workloads, push mutations into a queue with exactly-once delivery semantics and let the queue handle deduplication. The retry happens at the queue level, not the application level.

### Anti-Pattern 5: Synchronized Retries Across Replicas

This is the most subtle anti-pattern, and the one most engineers discover only in post-mortems.

Imagine a deployment with 500 service replicas. A downstream dependency experiences a brief outage at 14:23:00. All 500 replicas encounter failures. All 500 kick off their retry logic. All 500 wait exactly 2 seconds. At 14:23:02, all 500 simultaneously fire retry requests.

The downstream service, which has just started to recover, is immediately hit by 500 simultaneous requests, recreating the same overload that caused the outage. It fails again. All 500 replicas wait exactly 4 seconds. At 14:23:06, 500 simultaneous requests arrive again.

You can see the problem. Even exponential backoff doesn't help if all replicas are computing the same backoff values. A delay of `base × 2^attempt` is deterministic. With the same base and the same attempt count, every replica waits the same duration. The synchronized wave doesn't disappear, it reschedules.

This is why **jitter is not optional**. Jitter is not a minor improvement on top of exponential backoff. It is the mechanism that makes exponential backoff actually work. Without it, backoff is just a synchronized storm with a countdown timer.

---

## Part 3: Where Azure Catches You by Surprise

Azure's services have retry behaviors, some documented, some default, some automatic, that interact with your application-layer retries in ways that aren't always obvious. Each of the following can silently multiply the load your backends actually receive.

### Azure SQL: Transient Errors Are By Design

Azure SQL generates transient connection errors during routine maintenance operations: failovers, patching cycles, load-balancing reconfigurations. This is not a bug. It is expected, documented behavior, and it means retry logic for Azure SQL is not optional fault tolerance, it is a first-class architectural requirement.

Microsoft's documented guidance for retry intervals: wait **at least 5 seconds** before the first retry attempt. Shorter intervals risk overwhelming the service and prolonging the recovery. Maximum recommended retry delay: **60 seconds**, with exponential growth between.

The ADO.NET `SqlConnection` object has built-in connection retry parameters: `ConnectRetryCount` (default: 1, max: 255) and `ConnectRetryInterval` (default: 10 seconds, max: 60 seconds). The defaults are conservative. Know what they are before assuming your connection handling is correct.

One distinction matters operationally: a connection-phase failure (the `SqlConnection.Open()` call fails) and a query-phase failure (the connection is open but the query fails) require different retry approaches. For connection failures, retry the connection attempt after a delay. For query failures, establish a *fresh connection* first, then retry the query. Reusing a broken connection for a retry attempt is a common source of confusing behavior under load.

### Azure Event Hubs: The Invisible Throttle

The Event Hubs SDKs implement exponential backoff by default for retryable errors, but only if you're using the SDK. Custom AMQP clients don't get this behavior for free.

Kafka-based Event Hubs clients have a particularly subtle failure mode: throttling manifests as `throttle_time_ms` delays embedded in produce and fetch responses, not as explicit error codes. A Kafka client can be actively throttled without any visible error in its logs. The `throttle_time_ms` field in the response is the signal to watch.

When you scale Event Hubs consumers aggressively, you create a risk of backpressure not just on the hub itself, but on downstream services those consumers feed into. More throughput from the hub means more load on whatever your consumers are writing to.

The most dangerous default: Azure Functions retry policy supports a value of `-1` for infinite retries. Microsoft's own documentation explicitly warns that **indefinite retries should almost never be used with the Event Hub trigger binding**. If your Functions retry policy is set to `-1` and Event Hubs encounters a persistent error, your function will retry forever, blocking partition processing and accumulating lag.

Before retrying any Event Hubs error, inspect the `isTransient` flag on `AmqpException`. Errors like `QuotaExceededException`, triggered when you exceed the 20 consumer group limit, for example, are permanent configuration problems. No amount of retrying resolves them.

### Azure APIM: The Retry Multiplier in Your Gateway

APIM's `<retry>` policy lets you configure retry behavior at the gateway layer. This is a powerful feature with a specific trap: if both your client and APIM are retrying independently, the backend sees the product of both retry counts, not the sum.

Client retries 3×, APIM retries 3× each time, the backend receives 9 requests per original user action. Add another retry layer (a backend service that retries its own database calls) and you're at 27.

The principle: **pick one layer to own retry responsibility.** For external-facing APIs, APIM is typically the right owner, it has visibility into backend health and can implement circuit breaking at the gateway level. When APIM owns retries, internal services should be configured to fail fast. They propagate errors; they don't absorb them.

When APIM is retrying, it should also be responsible for respecting `Retry-After` headers from backends and for honoring circuit breaker state. The retry policy in APIM is powerful enough to implement these correctly; use it.

### Azure App Gateway and Front Door: Silent Gateway-Layer Retries

This is the most frequently missed retry source in Azure architectures.

**Application Gateway v2**: when a backend member times out, App Gateway doesn't immediately return a `504`. It automatically retries the request against a *second backend pool member*. If your application also retries after receiving the `504`, you have silently doubled the load on your backend pool, with no visibility into the gateway-level retry in your application logs.

**Azure Front Door**: explicitly recommends setting a forwarding timeout that is aligned to your origin's actual response time. If you leave the timeout mismatched, Front Door holds connections open longer than necessary, consuming capacity on both ends, and may trigger retry behavior that compounds with application-layer retries.

Front Door's WAF includes rate-limiting rules that are explicitly documented as a defense against retry storms. These are not optional production hardening, they are a named line of defense against this exact failure mode. If you're running Front Door without WAF rate limiting configured, you are one client retry storm away from a capacity incident.

The key question to ask about every layer in your architecture: *who is retrying, and how many times?* Audit your entire call chain before you're reading a post-mortem.

---

## Part 4: How to Actually Design This Correctly

Safe retry logic is not a single setting. It is a layered system where each component handles a specific aspect of the problem.

### Exponential Backoff: The Foundation

The formula: `delay = min(cap, base × 2^attempt)`

Attempt 1 waits 1 second. Attempt 2 waits 2 seconds. Attempt 3 waits 4 seconds. Each subsequent attempt doubles, until the delay is capped at a maximum. For Azure SQL, Microsoft recommends a base of 5 seconds and a cap of 60 seconds, a conservative baseline that reflects real measured recovery times.

The purpose of backoff is to give the recovering service time to stabilize between retry waves. Without it, each retry attempt arrives immediately after the previous failure, giving the service no recovery window at all.

### Jitter: The Mechanism That Makes Backoff Work

Let's be precise about this, because it's frequently treated as a minor optimization when it's actually the core of the solution.

Without jitter, exponential backoff is deterministic. If 500 replicas all computed their first retry delay as 2 seconds, they all wait exactly 2 seconds and fire simultaneously. The retry wave is just as synchronized as if there were no backoff at all, it's just 2 seconds later. Jitter breaks the synchronization by making each replica compute a slightly different delay.

There are three common jitter strategies:

**Full jitter** (recommended for most scenarios):
```
delay = random(0, min(cap, base × 2^attempt))
```
Maximum spread between retry attempts. Most effective at desynchronizing a fleet of clients.

**Equal jitter** (when a minimum wait is required):
```
delay = min(cap, base × 2^attempt) / 2 + random(0, min(cap, base × 2^attempt) / 2)
```
Guarantees a minimum wait while still adding randomness.

**Decorrelated jitter** (AWS's recommendation for some scenarios):
```
delay = random(base, prev_delay × 3)
```
Breaks correlation with the previous attempt's timing rather than with the attempt count.

Full jitter typically performs best in practice for cloud service scenarios. The math is simple. Adding `random(0, cap)` to every retry interval in your system, and to every TTL, costs a few minutes to implement and prevents the most common trigger of synchronized storms.

### Circuit Breakers: Stopping the Death Spiral

A circuit breaker is a state machine that wraps calls to a downstream service. It tracks the failure rate. When failures exceed a threshold, the circuit **opens**: all subsequent requests fail immediately, without hitting the downstream service at all. After a cooldown period, the circuit enters **half-open** state and allows a single probe request through. If the probe succeeds, the circuit closes and normal operation resumes. If the probe fails, the circuit opens again and the cooldown restarts.

The critical property is what happens during the open state: requests fail *fast*. The downstream service receives no traffic. This gives it the breathing room to actually recover, something it cannot do while your retry logic is continuously hammering it.

```
CLOSED (normal) → [failure threshold exceeded] → OPEN (failing fast)
OPEN → [cooldown elapsed] → HALF-OPEN (single probe)
HALF-OPEN → [probe succeeds] → CLOSED
HALF-OPEN → [probe fails] → OPEN
```

In the Azure ecosystem, **Polly** is the standard .NET implementation. It supports circuit breakers, retry policies, timeout policies, and bulkheads, and it composes them into a resilience pipeline. For Java, **resilience4j** is the modern choice (Hystrix is largely in maintenance mode). Both libraries implement the patterns correctly and are well-tested at production scale. Do not write your own circuit breaker from scratch.

### Retry Limits: Per-Request and Fleet-Wide

Per-request limits are the baseline: set a finite maximum retry count, and when it's exhausted, fail fast. For Event Hubs specifically, never set this to `-1`.

But per-request limits alone are not sufficient at fleet scale. Consider 500 replicas each configured with a maximum of 3 retries. On a shared failure event, each replica fires 3 retries, that is 1,500 simultaneous retry attempts against the recovering service. Each replica stayed within its limit. The fleet did not.

**Retry budgets** address this by applying a global limit across all instances. Rather than each replica independently deciding to retry, the fleet collectively budgets a maximum retry rate, for example, no more than 1,000 retries per minute across all instances. Requests that would exceed the budget are rejected at the source rather than allowed to pile onto the downstream service.

Implementing retry budgets requires coordination infrastructure, a rate limiter with a shared counter, often backed by Redis or a similar fast store. The operational complexity is real. But for high-replica-count services hitting shared dependencies, it is the difference between a retry policy that helps and one that amplifies.

### Additional Safeguards

**Honor `Retry-After` headers.** When a server returns `503` or `429` with a `Retry-After` value, that number is the server's own estimate of when it will be ready to accept requests again. Retrying before that time is not aggressive resilience, it is ignoring the clearest signal the server can give you. Read the header. Respect it.

**Idempotency before retry.** For any mutating operation, confirm idempotency before enabling retries. Use idempotency keys for operations that modify external state. Design upserts rather than inserts where possible. Treat idempotency as a precondition, not an afterthought.

**Dead-letter queues for async workloads.** When retries on a background or queue-based operation are exhausted, the message should move to a dead-letter queue, not be silently dropped, and not trigger an infinite retry loop. Dead-letter queues preserve the failed work for human review and controlled reprocessing.

**Use official SDKs.** The Azure SDKs implement correct retry behavior by default. They respect `Retry-After` headers, classify errors by transience, and use jittered exponential backoff out of the box. Custom AMQP clients and raw HTTP clients don't get any of this. If you're writing your own retry logic from scratch, you are starting from behind. Use Polly for .NET, `retry` for Node.js, or the equivalent for your stack. Hand-rolled retry loops are where every anti-pattern in this post comes from.

---

## Conclusion: Retries Are Load

The mental model shift that makes everything else follow: retries are not a safety mechanism that operates outside your normal load budget. They are requests. They consume the same resources as any other request. When you fire a retry at a struggling service, you are adding load to a casualty.

The failure mode of naïve retry logic is invisible during normal operations. When failures are rare, retries work fine. The service recovers, the retry succeeds, and nobody notices the gap in the design. The vulnerability surfaces only under stress, exactly when you need reliability most. This is why retry logic must be load-tested under adversarial conditions: inject failures, simulate partial degradation, observe how your retry behavior changes system load as things break.

The question is not "how do we retry?" The question is "how do we make sure our retry behavior doesn't prevent recovery?"

Seven principles to carry forward:

1. **Retries need backoff and jitter. Both. Neither alone is sufficient.**
2. **Only one layer in a call chain should own retry logic. Audit your stack.**
3. **Classify errors before retrying. Never retry permanent failures.**
4. **Ensure idempotency before retrying mutating operations.**
5. **Circuit breakers stop spirals. Retry budgets prevent fleet-level amplification.**
6. **Honor `Retry-After`. The service is telling you when to try again.**
7. **Test your retry behavior under load, not just in isolation.**

The engineers who wrote those infinite retry loops weren't being careless. They were doing what they'd been taught. The half-truth, "add retries", is widespread and well-intentioned. This post is an argument for the complete truth: add retries, with exponential backoff, with jitter, with circuit breakers, with finite limits, with idempotency, with respect for what the downstream service is telling you.

Resilience is not optimism. It is load management.

---

## Quick Reference

### Retry Decision Tree

```
Request fails
└── Is the error transient? (timeout, 503+Retry-After, 429)
    ├── No  → Fail immediately. Do not retry.
    └── Yes → Is the circuit breaker open?
              ├── Yes → Fail immediately. Wait for half-open probe.
              └── No  → Have we exceeded our retry budget?
                        ├── Yes → Fail. Route to dead-letter queue if async.
                        └── No  → Compute: exponential backoff + full jitter
                                  └── Is the operation idempotent?
                                      ├── No  → Attach idempotency key, then retry.
                                      └── Yes → Retry.
```

### Jitter Strategy Reference

| Strategy | Formula | Best For |
|---|---|---|
| Full jitter | `random(0, min(cap, base × 2ⁿ))` | Most cloud scenarios; maximum desynchronization |
| Equal jitter | `cap/2 + random(0, cap/2)` | When a guaranteed minimum wait is required |
| Decorrelated | `random(base, prev_delay × 3)` | Breaking correlation with previous attempt timing |

### Azure Service Retry Defaults

| Service | Default Behavior | Key Risk |
|---|---|---|
| Azure SQL (ADO.NET) | `ConnectRetryCount=1`, `ConnectRetryInterval=10s` | Likely under-configured for your workload |
| Event Hubs SDK | Exponential backoff built in | Custom AMQP clients get none of this |
| Azure Functions + Event Hubs | Configurable; `-1` is valid config | `-1` (infinite) should never be used in production |
| Application Gateway v2 | Auto-retries to second backend on timeout | Silent 2× multiplier on backend load |
| APIM `<retry>` policy | Configurable per policy | Compounds client-layer retries multiplicatively |
| Azure Front Door | Forwarding timeout must be configured explicitly | Mismatched timeouts cause unnecessary holds and retries |

---

## References

- [Azure Architecture: Retry Storm Antipattern](https://learn.microsoft.com/azure/architecture/antipatterns/retry-storm/)
- [Azure Well-Architected Framework: Recommendations for Handling Transient Faults](https://learn.microsoft.com/azure/well-architected/design-guides/handle-transient-faults)
- [Azure Architecture: Retry Pattern](https://learn.microsoft.com/azure/architecture/patterns/retry)
- [Azure Architecture: Circuit Breaker Pattern](https://learn.microsoft.com/azure/architecture/patterns/circuit-breaker)
- [Azure SQL Database: Troubleshoot Transient Connection Errors](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-connectivity-issues)
- [Azure Event Hubs: Resilient Design with Functions](https://learn.microsoft.com/azure/architecture/serverless/event-hubs-functions/resilient-design)
- [AWS Architecture Blog: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Wikipedia: Thundering Herd Problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)

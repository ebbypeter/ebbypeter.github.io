---
title: "Your DR Plan Has Never Been Tested"
draft: false
date: 2026-02-10
tags:
 - azure
 - disaster-recovery
 - sre
 - resilience
 - infrastructure-as-code
categories:
 - Azure
 - Architecture
 - Site Reliability Engineering
summary: "Most Azure DR tests confirm the secondary came up. They don't confirm your RTO is real, your RPO commitment holds under load, or that failback won't silently destroy the incident window. Here's how to test DR honestly, with exit criteria that actually prove the plan works."
---

*Most DR tests prove the secondary came up. Not that your RTO is real, your data is intact, or your failback won't destroy the incident window. Let's fix that.*

---

There's a version of this story that plays out in a lot of engineering organizations,
and it goes roughly like this.

The team schedules a DR test on a Saturday morning. They initiate a failover to the
secondary region. The secondary comes up. Traffic Manager switches. Someone checks
the portal, sees green, and marks the test complete. The compliance ticket closes.
The DR plan is valid for another year.

What nobody checked: whether users were actually being served at minute 91 of a
two-hour RTO window, or whether the application was still warming up its connection
pools. Whether the monitoring dashboards were even configured in the DR region —
because they weren't, which is why no alerts fired. Whether the database replication
lag had been running at 45 seconds under Friday's write load, meaning a forced
failover would have silently dropped nearly a minute of transactions. And whether
anyone had thought about what happens when the primary region comes back and the
team needs to fail back with 40 minutes of writes sitting on the secondary that the
old primary has never seen.

The test passed. The plan is fiction.

Passing a DR test and having a DR plan that works are two different things. The gap
between them is almost always the same thing: nobody defined what "passing" meant
before they started.

---

## What You're Actually Measuring (vs. What You Think You Are)

Most DR tests measure infrastructure health and elapsed time. The secondary region
came up, check. Time from failover initiation to infrastructure-reports-healthy was
under the RTO threshold, check. Test passed.

These are real measurements. They're just not the ones that matter to users.

What actually matters is user-visible RTO: the window between the moment the first
user request failed and the moment the first user request *succeeded against the
secondary*. That's not the same as "when we issued the failover command", it's later.
Often significantly later, for reasons we'll get to. And it's not the same as "when
infrastructure reported healthy", because infrastructure reporting healthy and
applications serving correct responses are two different events.

Similarly for RPO. The number in your DR plan is probably something like "< 5 minutes"
or "< 15 minutes." What that actually means is: if a disaster happened right now, the
maximum amount of data you'd lose is bounded by that window. But unless someone has
measured the replication lag under realistic write load and confirmed it stays within
that bound, the number is aspirational. It's what the architecture is *designed* to
achieve, not what it actually achieves on a busy Thursday afternoon.

And then there's application correctness, which most teams don't test at all. The
homepage returning 200 is not a valid DR test. Can a user authenticate? Can they
complete a write transaction and get a consistent response? Are background jobs
running, or have they silently stopped because the job scheduler is pointing at a
queue in the primary region that's now unreachable? These questions have answers.
Most DR tests never ask them.

There's another gap that tends to go unnoticed until an actual incident: monitoring
coverage in the DR region. Alert silence during a DR test is not evidence that nothing
is wrong. It often means nothing is watching. If your alert rules and dashboards live
in Azure Monitor workspaces that weren't deployed to the secondary region, or if your
Application Insights instance is still pointing at resources in the primary, your
observability stack goes dark the moment you need it most. This is one of those
configuration drift problems that's invisible until it isn't.

---

## The Timing Chain Azure Doesn't Advertise

The Azure documentation is accurate. What it doesn't do is assemble a complete picture
of what actually happens between "the primary failed" and "users are being served by
the secondary." Understanding that chain is what lets you build a test that measures
the right things.

### Traffic Manager

Traffic Manager works at the DNS layer. It does not proxy traffic, once a client
has resolved the DNS name to an IP address, Traffic Manager is out of the loop. That
distinction matters enormously for RTO calculations.

When your primary region fails, Traffic Manager detects the failure through health
probes, updates its DNS response to point to the secondary, and from that point forward,
new DNS lookups return the secondary's endpoint. But clients that have already resolved
the DNS name, which is most of them, continue connecting to the primary IP until
their cached DNS entry expires.

The default TTL for Traffic Manager is 300 seconds. That's five minutes of clients
continuing to hammer a dead endpoint after Traffic Manager has already switched. Add
to that the detection time: with default probe settings (30-second interval, 3
consecutive failures required), Traffic Manager doesn't even know the primary is down
until at least 90 seconds have passed. The full formula is:

```
Failover window = TTL + (retry count × probe interval)
        = 300 + (3 × 30)
        = 390 seconds (~6.5 minutes)
```

That's the minimum gap between primary failure and new clients receiving the secondary
endpoint, at default settings, before any application warm-up, before connection pool
initialization, before anything application-level happens.

You can tune this aggressively: TTL of 10 seconds, probe interval of 10 seconds with
3 retries gets you down to roughly 40 seconds. But lower TTL means more DNS queries
(Traffic Manager pricing is query-based), and lower probe intervals mean more probe
traffic against your origin. The right values depend on your RTO target and what your
origin can absorb.

There's a subtler problem with Traffic Manager health probes that catches teams off
guard. Traffic Manager can only evaluate what the health probe endpoint returns. If
your probe endpoint is a simple HTTP check against the web process, Traffic Manager
has no idea whether the database finished promoting, whether the connection pool is
healthy, or whether the app is returning 500s to every actual user request. A health
probe that checks "is the web process running" will return 200 while users are getting
database connection errors downstream. Your probe endpoint needs to check the
dependencies that actually matter, database reachability, cache availability, any
downstream services required for the application to function.

### Front Door

Front Door is a different story because it actually proxies traffic, it's a reverse
proxy running at Microsoft's global POPs. This means the cutover is cleaner once
detection happens, because you're not dealing with client-side DNS caching in the
same way.

But detection still takes time. With default settings, 30-second probe interval, 3
consecutive failures required, Front Door won't mark an origin unhealthy for up to
90 seconds. And there's a documented behavior that surprises most people who haven't
tested it explicitly: if every origin in an origin group fails its health checks, Front
Door doesn't drop traffic. It routes round-robin across all of them. Which means
during a full-region outage, before the secondary's health checks have confirmed it's
healthy, users may be getting errors from both endpoints simultaneously. Worth knowing.
Worth testing.

### Azure SQL Auto-Failover Groups

SQL cross-region replication is always asynchronous. There's no synchronous cross-region
option for Azure SQL Database. This means there is always a replication lag, a window
of committed transactions on the primary that haven't yet been hardened to the secondary.
Under normal conditions, Microsoft targets less than 5 seconds. Under high write load
with a secondary that's underpowered relative to the primary, that lag can grow
considerably and will keep growing until the secondary falls so far behind it becomes
unavailable.

Here's the part that most teams get wrong: there are two kinds of failover, and they
have completely different RPO characteristics.

**Planned failover**, the kind you can initiate manually when there's no active outage
— waits for the secondary to fully synchronize before promoting it to primary. Zero
data loss, essentially guaranteed. RTO is typically under 60 seconds. This is the
failover most teams test.

**Forced failover**, what happens when the primary region is actually inaccessible —
promotes the secondary immediately, regardless of replication state. Data loss is
possible and the extent of it is exactly whatever was in the replication lag at the
time. This is what happens in a real disaster.

If you've only ever tested planned failover, you've never tested the scenario that
actually matters. The replication lag at the moment of a real outage is the variable
nobody has measured.

The key DMV here is `sys.dm_geo_replication_link_status`, specifically the
`replication_lag_sec` column. This tells you, right now, how many seconds of
transactions on the primary haven't been hardened to the secondary. If that number is
running at 8 seconds during normal operations, your real-world RPO floor is 8 seconds
— not the < 5 seconds in the architecture diagram. And as of 2025, this is also
surfaced as a native Azure Monitor metric with one-minute granularity and 93-day
retention, so you can set alerts and see trends over time rather than just spot-checking.

There's one more thing worth calling out: `GracePeriodWithDataLossHours`. This
parameter controls how long Azure SQL waits before auto-triggering a forced failover
when the primary is unreachable. The default is 1 hour. That means if your primary
goes down and you haven't manually intervened, your secondary won't automatically
promote for an hour. For a lot of workloads, that's a silent assumption baked into
the architecture that nobody has thought through.

### Azure Storage GRS and RA-GRS

Storage replication is also asynchronous, and the RPO picture is similar: typically
less than 15 minutes, but with no contractual SLA on replication delay, unless you've
opted into Geo Priority Replication for Block Blobs, which went GA in late 2025 and
provides an actual 15-minute SLA covering 99% of billing months.

The number that matters is `Last Sync Time`, available in the portal under
Redundancy → Prepare for failover, or via
`az storage account show --query "geoReplicationStats"`. This tells you the most
recent point at which all primary writes are guaranteed to be present on the secondary.
Anything written after that timestamp is at risk in a failover.

What frequently gets left out of RTO calculations: initiating a storage account failover
takes 30 to 60 minutes, depending on account size and number of objects. After that
failover completes, the account is LRS in the new region, not GRS. You have to
explicitly re-enable geo-redundancy and wait for replication to re-establish before
you have any redundancy again. That recovery-from-recovery window is often months long
in practice because teams forget to re-enable it.

---

## How to Test Without Taking Production Down

Before getting into testing approaches, it's worth naming an assumption that makes all of them moot: "we'll IaC it when we need it." This is more common than it should be, a hub network in the secondary region, no app stack deployed, an assumption that Terraform or Bicep will save the day when the primary goes down. The theory is sound. The practice fails because nobody has ever run that IaC under incident conditions, with a degraded team, against a primary region they can't access to verify state, with pipeline credentials that may or may not be stored somewhere the DR region can reach. The first time that IaC runs for real is during the disaster itself, which is, definitionally, the worst possible moment to discover that a module has a hardcoded region variable, a managed identity doesn't have the right permissions in the secondary, or the pipeline agent pool lives in the region that just went down. "We can deploy it when we need it" is not a DR strategy. It's a hypothesis that has never been tested.

Testing DR properly doesn't require risking a production outage. There are four
approaches that actually move the needle, and importantly, they build on each other.
Start here, in this order.

**Health probe poisoning** is the fastest way to measure your actual failover detection
gap without touching anything in the primary. Configure your health probe endpoint to
return a non-200 response, through a feature flag, a temporary config change, whatever
you have available, and watch what happens. Specifically: how long does it take Traffic
Manager or Front Door to update its routing? And how long after that do clients actually
receive the secondary endpoint? The delta between those two events is your DNS TTL in
practice, which may not match what's configured, because intermediate resolvers may
be caching aggressively. Run a synthetic monitor from Application Insights Availability
Tests pointed directly at your DR FQDN, not your primary, so you can observe recovery
from a real client perspective. This test has zero risk to production traffic and tells
you your real RTO floor before you've touched a database or a compute layer.

**Replication lag load testing** answers the question most teams have never asked: does
your replication lag stay within your RPO commitment under realistic write load?
Drive your primary database and storage at production-representative write volumes for
a sustained period, not a trivial benchmark, but something that approximates a busy
business day. Watch `replication_lag_sec` and `geoReplicationStats.lastSyncTime`
continuously during that window. If the lag stays at 2 seconds, your RPO commitment is
credible. If it drifts to 45 seconds and keeps climbing, you have a secondary that's
underpowered for the write profile it would need to absorb, and your RPO commitment
is a number someone invented. This test runs against production during off-peak hours
with zero impact on availability.

**Dependency failover drills** let you test component resilience in isolation. Initiate
a failover for a single SQL Failover Group, or a single storage account, without doing
anything at the regional level. Watch the application layer respond. Does it reconnect
cleanly, or does it throw a cascade of errors before retry logic kicks in? Do any
services hard-code the primary region endpoint and fail completely? Is the connection
pool recovery time acceptable? This drill tends to surface the configuration drift
issues, hardcoded endpoints, missing secrets in the secondary Key Vault, RBAC
assignments that were made to production but never mirrored, before they surface
during a real event.

**Full failover with traffic isolation** is the real test, done safely. Use Front
Door weighted routing to shift 5 to 10% of traffic, or a specific internal user
cohort, if you can route by identity, to the secondary region while the majority
stays on primary. Validate application behavior under real request shapes. Measure
error rates and latency during the transition. This isn't a simulation. It's real
traffic hitting the real DR stack, just not all of it. If something is broken in the
secondary region, you find out with 10% of traffic, not 100%.

---

## Defining "Pass" Before You Start

This is the part that determines whether you're running a DR test or a DR exercise.
The distinction is simple: a test has pass/fail criteria defined before you start. An
exercise is a walk-through that produces observations.

Both have value. But only one tells you whether your DR plan works.

Before the test starts, write down three things.

**The user-visible RTO target.** Define T₀ as the moment the first user request fails
— not the moment someone noticed the problem, not the moment the failover was
initiated. Define what "serving" means: not infrastructure healthy, but a specific
transaction completing successfully against the secondary. State it plainly: "At
T₀ + 45 minutes, a user must be able to authenticate, submit an order, and receive
a confirmation. If they cannot, the test has failed."

**The actual data loss window.** Before initiating failover, write a sentinel
transaction, a specific, identifiable write with a known timestamp. After failover
completes, check whether that transaction is visible on the secondary. If it is,
record how far it was from the failover initiation, that's the replication lag at
time of failure. If it isn't visible, you've lost at least that transaction. Compare
the result against your committed RPO. If the loss exceeds the commitment, the test
failed.

**The application smoke test.** Define a suite, authentication, a read, a write,
a background job check, that must pass against the secondary before the test is
complete. Not "the homepage loaded." A real business transaction. Run this suite
from outside Azure, from a network path that doesn't have access to the primary,
so you're validating what an actual user would experience.

When the test runs and you measure these three things, one of two things happens.
Either the numbers match the commitments, in which case you have real evidence that
your DR plan works. Or they don't, in which case you have specific data about where
the gap is, and you can fix it.

What you want to avoid is what happens without exit criteria: the test runs,
infrastructure comes up green, nobody measures anything, and the ticket closes. That's
not a test. That's a ceremony.

---

## Failback: The Test Nobody Runs

Failover gets all the attention. Failback is where the data loss actually happens.

The scenario: your primary region fails at 2pm. You execute failover. The secondary
becomes primary. Your team operates on the secondary from 2pm to 4pm, two hours of
transactions, updates, customer activity. The primary region recovers at 4pm. Your
runbook says "fail back to primary." Someone follows the runbook.

If that runbook doesn't have an explicit step that says "check replication lag / Last
Sync Time before executing failback, and do not proceed until secondary-to-primary
replication covers the full 2pm–4pm window," then you are about to permanently lose
two hours of production data. The old primary comes back with stale state, you cut
over to it, and everything written between 2pm and 4pm is gone.

Azure's own documentation explicitly calls this out for storage: check Last Sync Time
before failing back. Teams still skip it, usually because they're under pressure to
restore the primary and the urgency overwhelms the procedure.

For SQL, the old primary reconnects as a secondary after a failover and begins redo
replication from the current primary. The replication lag clock is running. You need
to watch `replication_lag_sec` and wait until it reaches zero, or at least until
it's within your RPO tolerance, before initiating failback. Under heavy write load,
this can take longer than you expect.

For storage, you need `geoReplicationStats.lastSyncTime` on the new primary (which
was your secondary) to advance past the point at which you wrote your last significant
transaction during the incident window. If it hasn't, failing back means losing
everything written after that sync point.

The specific test to run: initiate a full failover, then deliberately generate 30
minutes of realistic write activity against the new primary, simulating what your
application would actually do during a real incident. Then attempt to execute your
failback procedure. This single test reveals more about your DR plan's real-world
viability than any number of clean failover drills, because it answers the questions
that matter: does your runbook have the right lag-check steps? Does your team know to
wait? Does your observability correctly surface the re-replication state so the
decision to fail back is made with real data rather than impatience?

Failback is not the reverse of failover. It's a separate procedure with separate
failure modes, separate timing considerations, and separate exit criteria. It needs
its own test, scheduled separately, with its own pass/fail definition.

---

## A DR Plan Not Tested Is a DR Plan That Fails

The tools are there. Azure Monitor surfaces `replication_lag_sec` as a native metric
with one-minute granularity. `Last Sync Time` is a CLI query away. Front Door health
probe logs tell you exactly which POP saw which origin fail and when. Traffic Manager
exposes the full failover formula and lets you tune every variable in it.

The gap isn't tooling. It's not even time or budget, really. It's the willingness to
define what failure looks like before you start, and then to actually call the test
failed when the numbers don't match.

So before the next DR test: write three numbers on a whiteboard. The user-visible RTO.
The actual data loss window your architecture can sustain under realistic load. The
time to a safe, confirmed failback with no data lost from the incident window. Run
the test. Measure those numbers. Put the results next to the commitments.

If they match, great. You have evidence. Your DR plan is real.

If they don't, now you have something more valuable than a passing test: you have
the specific information needed to make the plan actually work.

That's a DR test. Everything else is a drill.

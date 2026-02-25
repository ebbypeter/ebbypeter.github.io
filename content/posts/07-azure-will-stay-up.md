---
title: "Azure Will Stay Up. Your System Is a Different Story."
draft: false
date: 2026-02-17
tags:
  - azure
  - high-availability
  - reliability
  - distributed-systems
  - sre
  - architecture
  - cloud
categories:
  - Azure
  - Architecture
summary: "Azure's infrastructure is genuinely reliable. That's exactly the problem. The more stable the platform, the easier it is to mistake platform health for system health, and that gap is where the expensive outages live. Availability is an architectural choice, not a SKU."
---

*MThe Azure status page is green. Your users are getting errors. Somewhere in the gap between those two facts is the outage nobody planned for.*

---
The migration is done. The post-project retrospective is done. Leadership has signed off on the cloud transformation, the slide deck declared victory, and somewhere in a wiki that nobody updates anymore there's a note confirming that zone redundancy is enabled and geo-replication is configured. The team moves on to the next thing.

Then, six months later, users start getting errors. The on-call engineer pulls up Azure Monitor. All the infrastructure metrics look fine. The Azure status page is green. And yet something is clearly broken, has been broken for twenty minutes, and nobody quite knows why.

This scenario plays out constantly, not because Azure is unreliable, but because of a confusion that's easy to acquire and expensive to correct: the belief that *infrastructure availability* and *system availability* are the same thing. They're not. Azure's infrastructure is genuinely, impressively reliable. Whether your *system* is available is a separate question entirely, and the answer depends almost entirely on decisions your team made, or didn't make, in the design of your application.

This isn't an Azure criticism. The argument here is actually the opposite: the platform is good enough that it's easy to stop thinking carefully about the part you still own.


## The Line Nobody Reads

Microsoft is fairly explicit about this. Azure's reliability documentation describes a three-tier shared responsibility model. The first tier, core platform reliability, is entirely Microsoft's problem. The redundant hardware, the fault-tolerant networks, the datacenter power systems and cooling and physical security: all Microsoft. The third tier, application design, is entirely yours. Microsoft provides the Azure Well-Architected Framework as guidance, but nobody is auditing your architecture, nobody is reviewing your code, and nobody is checking whether you've actually applied any of it.

The middle tier is where things get interesting. Reliability-enhancing capabilities, availability zones, geo-replication, backup configuration, service tier selection, Microsoft builds these and makes them available. But selecting and enabling them is your responsibility. And more importantly, *configuring your application to actually benefit from them* is your responsibility.

Here's the sentence buried in the official documentation that matters most: *"You are responsible for understanding and meeting [SLA] conditions; Microsoft does not monitor or enforce your eligibility."*

Read that again. You can be running on Azure, have zone-redundant SQL Database deployed, have the right subscription tier, and still be ineligible for the SLA you're counting on because of how your application is configured. Azure isn't going to tell you. The portal isn't going to flag it. You'll find out when something fails.

The terminology that helps here is the distinction between *infrastructure availability* and *system availability*. Infrastructure availability is what Azure sells and what Azure's SLAs describe: the probability that the platform service is operational and reachable. System availability is what your users experience: the probability that they can actually complete whatever they came to do. These two things overlap significantly, but they're not the same measurement, and a gap between them is where most real-world outages live.


## The Checkmark Problem

Most teams that have "done HA" have worked through some version of the same checklist. Zone redundancy: enabled. Geo-replication: configured. Managed services instead of VMs where possible. Health probes: yes, those exist. Premium tier: yes, that's what we're on. Check, check, check.

The problem isn't that these things are wrong to do. They're the right things to do. The problem is what teams believe they've accomplished by doing them.

Take zone redundancy. Enabling it means Azure distributes your resources across physically separate datacenters within a region. For PaaS services, Microsoft handles failover automatically. That's real, and it's genuinely useful. But when Azure fails over your zone-redundant SQL Database, your application sees a brief connectivity disruption, typically under 60 seconds. If your application has no retry logic for transient faults, that 60-second platform event becomes a 60-second user-facing outage. Azure met its SLA. Your system failed its users. The feature did exactly what it was supposed to do. The gap was in the application.

SLA eligibility conditions are another place where the gap hides. Azure SQL Database's 99.99% SLA requires zone-redundant configuration *and* a supported service tier. AKS's SLA only applies when you're on the Standard or Premium pricing tier, the Free tier carries no SLA whatsoever. These conditions aren't buried in footnotes; they're in the documentation. But teams often select services, enable features, and then assume the headline number applies without verifying that their specific deployment topology actually qualifies.

Health probes are perhaps the subtlest trap. The default Azure Load Balancer probe interval is 5 seconds, with an unhealthy threshold of 2 consecutive failures. That means up to 10 seconds of continued traffic routing to a failed instance before the load balancer makes a change. During those 10 seconds, users get errors. And that's the best case, assuming the probe is correctly configured. If your TCP probe is checking port 80 and your application actually listens on 8080, every instance in your backend pool looks perpetually healthy to the load balancer while users get nothing from any of them.

Then there's the arithmetic that almost nobody actually does. Individual Azure service SLAs look reassuring, 99.99% here, 99.95% there. But when services are chained together in a dependency graph, composite availability is multiplicative, not additive:

Composite SLA = SLA₁ × SLA₂ × SLA₃ × ... × SLAₙ

A simple three-service stack, App Service at 99.99%, Azure SQL at 99.99%, Azure Cache for Redis at 99.9%, gives you a composite of roughly 99.88%. That's approximately 10.5 hours of platform-acceptable downtime per year, before your application code does anything wrong, before anyone makes a configuration mistake, before any deployment goes sideways.

Scale that to a realistic enterprise workload with eight to twelve services on the critical path, mixed tiers, and services that don't individually qualify for their headline SLAs, and you can easily land below 99.5%. That's over 43 hours of annual downtime, nearly two full days, from platform-level math alone. And it still doesn't account for the application.


## Azure Stays Up. Your Configuration Doesn't.

The pattern in real Azure incidents is worth examining, because it runs consistently counter to the assumption that infrastructure failure is the main risk.

In October 2025, a faulty tenant configuration change was pushed to Azure Front Door infrastructure. A software bug in the protection mechanism that should have caught the change allowed it to propagate an inconsistent configuration across AFD's global edge network. DNS and the underlying physical network were unaffected, traffic was reaching Azure's edge nodes just fine. But the routing logic was broken. Requests arrived and had nowhere valid to go. Microsoft 365, Teams, Outlook, Xbox Live, Alaska Airlines, Starbucks, Costco, all disrupted. The physical infrastructure performed exactly as designed. The configuration layer failed. The auto-rollback mechanism that should have triggered to limit the blast radius? It had been disabled since 2023. Nobody had triggered it, nobody had needed it, until they did.

In July 2024, an automated workflow in the Central US region published an allow-list update for Azure Storage that was missing address ranges for a significant number of VM hosts. The workflow failed to detect the missing data and published the incomplete configuration anyway. VMs couldn't reach their storage. Infrastructure was healthy. Config propagation logic was not. The region experienced widespread VM and storage disruptions that lasted hours.

In January 2023, a WAN IP configuration change caused multi-hour, multi-continent service disruption. Physical hardware: untouched. Network configuration: wrong.

The pattern across nearly every major Azure incident, and this pattern holds across AWS and GCP incidents too, is that the physical infrastructure survives. What fails is a configuration change that propagated incorrectly, a software bug in a control plane component, a dependency interaction that nobody modeled, or an operational procedure that worked in normal conditions and fell apart under stress. These aren't infrastructure problems. They're architectural and operational ones. And they're problems that zone redundancy and geo-replication do essentially nothing to address.

To be fair: genuine infrastructure failures happen. The West US datacenter power event in early 2026, a transformer failure that cascaded because the automated transfer to generator power didn't complete, was a real physical infrastructure failure, and zone redundancy across regions is exactly the right protection against events like that. But those events are increasingly the minority. The more common failure mode is configuration, control plane logic, and application behavior under degraded conditions.


## The Ways Your Application Ends Itself

The failure modes worth understanding most carefully aren't the dramatic ones, a whole region going dark, a major Azure service failing. Those are visible, declared on the status page, and short-lived. The insidious failures are the ones your application engineers for itself, slowly, with no help from Azure.

Consider the slow dependency spiral. A downstream service starts degrading, not failing, just responding slowly. Your application's connection pool starts filling up with threads waiting for responses that are taking three seconds instead of three hundred milliseconds. Before long the pool is full, and now requests to other, perfectly healthy dependencies start failing too, because there are no free threads to handle them. Your load balancer detects the elevated error rate and starts marking instances unhealthy. It removes them from rotation. The traffic that was distributed across five instances now concentrates on three, which get overwhelmed, which start returning errors, which triggers more probe failures, which removes more instances. Within minutes your entire backend has been removed from load balancing rotation, not because anything in Azure failed, but because one slow third-party API call occupied all your connection threads. The platform is fine. Your application ate itself.

The health probe lie compounds this. Your `/health` endpoint dutifully returns HTTP 200. The load balancer sees a healthy instance. What it doesn't see is that your database connection pool is exhausted and every actual user request is returning a 500 error. A health probe that checks port reachability or process liveness without verifying actual dependency health isn't a health check, it's a false alibi. A meaningful health endpoint verifies database connectivity, cache reachability, and the status of any critical dependencies. If those checks fail, the instance should fail its health probe, so the load balancer can route around it while the problem is resolved. Shallow probes rob you of the one mechanism that could have contained the blast radius.

Transient fault handling, or the absence of it, is where Azure's reliability can paradoxically hurt you. Azure SQL, Cosmos DB, Redis Cache, these services are maintained, patched, and occasionally fail over. These events are designed to be brief. A database failover in a zone-redundant SQL deployment is typically resolved in under 60 seconds. That brief disruption is *supposed* to be invisible to users. It is invisible, if your application implements retry logic with exponential backoff. Without it, a 60-second platform maintenance event surfaces directly to users as a 60-second outage. The platform held up its end. The application didn't hold up its end.

Then there's the well-intentioned disaster of naive retry logic. Service A calls Service B, which is struggling under load and responding slowly. Service A retries the call immediately on timeout. Service B, already saturated, now receives the original volume *plus* the retry volume. The added pressure pushes it past the threshold. Service B fails completely. Service A retries more aggressively. Other services that depend on B follow the same pattern. What started as degraded performance in one service becomes a full cascade because every caller's retry logic transformed a struggling system into a collapsed one. Retry logic without exponential backoff and jitter isn't a resilience pattern, it's a self-inflicted DDoS waiting for the right moment.


## The Platform Maturity Paradox

Here's the argument that tends to make people uncomfortable, because it's counterintuitive: Azure's reliability improvements have made certain failure modes harder to defend against. Not because Azure is worse, but because platform reliability breeds exactly the kind of complacency that leaves systems vulnerable to the failure modes the platform doesn't protect against.

Think about the operational posture of an on-premises infrastructure team fifteen years ago. Disks failed regularly. Networks were flaky. Power events happened. Everyone had runbooks. Failure was expected, practiced for, and responded to as a matter of routine. Teams had muscle memory for incident response because incidents weren't rare.

On Azure today, genuine infrastructure failures are infrequent enough that many engineering teams have never responded to one. Runbooks exist but haven't been opened in two years. Failover procedures are documented but untested. Auto-rollback mechanisms, like the one in the AFD incident, get quietly disabled because they've never been needed, and nobody schedules the work to maintain them. The safety nets accumulate technical debt while the platform maintains its uptime, and everyone assumes the safety nets are fine until the one day they're needed.

The feature checklist accelerates this. Zone redundancy: ✓. Geo-replication: ✓. Premium tier: ✓. There is something psychologically satisfying about a list of enabled features in the Azure portal, and that satisfaction is dangerous. What the list represents is the purchase of prerequisites for availability. Earning actual availability is the next step, and it requires engineering work, application resilience patterns, meaningful health checks, chaos engineering, tested failover procedures, that no portal toggle can do for you.

Abstraction makes it worse. On-premises, failure modes were legible. You knew what a failing disk meant for the systems that depended on it. You could trace the dependency chain. In the cloud, PaaS abstractions that make development faster also obscure the infrastructure underneath. Teams often don't know what they're genuinely depending on until something fails in a way they didn't anticipate. The higher the abstraction, the more invisible the failure modes, and the more surprised you are when they surface.

The platform maturity paradox, stated plainly: the more reliable Azure becomes, the less teams practice for failure, the more they trust the checklist, and the less they understand the failure modes that remain. Azure gets better at the part it controls. The part you control drifts.


## What Actual Availability Engineering Looks Like

The good news is that none of this is particularly mysterious. The engineering discipline for high-availability systems is well understood. It just requires treating availability as ongoing engineering work rather than a one-time configuration exercise.

**Start with failure mode analysis before you think about features.** Map your critical user flows, the paths through your system that matter most if they fail. For each component in those flows, ask: how can this fail? What's the user impact? What mitigates it? This is unglamorous work. There's no satisfying portal dashboard for it. But it's where actual availability gets built, because you can't design for failures you haven't named. Azure's Well-Architected Framework recommends failure mode analysis explicitly. Very few teams do it rigorously. Do it anyway.

**Do the composite SLA math for your actual system.** Don't look at individual service SLAs in isolation. List every service on your critical path, verify your deployment configuration qualifies for the headline SLA numbers, and multiply them. If the resulting number is worse than your availability target, that's not a problem to explain away, it's information. Either you need redundancy at that layer, or your targets need to reflect reality. Both are defensible positions. Pretending the math doesn't apply is not.

**Build resilience into the application, not just the infrastructure.** Circuit breakers on every external dependency, not to prevent failures, but to detect them fast and fail gracefully rather than letting a struggling service exhaust your connection threads. Bulkheads to isolate connection pools by dependency so a slow third-party API can't starve your database calls of threads. Exponential backoff with jitter on all retries, jitter specifically to prevent the thundering herd problem where every caller retries at exactly the same time and amplifies load on a recovering service. Health endpoints that check actual dependency health, not just process liveness. These patterns aren't exotic; they're the Polly library in .NET, Resilience4j in Java, built-in SDK capabilities in most Azure client libraries. The question isn't whether the tools exist, they do, it's whether anyone configured them.

**Test your failover assumptions, because they're probably wrong.** A failover procedure that's never been executed is a failover procedure you cannot trust. Schedule regular chaos engineering exercises in non-production environments. Terminate database instances and verify your retry logic handles it cleanly. Kill pods and verify your health probes and scaling behave as expected. Test your AKS cluster with an availability zone failure injected. What you'll find, the first time you do this in a real environment, is that the system doesn't behave the way you assumed. Connection pools don't drain the way you expected. Health endpoints don't fail fast enough. Retry storms emerge from places you didn't anticipate. Finding this in a chaos exercise at 2pm on a Tuesday is dramatically better than finding it in production at 2am on a Friday.

**Monitor the user experience, not just the infrastructure.** Infrastructure metrics, CPU utilization, memory pressure, error rates on individual services, tell you whether the infrastructure is healthy. They don't tell you whether a user can successfully complete a checkout, submit a support ticket, or log in to their account. Those are different questions. Synthetic transaction monitoring runs real user flows against your production system on a schedule and alerts when they fail. It's the monitoring that answers the question your users are asking. Most teams have the infrastructure monitoring. Fewer have the synthetic transaction monitoring. The second kind is the one that actually catches system-level failures.

**Treat deployment as an availability event.** Every deployment carries availability risk. Zero-downtime rolling deployments sound straightforward, but they require stateless application design, in-memory session state breaks when users get routed to a new instance mid-session. Database schema changes during rolling deployments require backwards compatibility between old and new application versions, a migration that drops a column the old version is still reading causes failures during the transition window. Rollback procedures need to be tested, not assumed. The platform can execute a rolling deployment; whether your application survives one is a function of how it was designed.


## Availability Is a Posture, Not a Purchase

Azure provides a genuinely excellent foundation. Zone redundancy, managed failover, geo-replication, SLA-backed services, these are real capabilities that meaningfully reduce infrastructure risk. The platform has earned its reputation. Use it.

But the foundation is exactly where Microsoft's responsibility ends and yours begins. Nobody at Microsoft is reviewing your application for missing retry logic. Nobody is verifying that your health probes actually reflect application health. Nobody is auditing your dependency graph for circuit breaker coverage or testing whether your chaos runbooks are current.

The teams that operate highly available systems on Azure don't just enable the right features. They design for failure from the start, test their assumptions regularly, and treat availability as an ongoing engineering discipline, something you maintain, not something you achieve once and move on from. The checklist gets you the prerequisites. The engineering gets you the outcome.

Azure will stay up. Make sure your system does too.

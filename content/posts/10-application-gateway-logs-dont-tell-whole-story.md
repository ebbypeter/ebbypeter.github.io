---
title: "Why Your Application Gateway Logs Don't Tell the Whole Story (Until You Correlate Them)"
draft: false
date: 2026-03-10
tags:
  - "azure"
  - "application-gateway"
  - "observability"
  - "waf"
  - "kql"
  - "sre"
  - "troubleshooting"
categories: 
  - Azure
  - Site Reliability Engineering
summary: "Access logs, firewall logs, backend health, and metrics each tell a partial truth about what Application Gateway is doing. Here's how they mislead you in isolation, and the KQL that fixes that."
---

You get paged. Users can't reach the app. You pull up Application Gateway access logs, find the errors, check backend health in the portal, and everything looks fine. So you stare at the access logs a bit longer, maybe open the metrics blade, maybe run the same query twice. Twenty minutes in, you still don't know what happened.

This isn't a skill problem. It's a structural one.

Application Gateway produces four distinct telemetry surfaces: access logs, firewall logs, backend health, and metrics. Each one is scoped to a different layer of the system. None of them were designed to natively join with the others. And critically, the access log and firewall log in v2 share no common key field, meaning there's no clean way to say "this 403 in the access log corresponds to *this* WAF rule evaluation in the firewall log." You're doing fuzzy correlation on IP address and URI and hoping the timestamps line up.

That architecture means specific failure scenarios are almost guaranteed to mislead you if you're working from a single source. The three scenarios below are the ones that come up most in practice. Each one shows you the misleading signal first, explains what's actually happening, and ends with the query that cuts through it.

One thing to sort out before an incident, not during it: access logs and firewall logs are both opt-in. They're separate diagnostic settings, and it's completely possible to have access logs flowing while firewall logs are dark. Both need to land in the same Log Analytics workspace for any of the cross-source correlation to work. Check that now.

---

## "502s, but the backend looks healthy"

The pager fires. You check the access log: `httpStatus` 502, volume is significant, started roughly eight minutes ago. Backend health portal shows all green. You refresh it. Still green.

Here's what the portal isn't showing you. The default health probe fires every 30 seconds, waits 30 seconds for a response, and requires 3 consecutive failures before marking a backend unhealthy. That's a 90-second worst-case window between a backend going down and AppGW acknowledging it. During those 90 seconds, live traffic keeps hitting the failing backend, and your users keep getting 502s, while the portal continues to report Healthy because the probes haven't crossed the threshold yet.

Backend health is a point-in-time state snapshot, not a historical log. You can't query it for what it looked like at 14:32. It's also meaningless after the fact, by the time you open the portal, the backend might have recovered, and you're looking at current state while trying to diagnose something that happened ten minutes ago.

The access log field that actually helps here is `ServerStatus`. In v2, this records the HTTP status code that Application Gateway received from the backend on that specific request. When AppGW can't reach the backend at all, timeout, refused connection, incomplete response, this field is absent. That's the tell.

Two different problems produce 502s in the access log, and they look identical to the end user but have completely different causes. One is "AppGW reached the backend and got something it couldn't use." The other is "AppGW never got a response." The `ServerStatus` field tells you which one you're dealing with.

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where HttpStatus == 502
| extend BackendResponded = isnotempty(ServerStatus)
| summarize
    Total = count(),
    GotBackendResponse = countif(BackendResponded == true),
    NoBackendResponse = countif(BackendResponded == false),
    BackendStatuses = make_set(ServerStatus, 5)
    by bin(TimeGenerated, 5m), ServerRouted
| order by TimeGenerated desc
```

If `NoBackendResponse` is climbing, you have a connectivity or reachability problem. NSG rules, UDRs, or an actual service failure. The backend health portal will eventually catch up, but this query tells you what's happening in real time with per-instance granularity.

There's a second variant worth flagging. Application Gateway's default probe hits `http://127.0.0.1:<port>/`, the root path on the loopback interface, using whatever port is in your HTTP settings. If your application's health isn't accurately represented by a 200 from that path, your probes are measuring something that doesn't matter. The probe hits `/` and returns 200 from the static web server. Real traffic hits `/api/orders`, which is failing because the database connection pool is exhausted. Backend health says Healthy. Access logs say 502.

Custom probes aren't complicated to set up. If you're on default probes for anything more complex than a static file server, it's worth revisiting what they're actually validating.

---

## "Users are getting 403s, but the app doesn't return 403"

A user reports that a specific request is returning 403. You find the entries in the access log: `HttpStatus` 403, `ServerStatus` blank. The app doesn't have a 403 code path on that endpoint. Something between the client and the backend made that decision.

Almost certainly WAF. And if your firewall logs aren't enabled, you're done. You know WAF is probably involved, you know which client and URI are affected, and you have absolutely nothing else to go on. Access logs and firewall logs are separate diagnostic settings. Enabling access logs does not enable firewall logs. This is the single most common gap when teams try to troubleshoot WAF incidents, and it only becomes obvious when the incident is already in progress.

Assuming firewall logs are flowing, the next problem is the join. In v2, the firewall log has a `TransactionId` field that groups every rule evaluation for a single request. The access log doesn't have this field. There's no foreign key. To correlate a 403 in the access log with its corresponding firewall evaluation, you join on `ClientIp + RequestUri + TimeGenerated` within a narrow time window. Under any meaningful load, especially if you're dealing with a scanner or bot that's hammering the same URI from the same IP, that join produces noise.

```kql
let AccessBlocked = AGWAccessLogs
| where TimeGenerated > ago(1h)
| where HttpStatus == 403
| where isempty(ServerStatus)
| extend JoinKey = strcat(ClientIp, "|", RequestUri);

let FirewallBlocked = AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where Action == "Blocked"
| extend JoinKey = strcat(ClientIp, "|", RequestUri);

AccessBlocked
| join kind=leftouter (FirewallBlocked) on JoinKey
| project
    TimeGenerated,
    ClientIp,
    RequestUri,
    WAFRuleId = RuleId,
    WAFMessage = Message,
    RuleSetVersion
| order by TimeGenerated desc
```

It'll get you close. But understand what you're looking at: an approximate match, not a confirmed one.

Now, even when you find the firewall log entry for a blocked request, there's another layer to understand. Under CRS 3.x, which is the default ruleset, WAF uses anomaly scoring. Individual rules don't block on their own. They each contribute a score, and the request gets blocked when the cumulative score exceeds a threshold (typically 5). A single blocked request can produce four, six, even ten firewall log entries, all sharing the same `TransactionId`. The final entry has `Action: Blocked` and points to rule 949110, "Inbound Anomaly Score Exceeded."

Rule 949110 tells you nothing about what actually triggered the block. It's the mechanism, not the cause. The cause is in the earlier entries with `Action: Matched`. Those are the rules that accumulated the score. If you only look at the Blocked entry and try to suppress rule 949110, you'll suppress the blocking infrastructure and WAF will stop working entirely.

```kql
let BlockedTxns = AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where Action == "Blocked"
| distinct TransactionId;

AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where TransactionId in (BlockedTxns)
| where Action == "Matched"
| summarize
    Firings = count(),
    AffectedClients = dcount(ClientIp),
    SampleURIs = make_set(RequestUri, 5)
    by RuleId, Message
| order by Firings desc
```

This is what you tune against. The rules at the top of this list are the ones accumulating score and causing real traffic to be blocked. Add exclusions here, not on 949110.

One more thing worth saying plainly. WAF in detection mode logs every rule match but blocks nothing. Detection mode is the right starting point for new deployments, you can build up exclusions and tune the policy without immediately breaking traffic. But detection mode where nobody reviews the firewall logs is security theater. You're running WAF, paying for it, taking the latency hit from rule evaluation, and getting none of the protection. If you haven't looked at your firewall logs in the last week, run this:

```kql
AGWFirewallLogs
| where TimeGenerated > ago(24h)
| where Action == "Detected"
| summarize
    Hits = count(),
    UniqueClients = dcount(ClientIp),
    UniqueURIs = dcount(RequestUri)
    by RuleId, Message
| order by Hits desc
```

If the results are noisy, there's work to do. Either tune the policy and move to prevention, or accept that you're operating in a mode that provides logging without enforcement, and make that a conscious decision.

---

## "Latency spiked, is it the app or the gateway?"

P95 latency has jumped. Users are complaining. The instinct is to go straight to the application team. But Application Gateway sits in every request's path and owns multiple segments of that request's lifecycle. Without decomposing the timing, you're guessing at which one to look at.

The access log's `TimeTaken` field records the full gateway-observed duration: from the moment Application Gateway receives the first byte from the client to the moment it finishes sending the response back. That number is a sum of segments, client-to-gateway network, gateway processing (including WAF rule evaluation and TLS offload), connection establishment to the backend, backend processing time, backend-to-gateway response transfer, and gateway-to-client send. If `TimeTaken` is elevated, something in that chain is slow. `TimeTaken` alone won't tell you where.

`ServerResponseLatency` breaks off the backend's contribution. It records the time from when AppGW sent the request to the backend until it received the last response byte. Subtract `ServerResponseLatency` from `TimeTaken` and you get a rough approximation of gateway overhead, everything the gateway itself is spending time on.

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where TimeTaken > 2
| extend GatewayOverheadSec = TimeTaken - ServerResponseLatency
| summarize
    P95Total = percentile(TimeTaken, 95),
    P95Backend = percentile(ServerResponseLatency, 95),
    P95Overhead = percentile(GatewayOverheadSec, 95),
    RequestCount = count()
    by bin(TimeGenerated, 1m), ServerRouted
| order by TimeGenerated desc
```

If `P95Backend` is climbing, the backend is slow. Database query times, downstream service dependencies, GC pauses, that's where to look.

If `P95Overhead` is climbing with stable backend latency, the time is being spent inside the gateway. A few causes worth checking: WAF rule evaluation (CRS with large exclusion lists and custom rules adds measurable overhead that accumulates under load), TLS handshake time under high connection churn, and AppGW instance capacity. For connection establishment specifically, the `BackendConnectTime` metric in Azure Monitor is your signal, if establishing connections to the backend is taking longer, you're looking at network policy or routing between the AppGW subnet and the backend subnet, not application performance.

The reason this decomposition matters beyond assigning blame: WAF-induced latency is regularly misattributed to the application. Someone adds a wave of custom rules or a new managed ruleset, P95 climbs, and the app team spends a day profiling code that's not the problem. The gateway overhead calculation surfaces that directly.

---

## Getting the setup right before the next incident

The scenarios above require multiple log sources to be in the same workspace. If they're not, the correlation queries don't work. A few things worth verifying now rather than during an outage.

Enable both log types and route them to the same workspace. Access logs and firewall logs are independent diagnostic settings under the same resource. Easy to miss the second one when you're setting things up for the first time.

Use resource-specific tables. When configuring diagnostic settings, choose "Resource-specific" rather than "Azure Diagnostics." This routes logs to `AGWAccessLogs` and `AGWFirewallLogs` instead of the shared `AzureDiagnostics` table. The schemas are cleaner, field names are predictable without type suffixes, queries run faster. If you're migrating from AzureDiagnostics, run both settings in parallel for a few days while you validate, then remove the old one. Leaving both active permanently means you're ingesting and paying for everything twice.

Write real health probes. The default probe is almost certainly not validating what you actually need it to validate. A probe that hits the root path and gets a 200 from the web server tells you the web server process is alive. It tells you nothing about database connectivity, session store availability, or whether the application can actually handle a real request. Custom probes are straightforward to configure and they're the only reliable way to avoid the scenario where backend health says green while your users are getting 500s.

Alert on `UnhealthyHostCount > 0` per backend pool. It's a trailing indicator, the 90-second detection window means you'll always be a minute behind, but it's reliable and low-noise. When something goes persistently unhealthy, you want that alert routing somewhere that gets immediate attention, not sitting in a dashboard nobody has open.

And alert on backend 5xx in `ServerStatus`, not just on `HttpStatus`. Application Gateway rewrites responses in various configurations: custom error pages, rewrite rules, WAF blocks. The status code returned to the client doesn't always reflect what the backend said. A backend flooding 500s while AppGW returns custom 200 error pages is a real scenario, and `ServerStatus` is the only field that catches it.

---

## The queries, collected

For when you need them fast.

**502 triage, backend response or no contact at all:**

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where HttpStatus == 502
| extend BackendResponded = isnotempty(ServerStatus)
| summarize
    Total = count(),
    GotResponse = countif(BackendResponded == true),
    NoResponse = countif(BackendResponded == false)
    by bin(TimeGenerated, 5m), ServerRouted
| order by TimeGenerated desc
```

**WAF block root cause, contributing rules, not the threshold rule:**

```kql
let BlockedTxns = AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where Action == "Blocked"
| distinct TransactionId;
AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where TransactionId in (BlockedTxns)
| where Action == "Matched"
| summarize
    Firings = count(),
    AffectedClients = dcount(ClientIp)
    by RuleId, Message
| order by Firings desc
```

**Latency decomposition, gateway overhead vs backend time:**

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where TimeTaken > 2
| extend GatewayOverheadSec = TimeTaken - ServerResponseLatency
| summarize
    P95Total = percentile(TimeTaken, 95),
    P95Backend = percentile(ServerResponseLatency, 95),
    P95Overhead = percentile(GatewayOverheadSec, 95)
    by bin(TimeGenerated, 1m)
| order by TimeGenerated desc
```

The mental model to carry forward: Application Gateway is three semi-independent subsystems, the gateway, the WAF engine, and the health probing system. Their logs don't share a primary key. Correlation across them is always approximate, always time-windowed, and only works if they're all in the same workspace. The individual logs aren't misleading on purpose, they're just each honest about their own layer and unaware of the others. Bridging that is your job. Now you have the tools to do it.

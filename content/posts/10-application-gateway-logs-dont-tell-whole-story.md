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

Application Gateway exposes four telemetry surfaces: access logs, firewall logs, backend health, and metrics. Each is scoped to a different subsystem. None of them natively join with the others. In v2, the access log and firewall log don't even share a common key field — `transactionId` exists in the firewall log but not in the access log, so there's no clean way to say "this 403 corresponds to *that* WAF rule evaluation." You're correlating on IP address, URI, and timestamp and hoping it holds.

That architecture means specific failure scenarios will reliably mislead you if you're reading from a single source. The three below are the ones that come up most in production.

One prerequisite: access logs and firewall logs are both opt-in, and they're separate diagnostic settings. Both need to land in the same Log Analytics workspace for the correlation queries here to work. Verify that before the next incident, not during it.

![Application Gateway Telemetry Architecture](/images/posts/appgw-telemetry-architecture-light.svg)

---

## Scenario 1: "502s, but backend health is green"

You check the portal. All backends showing Healthy. You go back to the access logs, stare at the 502s, and hit a wall.

Here's what the portal isn't telling you. The default health probe fires every 30 seconds, waits up to 30 seconds for a response, and requires 3 consecutive failures before marking a backend unhealthy. That's a 90-second worst-case window between a backend going down and AppGW acknowledging it. During those 90 seconds, live traffic keeps routing to the failing backend. The portal still shows green because the probe threshold hasn't been crossed yet.

Backend health is a point-in-time state snapshot. It tells you nothing about what it looked like 90 seconds ago.

The access log field that cuts through this is `ServerStatus`. In v2, this records the HTTP status code AppGW received from the backend on that specific request. When AppGW can't reach the backend at all — timeout, refused connection — the field is absent. That absence is your signal. A 502 with a populated `ServerStatus` means the backend responded but sent something unusable. A 502 with a blank `ServerStatus` means the gateway never got a response. Different problems, different remediation paths, identical from the client's perspective.

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where HttpStatus == 502
| extend BackendResponded = isnotempty(ServerStatus)
| summarize
    Total = count(),
    GotResponse = countif(BackendResponded == true),
    NoResponse = countif(BackendResponded == false),
    BackendStatuses = make_set(ServerStatus, 5)
    by bin(TimeGenerated, 5m), ServerRouted
| order by TimeGenerated desc
```

If `NoResponse` is climbing, you have a connectivity or reachability problem. Check NSG rules and UDRs. The backend health portal will catch up eventually, but this tells you what's happening now, per backend instance.

A related failure: the probe path mismatch. The default probe hits the root path (`/`) via the loopback interface. If your application's real health isn't represented by a 200 from that path, you're measuring something that doesn't matter. Probe returns 200, real traffic to `/api/orders` fails because the database pool is exhausted, backend health says Healthy. Custom probes aren't complicated to configure and they're the only way to close this gap.

---

## Scenario 2: "Users are getting 403s, but the app doesn't return 403"

Access log shows `HttpStatus` 403, `ServerStatus` blank. The backend was never contacted.

WAF blocked the request. And if firewall logs aren't enabled, you're done. You know WAF is involved, you know which client and URI triggered it, and you have nothing else to work with. Access logs and firewall logs are separate diagnostic settings. This is the most common instrumentation gap on WAF-enabled gateways, and it only becomes obvious mid-incident.

Assuming firewall logs are flowing, the correlation is still fuzzy. The firewall log's `TransactionId` groups every rule evaluation for a single request. The access log doesn't have this field. To connect a specific 403 to its WAF evaluation you join on `ClientIp + RequestUri + TimeGenerated` within a narrow window. Under load, that's approximate at best.

Even once you find the firewall entries, there's another layer. Under CRS 3.x (the default ruleset), WAF uses anomaly scoring. Rules contribute scores; the request blocks when the cumulative total exceeds a threshold. A single blocked request generates multiple firewall log entries, all sharing the same `TransactionId`. The final entry has `Action: Blocked` and points to rule 949110, "Inbound Anomaly Score Exceeded." That rule tells you nothing about why the request was blocked. The actual contributing rules are the earlier entries with `Action: Matched`.

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

Tune against these results. Don't suppress rule 949110 — that breaks the blocking mechanism entirely.

On detection mode: WAF in detection mode logs every rule match but blocks nothing. It's the right starting posture for a new deployment while you build exclusions. But detection mode where nobody reviews the firewall logs isn't a posture — it's security theater. You're paying for WAF overhead and getting none of the protection. If you haven't looked at your `Action == "Detected"` entries recently, that's worth an hour of your time.

---

## Scenario 3: "Latency spiked — where is the time going?"

P95 is elevated. `TimeTaken` in the access log confirms the gateway is slow. But `TimeTaken` is the full gateway-observed duration: client-to-gateway network, WAF rule evaluation, TLS offload, connection establishment to the backend, backend processing, and response transfer back. Something in that chain is slow. `TimeTaken` alone doesn't tell you which segment.

`ServerResponseLatency` isolates the backend's contribution — the time from when AppGW sent the request until it received the last response byte. The difference between `TimeTaken` and `ServerResponseLatency` approximates everything the gateway itself is spending time on.

```kql
AGWAccessLogs
| where TimeGenerated > ago(1h)
| where TimeTaken > 2
| extend GatewayOverheadSec = TimeTaken - ServerResponseLatency
| summarize
    P95Total = percentile(TimeTaken, 95),
    P95Backend = percentile(ServerResponseLatency, 95),
    P95Overhead = percentile(GatewayOverheadSec, 95)
    by bin(TimeGenerated, 1m), ServerRouted
| order by TimeGenerated desc
```

High `P95Backend` means the backend is slow. High `P95Overhead` with stable backend latency means the time is inside the gateway. For connection establishment specifically, the `BackendConnectTime` metric in Azure Monitor tells you whether AppGW is spending time just reaching the backend — a signal for network policy or routing issues between the subnets.

WAF-induced latency is worth calling out directly. CRS with complex exclusion lists and custom rules adds overhead that accumulates under load. It shows up as gateway overhead, not backend latency. This gets misattributed to the application more often than it should. Someone adds a batch of custom rules, P95 climbs, the app team spends a day profiling code that isn't the problem.

---

## Get the setup right before the next incident

Three things to verify now.

Enable both log types and route them to the same workspace. Access logs and firewall logs are independent diagnostic settings on the same resource. It's easy to enable one and miss the other.

Use resource-specific tables. When configuring diagnostic settings, choose "Resource-specific" over "Azure Diagnostics." This routes to `AGWAccessLogs` and `AGWFirewallLogs` instead of the shared `AzureDiagnostics` table. Cleaner schema, predictable field names, better query performance. If you're migrating from the legacy table, run both settings in parallel briefly to validate, then remove the old one. Running both permanently means you're ingesting and paying for everything twice.

Write real health probes. The default probe doesn't validate application readiness. A probe that returns 200 from the web server root tells you nothing about database connectivity, downstream service availability, or whether the app can serve a real request. Custom probes are straightforward to configure and the only reliable way to avoid the Scenario 1 failure mode.

---

## The mental model

Application Gateway is three semi-independent subsystems — the gateway, the WAF engine, and the health probing system. Their logs don't share a primary key. Correlation across them is always approximate, always time-windowed, and only works when all sources are in the same workspace.

The individual logs aren't misleading by design. They're each honest about their own layer. What they can't do is see past it. That's your job to bridge, and now you have the queries to do it.


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

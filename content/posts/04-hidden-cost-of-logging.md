---
title: "The Hidden Cost of 'Just Turn On Logging' in Azure"
draft: false
date: 2026-01-27
tags:
  - azure-monitor
  - log-analytics
  - azure-diagnostics
  - cost-optimization
  - dcr
  - observability
  - finops
  - kql
  - resource-specific-logs
  - azure-policy
categories:
  - Azure
  - Cloud Engineering
  - FinOps
summary: "Your team enabled logging everywhere, a responsible move. Then the Azure bill arrived. This post traces exactly why Log Analytics costs spiral without warning, what the AzureDiagnostics table is quietly doing to your budget, and how resource-specific tables plus DCR transformations give you back control."
---

*Why your Log Analytics bill is lying to you, and how to fix it before it compounds.*

---

Nobody budgets for observability until the bill arrives.

Here's a scenario that plays out more often than anyone admits. A platform team, doing the right thing, rolls out an Azure Policy with a `DeployIfNotExists` rule: if a resource doesn't have a Diagnostic Setting, create one. Automatic. At scale. All categories. Destination: Log Analytics workspace.

Three months later, the cloud bill has a Log Analytics line item that's four times what anyone expected. Someone runs a cost analysis, traces it back to the policy, and realizes that hundreds of resources have been silently streaming every log category, SQL query statistics, AKS control plane chatter, firewall network flows, into a single workspace, billed at full price, forever.

The instinct is to blame logging. The real problem is the assumption behind it: that logging is a safe default, a simple toggle, something you turn on and forget.

It isn't. And the gap between that assumption and reality is where Azure bills quietly grow.

---

## Section 1: First, Correct the Mental Model

### Logging isn't on by default. So where's the money going?

Here's something that surprises a lot of teams: Azure resource logs are **not collected by default on any resource**. Not Key Vault. Not Azure SQL. Not AKS. Not Azure Firewall. Every single resource requires an explicit Diagnostic Setting that specifies what to collect and where to send it.

So if logging is off by default, how does the bill get out of control? The answer almost always involves scale. A Diagnostic Setting on one resource is a manageable decision. A Diagnostic Setting on every resource, applied automatically, with all categories enabled, with no volume estimation, is a different thing entirely.

A Diagnostic Setting is essentially a declaration with three parts: which resource, which log categories, and which destination. The destination options are a Log Analytics workspace, a Storage Account, an Event Hub, or a combination. Each has its own cost profile. When teams route everything to a Log Analytics workspace on the Analytics pricing plan without thinking about it, they're making the most expensive possible choice by default.

The subtler trap is in the categories. When you create a Diagnostic Setting, you're presented with a list of log categories for that resource type and asked to tick the ones you want. The temptation, especially in a policy that runs automatically, is to tick all of them. But log categories are not equal in volume or value.

Consider Azure SQL Database. Its diagnostic categories include `SQLInsights`, `AutomaticTuning`, `Errors`, `DatabaseWaitStatistics`, `Timeouts`, `Blocks`, `Deadlocks`, `QueryStoreRuntimeStatistics`, `QueryStoreWaitStatistics`, and more. A team troubleshooting slow queries probably needs `QueryStoreRuntimeStatistics` and `Errors`. They almost certainly don't need `AutomaticTuning` generating records every time the database engine adjusts an index recommendation.

> **Diagnostic settings are a category selector, not a filter. You choose what type of log to collect, you don't choose which individual records within that type. That distinction costs teams thousands of dollars a month.**

This is the foundational misunderstanding. And it shapes every problem that follows.

---

## Section 2: The `AzureDiagnostics` Table Is the Worst Kind of Monolith

### A single table absorbing logs from every Azure service you've ever touched. What could go wrong?

When you create a Diagnostic Setting and point it at a Log Analytics workspace, you have a choice (where supported) about how the data gets stored. The legacy option sends everything into a table called `AzureDiagnostics`. Understanding what that table actually is, and what it does to your logging strategy, is one of the most valuable things you can do before scaling out your observability coverage.

### What `AzureDiagnostics` actually is

`AzureDiagnostics` is a single, flat Log Analytics table that acts as a catch-all for resource logs from every Azure service using legacy "Azure Diagnostics mode." Key Vault audit events, SQL Server query statistics, Application Gateway access logs, Azure Firewall network rules, AKS control plane events, they all land in the same table, merged into a shared schema.

Nobody designed this schema for any particular use case. It emerged over time as each Azure service team added their own columns to what was supposed to be a common structure. The result is a massive, heterogeneous, column-bloated table that nobody designed holistically and everyone is forced to query.

### The 500-column ceiling

Log Analytics workspaces impose a hard limit of 500 columns per table. `AzureDiagnostics` is, by far, the most likely table in any workspace to hit that ceiling, because it absorbs schemas from dozens of different services, each with its own set of fields.

Every workspace starts with at least 200 pre-defined columns in `AzureDiagnostics`. Workspaces created before January 19, 2021 may have additional legacy columns already occupying that budget. As more resource types pipe data into the table, new columns are added, until the 500-column limit is reached.

When it is, something unfortunate happens: any data that would have gone into a new column gets redirected into a dynamic property bag called `AdditionalFields`. Structured logs become unstructured again. Fields that should be queryable as typed columns become strings buried inside a dynamic object that requires manual parsing to use.

The logs are still there. They're just harder to access than if you'd never tried to structure them in the first place.

### Query performance degrades at scale

Because `AzureDiagnostics` contains data from every resource type in your workspace, any query that doesn't explicitly restrict its scope will scan the entire table. In a large environment, that means scanning SQL Server records to find Key Vault events, or scanning firewall logs to find App Gateway errors.

Microsoft's own guidance is unambiguous: always filter by `ResourceType` and `TimeGenerated` at the very top of any `AzureDiagnostics` query, before any other logic. The difference between the right and wrong approach looks like this:

```kql
// ❌ The wrong way, scans the entire table
AzureDiagnostics
| where Category == "AuditEvent"

// ✅ The right way, scopes first, then filters
AzureDiagnostics
| where ResourceType == "KEYVAULTS"
| where TimeGenerated > ago(1d)
| where Category == "AuditEvent"
```

The first query will be slow and expensive in any environment where `AzureDiagnostics` is large. The second is fast. The difference is just two lines added at the top, but most queries written against `AzureDiagnostics` don't include them, because the table's structure doesn't make the need obvious until performance becomes a problem.

### RBAC is all-or-nothing

The entire `AzureDiagnostics` table is a single security boundary. There is no mechanism for granting a security team read access to Azure Firewall logs without also giving them access to SQL Server audit trails, Key Vault access records, and anything else that flows into the same table.

This isn't just an ergonomics problem, it's a compliance and data governance problem. In environments where different teams should have visibility into different resource types, `AzureDiagnostics` makes that separation structurally impossible. You either give someone everything in the table or nothing.

### The pricing tier trap, the thing nobody talks about

This is the issue most overlooked in conversations about `AzureDiagnostics`, and it has the most direct impact on cost.

Log Analytics tables can be assigned to one of three pricing plans: Analytics, Basic, or Auxiliary. Analytics is the most expensive; Basic and Auxiliary are significantly cheaper (more on the actual numbers in the next section). For many log types, high-volume, low-priority data that you might query once in a forensics investigation but never use for alerting, Basic or Auxiliary is the right plan.

`AzureDiagnostics` cannot be placed on the Basic or Auxiliary plans.

Because `AzureDiagnostics` is a shared table that receives data from multiple sources across multiple resource types, it doesn't qualify for the cheaper table plans that require a dedicated, single-purpose table. Everything that flows into `AzureDiagnostics` is billed at full Analytics rates, regardless of whether the data is worth that price.

That Azure Firewall network flow log you'd happily pay $0.15/GB to store? Once it's in `AzureDiagnostics`, it costs $2.30–$4.30/GB. The table itself is a billing ceiling.

---

## Section 3: The Real Cost of Azure Monitor Logs (With Actual Numbers)

### Three pricing tiers. Most teams use the expensive one for everything.

Azure Monitor Logs has a three-tier pricing model for Log Analytics tables. Most teams know about pay-as-you-go rates for the Analytics plan and assume that's the only option. It isn't. The gap between tiers is significant enough to materially change the economics of any observability strategy.

| Plan | Designed For | Approx. Ingestion Cost (East US) | Query Cost |
|---|---|---|---|
| **Analytics** | Active monitoring, alerting, real-time dashboards | ~$2.30–$4.30/GB (PAYG) | Free |
| **Basic** | Debugging, occasional forensics, compliance search | ~$0.50/GB | Charged per GB scanned |
| **Auxiliary** | High-volume, low-priority data (firewall flows, proxy logs, audit archives) | ~$0.15/GB | Charged per GB scanned |

Auxiliary Logs became generally available in April 2025. Many teams working with older architectures haven't adopted it yet, which means they're paying Analytics rates for data that rarely gets queried and doesn't drive any alerts.

The right framing is this: **the pricing tier should match the access pattern**, not the resource type. A firewall log doesn't automatically belong on the Analytics plan. It belongs there only if you're actively building alerts or dashboards on top of it. If it exists for forensics or compliance purposes and you query it a few times a year, it belongs on Auxiliary, at less than 7% of the Analytics cost.

Commitment tiers, starting at 100 GB per day, can cut Analytics ingestion costs by up to 30% compared to pay-as-you-go rates. That's a meaningful discount. But it's optimizing the wrong layer if the data shouldn't be on the Analytics plan in the first place. A 30% discount on a price you shouldn't be paying is still a cost that shouldn't exist.

### The free tables most teams don't know about

Several tables in Log Analytics are completely free to ingest and retain. These include:

- `AzureActivity`, the subscription-level audit log, recording every create/update/delete operation across your subscription
- `Heartbeat`, agent heartbeats from VMs and servers
- `Usage`, workspace data ingestion metrics
- `Operation`, workspace operational logs

Teams occasionally route these through Diagnostic Settings or other pipelines in ways that generate a charge. They shouldn't. These tables exist specifically to be free, paying to ingest them is a configuration mistake worth checking for.

### What actually costs money

Log volume varies enormously by resource type. Before enabling Diagnostic Settings at scale, it's worth estimating what you're about to commit to. A rough formula, `entries per month × average entry size × ingestion rate per GB`, gives a useful approximation.

A Key Vault audit event is roughly 912 bytes on average. A single vault with 2,500 requests per month generates about 2.3 MB of logs, effectively free. Scale that to 100 high-traffic vaults and the volume becomes more meaningful, but it's still manageable.

Azure Firewall is a different story. Network rule logs record every allowed and denied connection through the firewall. In a busy environment, that's every DNS lookup, every health probe, every outbound HTTP request from every resource behind the firewall. A single active Azure Firewall can generate tens of gigabytes of network rule logs per day. Enabling `AzureFirewallNetworkRule` without thinking about it is the fastest way to generate a Log Analytics bill that surprises everyone.

AKS control plane logs have a similar profile. The `AKSAudit` and `AKSControlPlane` categories capture Kubernetes API server activity, which includes constant background activity from system processes calling `ListPods`, `GetConfigMap`, and similar read-only operations. These events have essentially zero operational value for most teams but generate significant log volume at a busy cluster.

SQL Server audit logs record every query execution, which means the volume scales directly with database load. At anything approaching production scale, this category warrants careful consideration of whether it belongs on Analytics or a cheaper plan.

To see where your current costs are actually coming from, this query runs against your existing workspace and produces a breakdown by resource type and category:

```kql
AzureDiagnostics
| where _IsBillable == true
| where TimeGenerated > now(-31d)
| summarize BillableDataGB = round(sum(_BilledSize)/(1024*1024*1024), 2)
    by ResourceProvider, ResourceType, Category
| sort by BillableDataGB desc
```

Run this before expanding your diagnostic coverage. It will tell you exactly which categories are responsible for the majority of your current spend and give you a data-driven basis for deciding what to keep, what to move to a cheaper tier, and what to stop collecting.

### The daily cap: circuit breaker, not quota

Log Analytics workspaces let you configure a daily ingestion cap, a hard limit on how much data the workspace will accept in a 24-hour period. This sounds like a cost control mechanism. It is, but it comes with a critical caveat: when the cap is hit, **all ingestion stops**, not just the noisy source that triggered it.

That means if an Azure Firewall starts generating unexpected log volume and hits the cap, you also stop receiving Key Vault audit events, AKS alerts, and everything else in the workspace. Potentially at 2am, silently.

The daily cap is useful as a safety net during initial rollout, to prevent a misconfigured diagnostic setting from generating an unlimited bill. It is not a substitute for designing your logging strategy correctly. If you find yourself relying on the cap as a permanent cost control, that's a sign the underlying configuration needs fixing.

---

## Section 4: Resource-Specific Tables, The Migration You Should Already Be Doing

### Microsoft is deprecating `AzureDiagnostics` for most services. Here's how to get ahead of it.

Microsoft has been moving Azure services away from `AzureDiagnostics` for several years and has stated that all services will eventually use resource-specific mode. Some services already only support resource-specific tables. Others let you choose. A few still only write to `AzureDiagnostics`. The list of services still using the legacy mode is shrinking.

Resource-specific mode is not a future option or a preview feature. It's the current recommendation and the direction the platform is moving. For any new Diagnostic Setting you create today, there's no good reason not to use resource-specific mode when the option is available.

### What resource-specific mode actually does

Instead of routing all log categories from all resource types into a single `AzureDiagnostics` table, resource-specific mode creates individual tables for each log category. Each table is narrow, purpose-built, and contains only the columns relevant to that log type.

The naming convention follows the resource type and category. Azure Firewall gets `AZFWApplicationRule`, `AZFWNetworkRule`, and `AZFWDnsQuery`, separate tables, each with a schema designed specifically for that log type. AKS gets `AKSAudit`, `AKSAuditAdmin`, and `AKSControlPlane`. Key Vault gets `KeyVaultAuditEvent`. Each table has documented columns, typed fields, and no risk of hitting a shared column ceiling.

The practical benefits over `AzureDiagnostics` are significant across every dimension:

Queries are faster and simpler because the table only contains data for one resource type. Schemas are discoverable and consistent, IntelliSense works properly, column types are what you expect. RBAC can be applied at the table level, so a security team can read `AZFWNetworkRule` without seeing `KeyVaultAuditEvent`. And crucially, each table can be assigned to the Analytics, Basic, or Auxiliary pricing plan independently.

That last point is what makes resource-specific tables the prerequisite for any meaningful cost optimization. You cannot assign `AzureDiagnostics` to a cheaper tier. You can assign `AZFWNetworkRule` to Auxiliary, `AKSAudit` to Basic, and `KeyVaultAuditEvent` to Analytics, applying the right price to the right data access pattern for each one.

### How to migrate without losing historical data

The migration path is straightforward but requires a brief parallel-running period. Historical data already in `AzureDiagnostics` stays there, it doesn't move to the new tables automatically. It will age out according to your workspace retention settings. New data, once you update the Diagnostic Setting to resource-specific mode, flows into the dedicated table immediately.

During the transition, queries need to cover both sources. The `union` operator handles this cleanly:

```kql
// Query during migration, covers both old (AzureDiagnostics) and new (KeyVaultAuditEvent) data
union AzureDiagnostics, KeyVaultAuditEvent
| where (ResourceType == "KEYVAULTS" or Type == "KeyVaultAuditEvent")
| where TimeGenerated > ago(90d)
| where OperationName == "SecretGet"
```

Once your retention period has passed and historical data has aged out of `AzureDiagnostics`, you can simplify queries to target only the resource-specific table. The union is a temporary scaffold, not a permanent state.

### The RBAC and pricing unlocks that come with migration

After migrating to resource-specific tables, you have three new levers that weren't available before.

The first is per-table pricing plan assignment. Move high-volume, rarely-queried tables to Basic or Auxiliary. Keep tables that drive alerts and dashboards on Analytics. This single change can substantially reduce ingestion costs for environments heavy on firewall or AKS logs.

The second is table-level RBAC. Assign read permissions on `AZFWNetworkRule` to the network security team, `AzureSQLDatabaseInsights` to the DBA team, and `KeyVaultAuditEvent` to the compliance team, all without cross-pollinating access. This is not achievable with `AzureDiagnostics`.

The third is per-table retention configuration. Compliance logs might need to be retained for 730 days. Debug logs might be useless after 30 days. With resource-specific tables, you can configure these independently rather than applying a single workspace-wide retention setting to everything.

---

## Section 5: DCR Transformations, Logging Like an API, Not a Dump

### Filter before you store. Shape what you ingest. Design your telemetry like you'd design an interface.

Even with resource-specific tables and the right pricing plan, there's a layer of refinement that most teams leave on the table: filtering *within* a log category before the data is stored. Diagnostic Settings let you choose which categories to enable, but they don't let you filter individual records within a category. What you enable, you ingest in full.

Data Collection Rules with ingestion-time transformations close that gap.

### What transformations are and how they fit in

A Data Collection Rule (DCR) is a configuration object that defines how data flows through the Azure Monitor ingestion pipeline. One of its capabilities is running a KQL transformation on incoming data after Azure receives it but before it's written to the Log Analytics table.

Think of it as middleware for your log pipeline. The transformation is a KQL query written against a virtual table called `source`, which represents the incoming data stream. You can filter rows, drop columns, rename fields, redact sensitive values, or discard records entirely. The result of the query is what gets stored.

This is what makes it possible to store only denied Azure Firewall traffic, or only error-level AKS events, or Key Vault operations that involve specific secret names, without any of the discarded data ever touching your storage.

### The workspace transformation DCR: the bridge for Diagnostic Settings

There's a subtlety here worth understanding. Diagnostic Settings don't use DCRs natively, they write directly to the Log Analytics workspace. To apply transformations to data arriving from Diagnostic Settings, you need a specific type of DCR called a workspace transformation DCR.

A workspace transformation DCR is attached to the workspace itself rather than to a specific data source. It intercepts data before it's written to supported tables and applies the transformation you've defined. Each workspace can have only one workspace transformation DCR, but that single DCR can contain transformations for any number of tables.

The practical implication is that you can apply ingestion-time filtering to all of your Diagnostic Settings-sourced data through a single configuration object, without modifying the Diagnostic Settings themselves.

### Real transformation examples

**Drop noisy AKS control plane events.** The `AKSAudit` table captures every Kubernetes API call, including constant read-only operations from system components. Most teams only care about write operations, creates, updates, deletes, and escalations. The read chatter can be safely discarded:

```kql
source
| where Verb !in ("list", "get", "watch")
```

Applied to a busy AKS cluster, this single filter can reduce `AKSAudit` ingestion volume by 60–80% while retaining everything operationally significant.

**Azure Firewall, ingest only denied traffic.** If your primary use case for firewall logs is security monitoring and incident response, allowed traffic is usually noise. Storing only denied connections dramatically reduces volume while preserving the records most likely to matter:

```kql
source
| where Action == "Deny"
| project TimeGenerated, SourceIp, DestinationIp,
          DestinationPort, Protocol, Action, RuleCollection
```

The `project` clause here serves a dual purpose: it drops the `Action == "Allow"` rows and also removes columns not needed for the security use case, reducing the stored record size.

**Redact sensitive fields before storage.** If your application logs contain tokens, session identifiers, or other sensitive values in URL parameters, transformations can sanitize them before they reach storage:

```kql
source
| extend RequestUri = replace_regex(RequestUri, @"token=[^&]+", "token=REDACTED")
| project-away SensitiveHeader
```

This gives you observability without creating a secondary data store full of credentials.

### ⚠️ The 50% filtering charge: the gotcha nobody tells you

This is the most commonly missed nuance around DCR transformations, and it has direct cost implications for anyone using aggressive filtering.

If a transformation drops more than 50% of the incoming data stream, Azure charges a data processing fee for the filtered excess above that threshold. The formula is:

```
Processing charge = [GB dropped by transformation] − ([GB incoming] ÷ 2)
```

To make that concrete: imagine 100 GB of Azure Firewall logs arrive per day. Your transformation keeps only denied traffic, 20 GB. You dropped 80 GB.

```
Processing charge = 80 GB − (100 GB ÷ 2) = 30 GB processing charge
```

You stored 20 GB and paid for 30 GB of processing. You still come out significantly ahead compared to storing 100 GB at Analytics rates, but the savings are smaller than a naive calculation would suggest.

The right response to this isn't to avoid transformations. It's to layer your cost-reduction strategy correctly:

1. **Reduce at the source first.** Choose narrower diagnostic categories in the Diagnostic Setting. Don't enable `AzureFirewallNetworkRule` if all you need is `AzureFirewallApplicationRule`.
2. **Move to the right pricing tier.** Put high-volume tables on Basic or Auxiliary before applying transformations.
3. **Apply transformations as precision filtering.** Use them to further refine the data that remains, not as the primary mechanism for reducing a flood.

There is one important exception: if Microsoft Sentinel is enabled on your workspace, transformation processing charges are waived entirely for Analytics tables. If you're running Sentinel, aggressive filtering via DCR transformations has no processing cost penalty.

### Transformation constraints to know before you start

Not all Log Analytics tables support ingestion-time transformations, the Azure Monitor table reference documents which ones do. Azure Firewall transformations, specifically, only work with resource-specific logs, if you're still using `AzureDiagnostics` mode for firewall logs, DCR transformations aren't available to you. This is yet another reason to complete the migration to resource-specific tables before investing in transformation logic.

Changes to transformation rules take up to 30 minutes to propagate. The portal wizard for creating transformations shows a preview of the transformation applied to sample data before you save, use it. Syntax errors that would silently drop records are caught at this stage, not after the fact.

---

## Section 6: The Layered Strategy, Defense in Depth for Log Costs

### Six levers, applied in the right order.

The tools are available. Resource-specific tables, tiered pricing plans, DCR transformations, table-level RBAC, Azure Monitor has evolved significantly, and teams that use the full toolkit pay a fraction of what teams using only default configurations pay for equivalent coverage.

The challenge is that the tools need to be applied in the right order. Transformations applied on top of `AzureDiagnostics` are less powerful than transformations on resource-specific tables. Pricing tier changes without category selection discipline don't capture the full savings. Doing everything at once is confusing. Here's the sequence that works.

**Layer 1, Audit before you expand.** Before enabling new Diagnostic Settings or adding new resources to existing policies, run the billing breakdown query from Section 3. Know where your current ingestion costs are coming from. Establish a baseline. This takes 10 minutes and prevents the most common form of cost surprise.

**Layer 2, Select categories with intent.** For each resource type, identify the specific categories that serve an actual use case: alerting, dashboards, compliance retention, incident forensics. Enable those. Don't tick the rest. If you're not sure what a category contains, it shouldn't be enabled yet.

**Layer 3, Use resource-specific mode for every new Diagnostic Setting.** When creating or updating a Diagnostic Setting, toggle "Resource specific" wherever the option appears. Run both modes during the migration window using `union` queries. Once historical data ages out, simplify to the resource-specific table only.

**Layer 4, Assign table plans to match access patterns.** After migrating to resource-specific tables, assign each table to the plan that reflects how you actually use it. Tables backing active dashboards and alert rules belong on Analytics. Tables used for debugging belong on Basic. High-volume tables used for compliance archival or security hunting belong on Auxiliary.

**Layer 5, Apply DCR transformations for precision filtering.** With the right pricing tier in place, use workspace transformation DCRs to further reduce noise within each table. Drop read-only AKS chatter. Filter allowed firewall traffic. Strip unnecessary columns. Be aware of the 50% threshold and don't use transformations as a substitute for upstream category selection.

**Layer 6, Monitor your monitoring.** Set a daily ingestion cap as a circuit breaker during rollout. Then set up an alert on the `Usage` table to proactively notify you when any specific table's ingestion exceeds an expected threshold:

```kql
// Ingestion spike detection, run as a scheduled alert
Usage
| where TimeGenerated > ago(1d)
| summarize GB = sum(Quantity) / 1000 by DataType
| where GB > 10  // adjust per-table thresholds based on your baseline
| order by GB desc
```

An alert here catches configuration mistakes, a new policy enabling an unexpected category, a workload behaving differently than anticipated, a third-party solution logging more than expected, before they compound into a billing surprise at month end.

---

## Conclusion: Logging Is Architecture, Not a Toggle

There's a useful way to think about the problem that unifies everything in this post.

A well-designed API has a contract. It exposes what callers need, hides what they don't, and is intentionally shaped rather than just exposed. You don't build an API by dumping your entire database schema as a JSON endpoint and hoping the caller knows what to ignore.

`AzureDiagnostics` is the opposite of that. It's a monolithic endpoint that accepts everything from every service, enforces no schema contract, charges by volume regardless of value, and requires the consumer to know what to ignore before they can use it. It accumulated rather than being designed.

The shift to resource-specific tables, tiered pricing plans, and DCR transformations represents Azure finally giving teams the tools to treat telemetry as a first-class design concern rather than an afterthought. The table is purpose-built. The schema is intentional. The access pattern determines the price. The ingestion pipeline can filter before it stores.

These tools don't reduce observability, they improve it. Noise is not the same as signal. A table that contains only denied firewall traffic is more useful for security monitoring than one that contains every connection the firewall ever processed. A well-transformed `AKSAudit` table that captures write operations and discards read-only chatter is easier to alert on, faster to query, and cheaper to retain.

The teams that will be ahead on observability costs over the next few years aren't the ones who log less. They're the ones who log intentionally, who treat their telemetry pipeline with the same design discipline they'd bring to any other part of their system.

The question was never whether to turn on logging. It's whether you've designed what happens after you do.

---

## Appendix: KQL Reference

### Query 1, Audit your current ingestion costs by source
```kql
AzureDiagnostics
| where _IsBillable == true
| where TimeGenerated > now(-31d)
| summarize BillableDataGB = round(sum(_BilledSize)/(1024*1024*1024), 2)
    by ResourceProvider, ResourceType, Category
| sort by BillableDataGB desc
```

### Query 2, Estimate log entry sizes for cost forecasting
```kql
AzureDiagnostics
| where Category == "AuditEvent"
| where TimeGenerated > ago(30d)
| summarize percentile(_BilledSize, 95), round(avg(_BilledSize))
```

### Query 3, Ingestion spike alert
```kql
Usage
| where TimeGenerated > ago(1d)
| summarize GB = sum(Quantity) / 1000 by DataType
| where GB > 10
| order by GB desc
```

### Query 4, AzureDiagnostics migration query (covers both old and new tables)
```kql
union AzureDiagnostics, KeyVaultAuditEvent
| where (ResourceType == "KEYVAULTS" or Type == "KeyVaultAuditEvent")
| where TimeGenerated > ago(90d)
| where OperationName == "SecretGet"
```

### Transformation 1, Drop noisy AKS read operations
```kql
source
| where Verb !in ("list", "get", "watch")
```

### Transformation 2, Azure Firewall: denied traffic only
```kql
source
| where Action == "Deny"
| project TimeGenerated, SourceIp, DestinationIp,
          DestinationPort, Protocol, Action, RuleCollection
```

### Transformation 3, Redact sensitive URL parameters
```kql
source
| extend RequestUri = replace_regex(RequestUri, @"token=[^&]+", "token=REDACTED")
| project-away SensitiveHeader
```

---

*Pricing figures are approximate for the East US region and reflect rates current as of early 2026. Always verify against the Azure pricing calculator for your specific region and usage patterns.*

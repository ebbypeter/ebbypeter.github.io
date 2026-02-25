---
title: "Detection Is Not Protection: What Azure WAF Detection Mode Actually Does (and Doesn't)"
draft: false
date: 2026-03-03
tags: 
  - "azure"
  - "waf"
  - "security"
  - "appsec"
  - "cloud-security"
  - "azure-front-door"
  - "application-gateway"
categories: 
  - "Security" 
  - "Azure"
summary: "Most teams think a WAF in Detection mode is partially protecting them. It isn't. Here's what actually happens to requests, why the logs actively mislead, and how organisations end up stuck in Detection mode indefinitely without noticing."
---

*Your WAF is enabled. Your dashboard is green. And every attack hitting your application is going straight through to the backend. Why?*

---

Somewhere in your Azure environment, there is probably a WAF policy sitting in Detection mode. Not because someone made that call explicitly. Not because there's an active incident requiring passive observation. Just because that's the default, tuning takes a while, and nothing ever forced the conversation to a close.

Your security posture dashboard says WAF is enabled. Technically, that's true.

Detection mode is not a weaker version of protection. It is the complete absence of protection, with a very convincing paper trail attached to it. This post is about understanding exactly what that means at the request level, why the logs make it look like more than it is, and why teams drift into permanent Detection mode without ever deciding to.

---

## What happens to a request

The WAF engine in Detection mode does its job. A request arrives, the engine evaluates it against the full managed rule set, and an anomaly score accumulates. Azure WAF uses OWASP anomaly scoring by default, each matched rule contributes to a running total based on severity. A Critical rule match contributes 5 points. A single SQL injection hit clears the threshold.

Here is where the modes diverge.

In Prevention mode, a score of 5 or above triggers a block. The WAF returns a 403, the connection closes, and the request never reaches your backend. In Detection mode, the same score triggers a log entry. The WAF writes the event, notes the matched rules, records the anomaly score, and then forwards the request to your backend unchanged. The attacker gets through. Whatever they were attempting, they succeeded.

The engine ran. The inspection happened. The WAF just didn't do anything with the result.

There is a narrow exception worth knowing about: a small set of mandatory rules, requests that fail body parsing, requests that exceed the configured body size limit, will cause a block even in Detection mode. These rules can't be disabled. But they're plumbing-level behaviours, not security controls. They don't protect against SQL injection, XSS, path traversal, or anything in the OWASP Top 10. The exception exists, it's just not relevant to any real attack surface.

The one-line summary: the WAF engine runs, inspects everything, accumulates the score, and then steps aside.

---

## How Detection mode becomes permanent

Every new WAF policy in the Azure portal defaults to Detection mode. That's intentional, Microsoft's own guidance says to start in Detection, observe real traffic, tune out false positives, then switch to Prevention. It's sensible advice for initial deployment.

The problem is what happens after.

Microsoft's FAQ on enabling WAF for Front Door describes a multi-step tuning process that "might take several weeks." What it doesn't mention is that there's no timer. No alert. No Azure Policy gate that fires when a WAF sits in Detection mode past its welcome. The tuning process has no formal exit criteria, which means it has no exit. Teams tune for a while, see alert volume drop, feel like progress is being made, and then move on to the next thing. Nobody formally closes the loop. Detection mode becomes the permanent state not through deliberate choice, but through the absence of one.

There's a specific cognitive trap buried in this: in Detection mode, a quiet WAF proves nothing. If you're not seeing many threat alerts, you cannot distinguish between "attacks aren't happening" and "attacks are happening and going straight through." The WAF will never surface a blocked request, because it never blocks one. The silence is not a signal. It's just silence.

Detection mode also trains teams to ignore WAF alerts, which compounds the problem in a non-obvious way. Because everything gets logged and nothing gets blocked, the alerts generated in Detection mode have no operational consequence. Over weeks and months, teams tune the alerting down, tighten the response criteria, or just stop routing WAF alerts to anyone who acts on them. When you eventually do switch to Prevention mode and need that alerting channel to catch false positives causing user impact, the signal has already been devalued. You've spent months conditioning your team to treat WAF events as background noise.

---

## The log that actively misleads

This is the part that causes the most concrete operational damage, and it almost never gets explained clearly.

When a request matches a WAF rule in Detection mode, the log entry looks like this:

```
action_s     : Block
policyMode_s : Detection
ruleName_s   : DefaultRuleSet-1.0-SQLI-942430
```

`action_s = "Block"`. A reasonable engineer reads that and concludes the request was blocked. It wasn't. In Detection mode, `action_s = "Block"` means "this is what would have happened in Prevention mode." The request went to your backend. Microsoft's own documentation includes a sample log table showing exactly this combination, `Action = Block`, `Mode = Detection`, as normal expected output. It's documented, just not explained clearly enough to prevent the misread.

The practical consequence: teams write KQL queries to verify their WAF is doing its job.

```kql
AzureDiagnostics
| where Category == "FrontdoorWebApplicationFirewallLog"
| where action_s == "Block"
```

In Detection mode, this query returns results. Every single result in that output represents a request that reached your backend. The query is syntactically correct. The interpretation is completely wrong.

The fix is straightforward, but only if you know the trap is there:

```kql
AzureDiagnostics
| where Category == "FrontdoorWebApplicationFirewallLog"
| project TimeGenerated,
          RuleID      = ruleName_s,
          Action      = action_s,
          Mode        = policyMode_s,
          ClientIP    = clientIp_s,
          URI         = requestUri_s
| where Action == "Block" and Mode == "Prevention"
```

Always project `policyMode_s` and filter on it explicitly. A `Block` action in `Prevention` mode means something was stopped. A `Block` action in `Detection` mode means something was observed and passed through. They are not the same event and should never be counted together in a dashboard or fed into the same alert.

There's a second layer to this for anyone running modern rule sets. With anomaly scoring (the default for CRS 3.x and DRS 2.1+), individual rule hits log as `action_s = "Matched"`, not "Block." The terminal rule, 949110, fires when the accumulated score hits the threshold and logs as "Blocked" in Prevention mode or "Detected" in Detection mode. So for a single request that triggers multiple rules, you'll see a mix of "Matched" entries and one terminal entry, and the terminal entry is the only one that tells you whether anything actually happened. SIEM correlation rules built on `action_s == "Block"` without the mode filter can end up capturing intermediate match events, not terminal decisions, on top of the mode confusion.

If you've built incident response runbooks, Sentinel analytics rules, or security dashboards on WAF logs without explicitly filtering on `policyMode_s`, there is a real possibility that your "blocked attacks" count includes a significant number of attacks that weren't blocked.

---

## The hybrid protection illusion

Teams that have done some WAF work often push back here: "we have custom rules that block, rate limiting, IP denylists. We're not completely unprotected."

This is partially true. It's also more dangerous than no protection story at all.

WAF mode governs managed rule behaviour. Custom rules with a Block action operate on their own terms, a rate limit rule that fires at 1,000 requests per minute from a single IP will still block in Detection mode. Your IP denylist custom rule will still block the IPs you've explicitly listed.

What those custom rules don't touch is the entire managed rule surface: the OWASP Core Rule Set, the Microsoft Threat Intelligence rules, the DRS rules covering SQL injection, XSS, path traversal, protocol violations. All of that is Detection-only. An attacker who stays under your rate limit and isn't on your denylist, which describes most attack traffic, passes straight through every OWASP control you thought you had.

The partial protection story is the dangerous version because it closes the conversation. "We have blocking rules" is true and misleading in the same sentence. The question isn't whether any rules block. It's whether the rules covering the attack surface you care about block. In Detection mode, they don't.

---

## Why nobody catches it

The failure pattern is consistent enough that the mechanisms are worth naming.

Security posture dashboards report resource existence. "WAF: Enabled" is a factual claim about whether a WAF policy exists. It says nothing about mode, nothing about whether Diagnostic Settings are configured (without which Detection mode produces no logs at all, making it completely invisible), and nothing about whether the WAF is actually on the traffic path between the internet and your origin. Most compliance reviews stop at "does a WAF exist." Nobody runs the query.

PCI DSS Requirement 6.4 requires WAF protection for internet-facing applications. In practice, most assessors confirm WAF presence. Whether the WAF is actually blocking anything is a different question, and it's usually not the one being asked. A Detection-mode WAF may satisfy the checkbox while delivering none of the protection the control was designed to require.

The origin exposure issue makes all of this worse. If your Application Gateway or Front Door origin isn't locked down to only accept traffic from those services, which is a separate step, not automatic, then the WAF can be bypassed entirely by anyone who finds the origin IP. Detection mode creates just enough visible activity to feel like security is happening, which delays teams from doing the harder, more impactful work of actually restricting origin access. The WAF isn't just failing to protect. It's occupying the team's attention while the real attack surface stays open.

Finally: Azure has built-in Policy definitions specifically designed to enforce WAF Prevention mode. Two of them, in fact, one for Application Gateway, one for Front Door. They can audit or deny WAF policies not running in Prevention mode. They're sitting in the built-in policy library, unused, in most environments. The enforcement mechanism that would close the loop exists. It just hasn't been deployed.

---

## Getting out

If you suspect your environment has production WAFs in Detection mode, start here.

Find out what you're actually dealing with. This Azure Resource Graph query doesn't depend on log data being configured, it goes straight to the resource properties:

```kql
Resources
| where type == "microsoft.network/frontdoorwebapplicationfirewallpolicies"
       or type == "microsoft.network/applicationgatewaywebapplicationfirewallpolicies"
| project name, resourceGroup, subscriptionId,
          mode = properties.policySettings.mode
| where mode != "Prevention"
```

That gives you the full inventory across subscriptions of WAF policies not in Prevention mode. For environments where logging is configured and you want a log-based view:

```kql
AzureDiagnostics
| where Category in ("FrontdoorWebApplicationFirewallLog", "ApplicationGatewayFirewallLog")
| summarize count() by policyMode_s, _ResourceId
```

Once you know where you are, set a hard tuning deadline. Two to four weeks for a stable application with representative traffic. Not "until it feels right", a date. Review logs daily during that window, not weekly. Define your exclusions as code in Bicep or Terraform so they survive rule set version upgrades without manual re-entry. The Microsoft recommendation to spend "several weeks" in Detection mode is not an invitation to treat it as indefinite.

When you flip to Prevention mode, expect friction. Some legitimate requests will get blocked. That is not a sign that something went wrong, it's the expected output of a WAF that's actually doing its job. Have a runbook ready before you flip: the KQL query to find what's being blocked, the process for creating a targeted exclusion, and the ownership chain for approving it quickly. The first week in Prevention mode is the hardest. It does not stay that way.

Lock your origin. Regardless of WAF mode, if your origin accepts traffic from anywhere other than your Application Gateway or Front Door, the WAF is bypassable. For Front Door this means Private Link or service tag-based restrictions. For Application Gateway it means no publicly reachable addresses on the backend subnet. Do this at the same time as the Prevention mode switch, not after.

Then deploy the Azure Policy built-ins to enforce Prevention mode going forward, audit first, then deny for new deployments. Wire them into your compliance dashboard. "WAF: Enabled" should mean something.

---

## The name is the problem

"Detection" implies something was caught. Nothing was caught.

If this mode had been called "Logging mode" or "Passive mode," teams would treat it as what it is: a temporary diagnostic state with a clear exit. The word "detection" creates a mental model where threats are being identified and presumably managed. That model is wrong. The detection is not connected to any protective action. It's forensic data generation with a security-sounding label, and the label does real damage.

What Detection mode actually is: a logging pipeline that happens to sit in the request path. The WAF inspects traffic, matches rules, accumulates scores, and writes events. Then it waves everything through.

If your WAF has been in Detection mode for more than a month in production, you don't have a WAF. You have detailed logs of exactly what attacks are reaching your application, and nothing stopping any of them.

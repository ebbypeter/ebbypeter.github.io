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

## What actually happens to a request

The WAF engine in Detection mode does its job. A request arrives, the engine inspects it against the full managed rule set, and an anomaly score accumulates. Azure WAF uses OWASP anomaly scoring by default — each matched rule contributes based on severity. A Critical rule match contributes 5 points. One SQL injection hit clears the threshold.

Here is where the modes split.

In Prevention mode, a score of 5 or above triggers a block. The WAF returns a 403, closes the connection, and the request never reaches your backend. In Detection mode, the same score triggers a log entry. The engine writes the event, records the matched rules, and then forwards the request to your application unchanged. The attacker gets through. Whatever they were attempting, they succeeded.

The engine ran. The inspection happened. The WAF just didn't act on any of it.

There is a narrow exception: a small set of mandatory rules covering body parsing failures and size limit violations will block even in Detection mode. But those are plumbing behaviours, not security controls. They don't protect against anything in the OWASP Top 10. The exception exists; it just doesn't matter for any real attack surface.

![Detection vs Prevention](/images/posts/waf_prevention_detection.svg)

---

## The default that sets the trap

Every new WAF policy in the Azure portal defaults to Detection mode. This is intentional. Microsoft's guidance says start in Detection mode, observe real traffic, tune out false positives, then switch to Prevention. That's sound advice for initial deployment. The problem is what happens after.

Microsoft's FAQ on enabling WAF for Front Door acknowledges the tuning process "might take several weeks." What it doesn't tell you is that there's no timer, no alert, and no Azure Policy gate that fires when a WAF sits in Detection mode past its welcome. The tuning loop has no formal exit criteria, which in practice means it has no exit.

The pattern repeats across organisations: team deploys WAF in Detection mode, reviews alerts, creates a few exclusions, sees alert volume drop, feels like progress is being made, and moves on to the next priority. Nobody closes the loop. Detection mode becomes permanent not through deliberate choice, but through the absence of one.

What makes this hard to catch is that Detection mode offers no signal that attacks are succeeding. A quiet Detection-mode WAF and an effective Prevention-mode WAF look identical in one metric: zero user impact from blocked traffic. You cannot distinguish "no attacks are happening" from "attacks are happening and going straight through." The silence proves nothing.

Detection mode also trains teams to ignore WAF alerts, which creates a second-order problem. Because everything gets logged and nothing ever blocks, alerts in Detection mode have no operational consequence. Over months, teams tune the alerting down or stop routing WAF events to anyone who acts on them. When you eventually flip to Prevention and need that channel to surface false positives causing real user impact, you've spent months conditioning your team to treat WAF alerts as noise.

---

## The log that misleads

This is where the most concrete operational damage happens.

When a request matches a WAF rule in Detection mode, the log entry looks like this:

```
action_s     : Block
policyMode_s : Detection
ruleName_s   : DefaultRuleSet-1.0-SQLI-942430
```

`action_s = "Block"`. A reasonable engineer reads that and concludes the request was blocked. It wasn't. In Detection mode, `action_s = "Block"` means "this is what would have happened in Prevention mode." The request went to your backend. Microsoft's own documentation includes a sample log table showing exactly this combination as normal expected output. It's documented. It's just not explained with enough emphasis to prevent the misread in practice.

The consequence is queries like this:

```kql
AzureDiagnostics
| where Category == "FrontdoorWebApplicationFirewallLog"
| where action_s == "Block"
```

In Detection mode, this returns results. Every single result represents a request that reached your application. The query is syntactically correct. The interpretation is completely wrong.

The fix is straightforward once you know the trap:

```kql
AzureDiagnostics
| where Category == "FrontdoorWebApplicationFirewallLog"
| project TimeGenerated,
          RuleID   = ruleName_s,
          Action   = action_s,
          Mode     = policyMode_s,
          ClientIP = clientIp_s,
          URI      = requestUri_s
| where Action == "Block" and Mode == "Prevention"
```

Always project `policyMode_s` and filter on it explicitly. A `Block` in `Prevention` mode means something was stopped. A `Block` in `Detection` mode means something was observed and waved through. They are not the same event and should never be counted together in a dashboard or alert rule.

If your SIEM correlation rules, Sentinel analytics, or security dashboards were built on WAF logs without filtering on `policyMode_s`, your "blocked attacks" count almost certainly includes attacks that weren't blocked.

---

## Why nobody catches it

Security posture tooling reports resource existence. "WAF: Enabled" is a claim about whether a WAF policy is associated with a resource. It says nothing about mode, nothing about whether Diagnostic Settings are configured (without which Detection mode is completely invisible), and nothing about whether the WAF is actually on the traffic path. Most compliance reviews stop at existence. Nobody asks `policyMode_s`.

PCI DSS Requirement 6.4 mandates WAF protection for internet-facing applications. In most assessments, that translates to: is there a WAF? A Detection-mode WAF satisfies the question. Whether it's blocking anything is a different question, and it usually isn't asked.

There's also an enforcement mechanism that almost nobody has deployed. Azure ships two built-in Policy definitions specifically for this: one requiring a specified WAF mode for Application Gateway, one for Front Door. Both can audit or deny WAF policies not running in Prevention mode. They're sitting in the built-in policy library unused in most environments. The tool to close this gap programmatically exists. It just hasn't been picked up.

---

## Getting out

Start with an inventory that doesn't depend on logs being configured. This Resource Graph query goes directly to resource properties and covers all subscriptions:

```kql
Resources
| where type == "microsoft.network/frontdoorwebapplicationfirewallpolicies"
       or type == "microsoft.network/applicationgatewaywebapplicationfirewallpolicies"
| project name, resourceGroup, subscriptionId,
          mode = properties.policySettings.mode
| where mode != "Prevention"
```

That tells you exactly which WAF policies aren't in Prevention mode, without needing logs to be flowing.

Once you know the scope, set a hard tuning deadline. Two to four weeks for a stable application with representative traffic, not "until it feels right." Review logs daily during that window. Define exclusions as code in Bicep or Terraform so they survive managed rule set version upgrades. The Microsoft recommendation to spend "several weeks" in Detection mode applies to initial tuning. It is not an invitation to treat Detection as an indefinite state.

When you flip to Prevention, expect some legitimate requests to get blocked. That's not a sign something went wrong. It's a WAF that's actually enforcing rules. Have a runbook ready: the query to surface what's being blocked, the process for creating a targeted exclusion, and an owner who can approve it quickly. The first week is the hardest part.

Then deploy the Azure Policy built-ins. Audit first, deny for new deployments once you understand the scope. Wire it into your compliance dashboard so "WAF: Enabled" actually means something.

---

## The name is the problem

"Detection" implies something was caught. Nothing was caught.

If this mode were called "Logging mode," engineers would treat it as what it is: a temporary diagnostic state with a clear exit. The word "detection" suggests threats are being identified and handled. That model is wrong. The detection is not connected to any protective action. It's forensic data generation with a security-sounding label, and the label does real damage in the gap between deployment and a tuning deadline that never arrives.

If your WAF has been in Detection mode for more than a month in production, you don't have a WAF protecting your application. You have detailed logs of exactly which attacks are hitting you, and nothing stopping any of them. That's worth fixing today.

What's your WAF policy mode set to right now? It takes about thirty seconds to check, and the answer might surprise you.
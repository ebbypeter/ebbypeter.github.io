---
title: "Your Alerts Are a Product. They're Just a Bad One."
draft: false
date: 2026-02-24
tags:
  - alerting
  - sre
  - observability
  - on-call
  - incident-response
  - slo
  - platform-engineering
categories:
  - Azure
  - Systems Design
  - Site Reliability Engineering
summary: "Alert fatigue isn't a people problem, it's a product design failure. Your on-call engineers are the users. Here's why noisy alerts are biologically inevitable under bad design, and what treating alerting as a product actually looks like."
---

*Your monitoring system is sending 2,000 alerts a week. Three percent of them matter. The other 97% aren't just noise, they're training your engineers to ignore you.*

---
It's 3am. The phone goes off. PagerDuty. An engineer rolls over, squints at the screen.

"Disk usage at 76% on web-03."

They've seen this alert before. Many times. It's fired on Tuesday nights for six months. Nothing ever happens. The disk doesn't fill. The service doesn't degrade. It just... fires. So they acknowledge it, put the phone face-down, and go back to sleep.

Two hours later, a real P1 lands. Authentication service down. Revenue impact starting immediately. The same engineer gets paged again, but now they're deep in broken sleep, groggy, running on the wrong kind of adrenaline. They take eleven minutes to acknowledge. The postmortem later will note: "delayed response contributed to extended incident duration."

The action item: "improve on-call culture and ensure engineers treat all alerts with appropriate urgency."

Wrong diagnosis. Completely, dangerously wrong.

That engineer didn't fail. They responded exactly as they'd been trained to respond, not by their manager, but by the system itself. Six months of false positives is six months of training data. The lesson delivered, repeatedly, reliably: *this alert doesn't matter.* The engineer learned it. Of course they did.

The problem isn't the engineer. The problem is what got shipped to them.

---

Industry benchmarks put the average at over 2,000 alerts per week for a typical DevOps team. About 3% require immediate action. That means 97% of what's paging your engineers is noise, and your engineers know it, even if they can't articulate it. They know it in their bones, which is precisely the problem.

Alert fatigue isn't a motivation problem. It's what happens when you design a bad product and then blame the users when they stop engaging with it.

---

## The Biology of Ignoring Your Pages

Here's something most SRE content skips: the reason alert fatigue is so hard to fix isn't organizational. It's neurological.

Habituation is one of the most fundamental, universal forms of learning in the animal kingdom. The mechanism is simple: expose an organism to a repeated stimulus that carries no consequence, and its response to that stimulus decays. Progressively. Predictably. This isn't laziness or inattention, it's the brain operating exactly as designed, filtering out irrelevant signals so it can allocate cognitive resources to things that actually matter.

Every single vertebrate on the planet does this. Including your on-call engineer.

The alerting implications are worse than most people realize, because habituation has some specific properties that make noisy alert systems particularly destructive.

**Frequency accelerates it.** The more often a stimulus fires without consequence, the faster the response decays. An alert that fires ten times a day, every day, trains the brain to dismiss it faster than one that fires once a week. Your noisiest alerts are doing the most damage.

**Stimulus generalization spreads it.** Habituation doesn't stay contained to the specific stimulus that caused it. It generalizes to similar stimuli. A noisy CPU alert trains the brain to dismiss *all* CPU alerts, including the ones that matter. You don't just lose signal on the bad alerts. You lose signal on the good ones that happen to look like the bad ones.

**The decrement goes below zero.** The response doesn't just drop to neutral indifference. Engineers develop active suppression behaviors, auto-acknowledging without reading, muting the Slack channel, building muscle memory that routes the alert straight to dismissal. This isn't negligence. It's the brain optimizing for a world where that alert has never meant anything.

And then there's the cognitive load problem on top of all this. Decision-making capacity is finite and depletable. A noisy shift, 30, 40, 50 low-quality alert decisions before the real incident even fires, means the engineer who finally faces the P1 is running on empty. Slower. More error-prone. More likely to miss a second failure lurking behind the first. You're not just training engineers to ignore alerts. You're structurally impairing their ability to respond when it matters.

This is a reliability problem. Not a wellness problem, not an HR problem, a direct, measurable impact on MTTR and incident quality.

So when the postmortem says "engineer failed to respond promptly," it's identifying the symptom and calling it the disease.

---

## What's Actually Broken

Most conversations about alert fatigue land on "fix your thresholds" and stop there. Raise the threshold so it fires less. Add an evaluation window so transient spikes don't page. Deduplicate related alerts.

All useful. None sufficient. Because the threshold problem is a symptom of a more fundamental failure: **nobody is building alerting as a product**.

Think about what product development actually involves. You have users. You understand their needs, what context they need, at what time, to make what decision. You have success metrics. You get feedback. You iterate. You retire things that don't work. You treat a bad user experience as an engineering problem, not a user discipline problem.

Now think about how most organizations build their alerting stack.

Someone deploys a new service. They copy monitoring configuration from a template, or from the last service they built, or from something they found on the internet. They set thresholds based on what feels right. They hook it into PagerDuty and move on. Nobody reviews whether those alerts are useful six months later. Nobody tracks how often they fire without action. Nobody retires them when the service changes. They just accumulate, layers of decisions made by people who have since moved on, baked into a system that grows noisier over time by default.

There's no ownership. No feedback loop. No success metric. No lifecycle.

Imagine shipping a mobile app this way. You release features, never look at engagement data, never read user feedback, never deprecate the things that aren't working, and when users stop opening the app, you put "improve user motivation" in the quarterly OKRs. Nobody would accept this. But it's exactly how most alerting systems are run.

The users of your alerting system are your on-call engineers. The product you're shipping them is the alert, its content, its timing, the context it carries, the workflow it kicks off. When that product is bad, they stop engaging with it. This is rational behavior. The failure is in the design, not the user.

---

## What Good Alert Design Actually Looks Like

### The only mandatory property

An alert should fire if and only if two conditions are both true: a human action is required urgently, and the system can't take that action itself.

Both conditions. Both.

"Urgently" is doing real work in that sentence. A problem that can wait until morning is not an alert, it's a ticket. A problem the system can handle automatically is not an alert, it's a runbook waiting to be written. Paging a human at 3am for something that isn't urgent, or that doesn't require a human, is a design defect. Full stop.

CPU at 85% for a single spike: what's the expected human action? Scale horizontally? Investigate? Restart a process? If you can't answer that question precisely, you don't have an alert, you have a metric with a notification attached.

Error rate burning your SLO at 14 times the sustainable rate over the past hour, meaning you'll exhaust your monthly error budget in two days: that's an alert. Clear urgency. Clear human action required. Clear stakes.

The discipline of forcing this question, *what do we expect a human to do, right now, that the system cannot do itself?*, eliminates a significant fraction of alert noise before you even get to threshold tuning.

### Context is UX

A bare metric value is a terrible interface for someone who's just been woken up.

"Memory usage: 91%" tells an on-call engineer almost nothing useful. Which service? What's the blast radius? What typically causes this? What's step one? They have to context-switch across three dashboards, half-asleep, before they can even form a hypothesis, and every minute of that context-switching is a minute your service is degraded.

Compare that to an alert that arrives with: what's happening ("payment service memory pressure, likely leak in connection pool"), what's probably causing it (top two hypotheses based on past incidents), what to check first (direct runbook link, not a wiki link three clicks from the right page), and what's *not* affected (checkout flow is degraded; order history is unaffected).

Same underlying event. Completely different cognitive experience.

This isn't about verbose alerts. It's about information density at the moment of highest cognitive stress. The engineer shouldn't have to investigate to understand the situation, they should be able to act from the alert itself, or at most follow one link to get there.

### Signal architecture: what SLOs actually solve

Threshold-based alerting is fundamentally cause-based. You're watching a metric, waiting for it to cross a line. The problem is that most metrics don't map cleanly to user impact. CPU at 90% might mean nothing if the service is still responding fast. A small error rate spike might mean everything if it's hitting checkout.

SLO-based alerting shifts to symptom-based. You're not asking "has this metric exceeded a threshold?" You're asking "are we consuming our reliability budget at a rate that threatens the commitments we've made to users?"

Burn rate is the key concept here. If your service has a 99.9% availability SLO over 30 days, your total error budget is about 43 minutes. A burn rate of 1 means you're consuming that budget at exactly the sustainable rate. A burn rate of 14 means you'll be out of budget in roughly 50 hours. That's a calibrated, actionable severity signal, far more useful than "error rate exceeded 1%."

The multi-window approach Google's SRE workbook recommends is worth understanding properly. You use two windows simultaneously: a longer window (say, one hour) to confirm the issue is significant, and a shorter window (five minutes) to confirm it's still happening. Both must indicate elevated burn rate for the alert to fire. This eliminates the spike-and-recover false positives that account for a huge chunk of noisy pages, a brief error spike that resolves before anyone can even look at it.

But let's be honest about the limitations. SLO-based alerting requires SLIs, which require instrumentation, which most teams don't have for everything that isn't an HTTP service. Background jobs, async message queues, batch pipelines, internal tooling, these rarely have meaningful SLOs, and forcing an SLO framework onto them before the instrumentation is in place creates a different kind of noise problem. For these systems, good threshold hygiene with evaluation windows is still the right approach. The SLO migration is worth pursuing, but it's a journey, not a toggle.

### The tiered response model

Most teams operate with two modes: alert fires, or nothing happens. The alert fires, someone gets paged. That's the whole model.

You need three tiers.

**Page**: requires human action right now, cannot wait, cannot be automated. This is what wakes people up. The bar should be high.

**Ticket**: real problem, needs addressing, but business hours are fine. Slow error budget burn that'll take days to become critical. Degraded non-critical path. This goes into the backlog with appropriate priority. Not a page.

**Log**: informational. Feeds dashboards, trend analysis, capacity planning. No action required. Nobody gets notified directly.

The ticket tier is where most organizations are leaving tremendous noise reduction on the table. They have no middle option between "page the engineer" and "nothing." So everything that needs attention eventually becomes a page, because that's the only notification mechanism available.

### Escalation paths as design decisions

The right alert going to the wrong person is a design failure. Not a rotation problem. Not a culture problem. A design failure with a design solution.

Escalation trees need to reflect actual ownership and expertise, not org chart hierarchy. Who actually understands this service? Who can make a call on whether to roll back, scale out, or escalate further? These questions have specific answers that should be encoded in the escalation path, and they should be reviewed every time service ownership changes.

Escalation timing matters too. Immediate escalation of everything creates a different failure mode: primary responders who never build familiarity with the system because they always get rescued too fast. The design should give L1 real time and real tools to diagnose and resolve before escalation kicks in. That builds competence and reduces dependency on escalation as a crutch.

---

## The Lifecycle Problem

Even if you build good alerts today, the system rots.

Services change. Traffic patterns shift. What was a meaningful threshold six months ago is noise now. The original alert author has moved to a different team. Nobody remembers why the alert exists. It just fires, and people ignore it, and over time the noise floor rises.

Alert debt is real, and it accumulates just like technical debt, invisibly, gradually, until suddenly the system is unmaintainable and nobody knows where to start.

A product approach demands a lifecycle.

Every alert needs an owner, not a team, a person who is responsible for its accuracy and actionability. Every alert needs to be reviewed regularly: is this still firing on the right conditions? Is it still actionable? Does the runbook still reflect reality?

Post-incident alerting review is as important as the technical postmortem, and most teams skip it entirely. After every significant incident, three questions: Did the right alert fire? Did anything fire that didn't need to? Did we miss a signal we should have had? These three questions, answered honestly and tracked over time, are the feedback loop that actually improves signal quality. Without them, you're flying blind.

And alerts need to be retired. This sounds obvious, but most alerting systems have no mechanism for it, no culture around it, and no incentive to do it. An alert that consistently false-positives should be fixed or deleted. There is no third option. Leaving it in place is an active choice to keep training your engineers to dismiss it.

If you want to measure where you are, track these: actionable alert rate (what percentage of pages in the last 30 days required a real human action?), false positive rate, after-hours pages per engineer per week, and mean time to acknowledge as a proxy for engagement. These are the product metrics of your alerting system. If you're not tracking them, you're not managing the system, you're just hoping it gets better on its own.

---

## The Part Nobody Wants to Say

Design framing is useful precisely because it depersonalizes the problem. "This alert has a 90% false positive rate over the past 90 days" is a design critique. It's fixable. It doesn't require accusing anyone of not caring, not being rigorous, not taking on-call seriously. It opens a conversation that blame closes.

But design can only go so far.

If three engineers are covering 47 services across a weekend rotation, no alert design solves the load problem. That's a staffing problem. Good alerting will make it more bearable; it won't make it sustainable.

And if leadership is reading reduced page volume as reduced team effort, if the implicit culture is that high pager load proves you're working hard, and quiet shifts prove you're slacking, no framework fixes that. That's a management problem. The conversation to have there isn't about alert design. It's about what the metrics actually measure and what they don't.

The strongest argument for fixing your alerting system, the one that actually moves budget and leadership attention, isn't engineer wellbeing. It's reliability. Fatigued engineers have higher MTTR. Habituated engineers miss critical alerts. The on-call experience is directly connected to your SLA performance, not as a soft HR concern, but as a mechanical fact. Every missed alert, every delayed acknowledgment, every decision made on depleted cognition extends your incidents. If you want to improve your reliability numbers, start by measuring how much signal your engineers are actually receiving.

---

The tools aren't the problem. PagerDuty, Datadog, Prometheus, Grafana, these are capable platforms. They can't fix a design problem. They're the canvas.

What fixes the problem is treating alerting with the same engineering rigor you bring to the systems being monitored. Ownership. Feedback loops. Iteration. Retirement. Asking hard questions about every alert that fires: did this require a human? Did that human have what they needed to act? Was this the right human?

The test is simple. If every alert that fires tonight requires a real human action that the system cannot take itself, and every engineer who gets paged has enough context to act without an investigation, you're building good alerting.

If not, you already know what the product review would say.

The engineers on your rotation aren't the last line of defense against the noise. They're the proof of what your design produces.

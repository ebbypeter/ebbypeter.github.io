---
title: "Everyone Owns Cloud Security. That's Why Nobody Does."
date: 2026-03-24
draft: true
description: "The shared responsibility model tells you where the cloud provider's obligations end. It says nothing about which team inside your organisation owns each control on your side of that line. That gap is where breaches live."
summary: "165 organisations got hit in the Snowflake breach using no novel attack — just stolen credentials, no MFA, and nobody watching. The shared responsibility model didn't fail technically. It failed organisationally. Security wrote the policy. Engineering assumed someone reviewed it. The platform team figured 'managed' meant secured. Procurement filed the SOC 2 and called it done. Nobody lied. Nobody was negligent. They just each assumed someone else had it."
tags:
  - cloud-security
  - security
  - sre
  - azure
  - aws
categories:
  - Security
  - Architecture
slug: everyone-owns-cloud-security-thats-why-nobody-does
---

*The shared responsibility model tells you where the cloud provider's obligations end. It says nothing about which team inside your organisation owns each control on your side of that line.*  
*That gap is where breaches live.*

---
In mid-2024, attackers walked into at least 165 enterprise Snowflake environments and
helped themselves. AT&T. Ticketmaster. Santander. LendingTree. Terabytes of customer
data, out the door. The method was not sophisticated: credentials stolen by infostealer
malware, some of them sitting unused since 2020, used against accounts with no MFA, no
network restrictions, no monitoring worth mentioning. Snowflake's platform was not
touched. The investigation confirmed it. Snowflake's CISO said it plainly: "customers
are responsible for enforcing MFA with their users."

Technically accurate. Completely useless.

Because here's the question nobody asked in those post-mortems: inside each of those
165 organisations, who specifically owned that? Not who used Snowflake. Not who signed
the contract. Who owned the access controls? The credential rotation schedule? The
list of accounts that were still active after people had left? In most cases, that
question had no named answer. And that's the actual story.

The shared responsibility model didn't fail because 165 security teams were negligent.
It failed because "the customer is responsible" is an organisational fiction. Customers
aren't monolithic. They're a collection of teams with different priorities, different
assumptions, and different mental models of who owns what. The model tells you where
the boundary is. It has nothing to say about which human being on the customer's side
of that boundary is accountable for any given control.


## What Teams Actually Hear

Spend any time across engineering, security, and platform teams at a mid-to-large
organisation and you'll find four distinct versions of shared responsibility running
in parallel. Nobody announces them. They're just the working assumptions each group
has quietly formed.

Security teams generally believe their job is policy. They write the runbook that says
MFA is required. They publish the cloud security standard. They run the awareness
training. What they don't own, in their view, is enforcement. That's operations, or
the platform team, or whoever provisioned the thing. So when a Snowflake account goes
three years without MFA because nobody enforced the policy, the security team will
point to the document. They're not wrong. They're also not useful.

Engineering and DevOps teams tend to operate on the assumption that if there were a
security problem, someone would have caught it before production. There's usually meant
to be a review gate somewhere. Sometimes there is one, but it's advisory. Sometimes
it was skipped for this particular service because it was "just" a data platform, not
a customer-facing app. Either way, the engineer who spun up the Snowflake workspace
two years ago assumed the security team reviewed it. The security team assumed the
platform team governed it. Nobody checked.

Platform and cloud teams have their own version. When organisations move from managing
bare metal to managed cloud services, there's a mental shift that happens alongside the
technical one. You used to patch the OS. Now there's no OS. You used to manage the
network appliance. Now there's a managed service. Each abstraction removes a real
responsibility, and that's the point of the abstraction. But it also trains teams to
stop asking what they own, because the answer kept being "less than before." Eventually
the habit becomes: managed service means managed security. It doesn't. Not even close.

Then there's procurement and compliance. They look at the vendor's SOC 2 report, see
that the cloud provider has ISO 27001 and FedRAMP authorisation, and file it as
evidence of security coverage. Google's own HIPAA documentation states that "complying
with HIPAA is a shared responsibility between the customer and Google." Most procurement
teams never read it that far. Why would they? The vendor passed the audit. Doesn't that
mean the environment is secure?

Every one of those positions makes sense from inside the team that holds it. Nobody is being reckless. But put them together and you get an organisation where security wrote a policy, engineering
assumed it was reviewed, the platform team assumed "managed" meant secured, and
procurement filed the vendor's cert as proof of compliance. The gap between those
assumptions is exactly where the breach lives.


## The Autopsy

The Snowflake incident is worth sitting with because it's so structurally clean.

The attackers, a financially motivated group tracked as UNC5537, didn't find a
zero-day. They didn't social-engineer their way past a well-defended perimeter. They
used credential stuffing: usernames and passwords stolen by infostealer malware, many of them years old, tried against Snowflake login pages. The accounts they got into had no MFA. Some hadn't rotated credentials since 2020. Some were demo
or orphaned accounts that should have been decommissioned and weren't, because nobody
was watching them.

Mandiant's investigation found that at least 79.7% of the compromised accounts were
using credentials stolen in prior infostealer campaigns. This was not a novel attack.
It was a known, documented threat pattern that organisations had years to defend against.

None of that happened, across 165 separate organisations, in the same window of time.

That number is the tell. One company failing to rotate credentials or enforce MFA is
a process failure. One hundred and sixty-five companies failing the same way, in the
same service, simultaneously, is a structural problem with how shared responsibility
lands inside organisations. Each of those companies had a security policy somewhere.
Most probably had a SOC 2 or equivalent. Some were among the most recognisable brands
in the world. The failure wasn't knowledge. It was ownership.

Snowflake's post-breach response is worth a brief note. They made MFA mandatory for
all newly created accounts in October 2024, and blocked password-only logins entirely
by November 2025. Read that again. The fix was to take a customer-side control and
make it non-optional at the platform level. Snowflake, having watched their customers
fail to own that control, effectively took it back. That's not a criticism. It was the right call. But it tells you something about where shared responsibility breaks down in practice, and what it takes to actually close the gap.


## The Managed Service Trap

The confusion compounds as you move up the abstraction stack.

With IaaS, the ownership boundary is uncomfortable but legible. You own the OS, the
application, the network config. Teams know it. With PaaS, you lose the OS. Most
engineers adjust and move on. Managed services are where the mental model quietly
breaks. RDS, Azure Database for PostgreSQL, Snowflake, Azure Functions. In each of
those names, "managed" is doing enormous cognitive work. It implies stewardship.
Someone competent is handling the important parts. And in some ways they are: the
platform is patched, the hardware is maintained, availability is someone else's SLA.

What "managed" doesn't touch: the security group controlling network access. The IAM
roles attached to the service. Whether encryption at rest is enabled. Whether audit
logging goes anywhere useful. Credential rotation. Who still has access after three
people left the team last quarter.

All of that is still yours. But "managed" trained the brain to stop asking.

Toyota discovered in 2023 that a cloud storage setting had been marked "public" instead
of "private." Data on 2.15 million customers had been accessible for ten years. Not
because the platform failed. Because nobody in the ownership chain was watching that
field.

## Why the Tools Won't Save You

Someone reading this is probably thinking: "isn't this what CSPM is for?"

Yes, and no. Tools like Wiz, Prisma Cloud, and Microsoft Defender for Cloud are
genuinely good at finding misconfigurations. They scan continuously. They surface
drift. They'll tell you the Snowflake network policy is missing, or the S3 bucket is
public, or the service account has permissions it shouldn't. The finding is not the
problem.

The problem is that a finding needs an owner. If the alert routes into a Jira backlog
that nobody has agreed to manage, the misconfiguration sits there. If the CSPM report
goes to the security team and the security team considers themselves advisory, the
platform team never hears about it. If the platform team hears about it and considers
it an engineering problem, the ticket ages. The tool identified the gap. The
organisation failed to close it for the same reason the gap existed in the first place:
nobody owns this.

DevSecOps has a similar limitation. Shifting security left into CI/CD pipelines catches
misconfigurations at build time, and that's worth doing. But it addresses the leading
edge of new deployments. It doesn't govern the posture of everything deployed before
the pipeline existed. And it doesn't resolve who owns remediation when a scan fires.
You can automate the detection. You cannot automate the accountability.


## The Fix Is an Org Problem, Not a Config Problem

Start with named ownership, not layer ownership. "Engineering owns security for their
services" is not an ownership model. It's a way of deferring the conversation until
after the breach. What actually works is naming a specific owner at the control level,
per service. For your Snowflake environment: who owns MFA enforcement? Who owns the
credential rotation schedule? Who owns the access review when someone leaves the team?
If that conversation surfaces a fight between teams, good. That fight needed to happen.
The gap you're arguing about is the gap that gets you breached.

Then encode the controls rather than documenting them. A policy that says MFA is
required is a liability if nobody enforces it. AWS SCPs, Azure Policy, Snowflake's
account-level MFA enforcement. These are how you turn an aspiration into a constraint.
If you're relying on humans to voluntarily comply with a written standard across
hundreds of services and dozens of teams, you've already made a bet you're going to
lose eventually. Cloud platforms give you the ability to make security controls
infrastructure. Use it.

The last one is harder. After a cloud security incident, the instinct is to find who
misconfigured the thing and fix it so it doesn't happen again. That addresses the
symptom. It doesn't touch the condition that let the misconfiguration sit undetected
for two years. The more useful post-incident question isn't "who made the mistake?"
It's "why did no named owner exist for that control?" That question points at what
needs to change in the org structure, not just in the config.

None of this is technically difficult. All of it requires someone willing to force
the ownership conversation when every team would rather leave the boundaries comfortable
and vague.


## Responsibility Isn't a Diagram

The shared responsibility model is a contract between your organisation and your
cloud provider. It is precise about where the provider's obligations end and yours
begin. What it was never designed to do is tell your platform team and your security
team which of them owns the MFA policy on a managed data warehouse.

Most organisations treat it like an org chart. It isn't one.

Before your next cloud deployment, try a different question. Not "are we covered under
the shared responsibility model?" but "can we name a specific person who owns each
control we're responsible for?" IAM policy governance: who? MFA enforcement: who? Credential rotation: who? Account lifecycle when someone leaves: who?

If the answer to any of those is "the team" or "security" or "whoever provisioned it,"
you've already found the vulnerability. It just hasn't been exploited yet.
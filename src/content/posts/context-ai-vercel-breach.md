---
title: How a Third-Party OAuth App Bypassed Vercel's CI/CD Platform Defenses
published: 2026-04-20
updated: 2026-04-20
description: "Recent Context.ai security incident and the vulnerabilities that allowed a third-party OAuth app to access Vercel's internal systems."
tags: [Security,Vercel,Context.ai,OAuth,Data Breach]
image: "../../assets/images/post-covers/context-ai-vercel-breach.png"
category: 'Security'
draft: false 
---




A [Vercel](http://vercel.com/) employee used an AI meeting assistant called [Context.ai](https://context.ai/) and gave it access to their Google Workspace account. An attacker later used that OAuth token to move through Vercel's internal systems and read customer secrets. The secrets marked as "sensitive" were protected because they were encrypted at rest, but everything else could be read in plain text.

## What happened

On April 19, 2026, Vercel published a security bulletin saying someone had gained unauthorized access to some of its internal systems and affected a small number of customers.
The company traced the attack back to Context.ai, a third-party AI tool used by a Vercel employee.

Here’s what happened:

1. Context.ai’s Google Workspace OAuth app was compromised. The attacker got access to the app with client ID `110671459871-30f1spbu0hptbs60cb4vsmv79i7bbvqj.apps.googleusercontent.com`. This was not a Vercel-only app. It was used by hundreds of organizations.
2. Using that OAuth access, the attacker took over the employee’s Google Workspace account at Vercel. Since Google Workspace is connected to Vercel’s internal systems, that account became a way into the company.
3. From there, the attacker moved into customer environments and accessed environment variables that were not marked as sensitive. Those variables were stored in a readable format.
4. The attacker then used those variables to discover and access more systems.

Vercel described the attacker as highly sophisticated because of how quickly they moved and how well they understood the company’s internal systems.

[Guillermo Rauch](https://x.com/rauchg/status/2045995362499076169) said the attacker moved unusually fast and seemed to know Vercel’s internal architecture very well. He also said the attack may have been significantly sped up by AI.


## What was exposed

Vercel has confirmed that environment variables marked as "sensitive" were not accessed. These variables use Vercel's sensitive environment variable feature, which stores values in a non readable format [once created](https://vercel.com/docs/environment-variables/sensitive-environment-variables). The feature was released before this incident and is available for production and preview environments. Development environment variables cannot be marked sensitive.

What was exposed: environment variables that had not been marked as sensitive. These are stored in an encrypted-but-readable format, protected at rest, but accessible to anyone with the right level of access to the project. In practice, this means API keys, tokens, database credentials, and signing keys that customers had entered without enabling the sensitive flag could have been read by the attacker.

Vercel has not disclosed the total number of affected customers or what specific data was exfiltrated. The investigation is ongoing. See the "What the threat actor is claiming" section below for additional details and context on the claims circulating online.

## What the threat actor is claiming

A threat actor using the [ShinyHunters](https://en.wikipedia.org/wiki/ShinyHunters) name posted on a hacking forum claiming to have breached Vercel and offering access keys, source code, database data, and internal deployment access for $2 million. They shared a sample file containing 580 Vercel employee records, names, email addresses, account status, and activity timestamps. The threat actor claimed on Telegram that they were in direct communication with Vercel about the incident, though Vercel has not independently confirmed this.

The forum post specifically claimed the attacker gained "access to multiple employee accounts with access to several internal deployments, API keys (including some NPM tokens and some GitHub tokens)." The threat actor also shared what appeared to be a screenshot of an internal Vercel Enterprise dashboard.

The identity of the threat actor is disputed. Members of the actual ShinyHunters group have reportedly denied to security reporters that they are involved in this incident. The ShinyHunters name has been used by multiple different actors over the years, and some of the individuals currently associated with it have denied the involvement. This matters because the real ShinyHunters is the group behind the Ticketmaster breach and several other high-profile incidents. If someone is using the name without the backing of the actual group, the data claims could be inflated, genuine, or somewhere in between.

[BleepingComputer](https://www.bleepingcomputer.com/news/security/vercel-confirms-breach-as-hackers-claim-to-be-selling-stolen-data/) has not independently confirmed whether the leaked data is authentic. Vercel has not confirmed whether their data was sold. Treat the threat actor's claims as allegations, the breach itself is confirmed, but the scope and sale are not.

## The bigger picture

This incident stands out because it shows a problem SaaS companies have struggled with for years: **employee productivity tools can become entry points into core infrastructure.**

An engineer might connect Vercel to their Google account, then connect that same account to an AI meeting assistant like Context.ai. That creates a chain of access the security team may never have directly approved or fully understood.

The compromise of Context.ai’s OAuth app may have affected hundreds of organizations, not just Vercel. Vercel is the most visible victim so far, but any company whose employees approved that app could face similar risk.

What makes this especially concerning for developers is the size of the blast radius. Vercel does not just host websites. It also manages deployment pipelines, build systems, and environment variables for thousands of projects. If an attacker gets access at the provider level, every downstream project can be exposed, even if those teams never approved the risky integration themselves.

This also highlights an important design problem. The feature for marking environment variables as sensitive already existed, but it was optional and not enforced. After the incident, Vercel changed that. For other platforms, the real question is not whether they offer a security feature like this. The real question is whether they make it the default.

## Sources

- [Vercel April 2026 Security Bulletin](https://vercel.com/kb/bulletin/vercel-april-2026-security-incident) — Official incident disclosure, root cause analysis, IOC, customer recommendations
- [Guillermo Rauch on X](https://x.com/rauchg/status/2045995362499076169) — CEO's detailed update covering escalation chain, AI-accelerated attacker assessment, supply chain verification, product changes
- [BleepingComputer: Vercel confirms breach as hackers claim to be selling stolen data](https://www.bleepingcomputer.com/news/security/vercel-confirms-breach-as-hackers-claim-to-be-selling-stolen-data/) — Reporting on ShinyHunters claims, $2M ransom demand, 580 employee records
- [The Hacker News: Vercel Breach Tied to Context AI Hack](https://thehackernews.com/2026/04/vercel-breach-tied-to-context-ai-hack.html) — Incident coverage and Context.ai connection
- [Vercel Docs: Sensitive Environment Variables](https://vercel.com/docs/environment-variables/sensitive-environment-variables) — Technical details on the sensitive env var feature and policy enforcement
- [Vercel on X](https://x.com/vercel/status/2045865072074035664) — Initial public disclosure and security bulletin link

---
title: "SLA vs SLO vs SLI"
aliases: [SLA, SLO, SLI, Service Level Agreement, Service Level Objective, Service Level Indicator]
tags: [system-design, glossary]
---

# SLA vs SLO vs SLI

> [!info] Definitions
> - **SLI** (Service Level *Indicator*) — the actual **measured metric**, e.g. "99.95% of requests succeeded last month."
> - **SLO** (Service Level *Objective*) — the **internal target** you're aiming for, e.g. "99.9% success rate." Usually stricter than the SLA, giving buffer before breach.
> - **SLA** (Service Level *Agreement*) — the **external promise** to customers, often with financial penalties if breached, e.g. "99.9% uptime or you get a service credit."

**The relationship:** SLI is what you *measure*, SLO is what you *target* internally, SLA is what you *promise* externally (and is typically looser than the SLO, so normal operational noise doesn't trigger a customer-facing breach). "**Five nines**" (99.999% availability) means roughly 5 minutes of downtime allowed per year — a useful number to have memorized for scale discussions.

**Used in:** [[HLD/01 - Foundations & Estimation/Index|HLD 01 - Foundations & Estimation]]

---
title: "Cloud Fundamentals"
aliases: [IaaS, PaaS, SaaS, Availability Zones, Regions, Managed Services]
tags: [system-design, cs-fundamentals, architecture-patterns, cloud]
status: reference-quality
---

# Cloud Fundamentals

> [!abstract] What you'll be able to do after this chapter
> State the IaaS/PaaS/SaaS hierarchy precisely (a classic, exact-wording interview question), explain the real region-vs-availability-zone distinction, and name vendor lock-in as a genuine, named cost of managed services rather than presenting cloud as strictly superior to self-hosting.

---

## The big picture

```mermaid
graph TD
    A["Cloud Computing"] --> B["Service Models<br/>(how much YOU manage)"]
    A --> C["Geographic Structure"]
    A --> D["Managed Services"]

    B --> B1["IaaS — raw VMs, storage, network<br/>(you manage OS + up)"]
    B --> B2["PaaS — you deploy code<br/>(provider manages OS/runtime)"]
    B --> B3["SaaS — fully managed app<br/>(you just use it)"]

    C --> C1["Region<br/>(a geographic area)"]
    C --> C2["Availability Zone<br/>(isolated data center WITHIN a region)"]

    D --> D1["Reduces operational burden"]
    D --> D2["Real cost: vendor lock-in"]
```

## What is it, and why does it exist?

Cloud computing is the model of renting computing resources — servers, storage, databases, networking — from a third-party provider (AWS, GCP, Azure) over the internet, on demand, instead of buying and operating physical hardware yourself.

**The problem this solves:** before cloud computing, running a service meant buying physical servers (a large upfront capital expense, months of lead time to procure and install), housing them in a data center, and staffing people to maintain the hardware — and over-provisioning for peak load that then sits mostly idle the rest of the time. Cloud computing turns that fixed capital expense into a variable operational one: rent exactly the capacity needed, scale it up or down on demand, pay only for what's actually used.

> [!example] Layman analogy
> Owning a car vs. using a ride-sharing service. Owning means a large upfront cost, and you personally maintain a vehicle that sits idle most of the day. Ride-sharing means paying only when you actually need a ride, someone else maintains the fleet, and you can summon far more capacity instantly (multiple cars at once) for an unusual event, without owning that capacity year-round.

## IaaS / PaaS / SaaS — the exact hierarchy

```mermaid
graph TD
    A["IaaS<br/>you manage: OS, runtime, app, data<br/>provider manages: physical hardware, virtualization<br/>example: AWS EC2"] 
    B["PaaS<br/>you manage: app, data<br/>provider manages: OS, runtime, scaling infra<br/>example: AWS Elastic Beanstalk, Heroku"]
    C["SaaS<br/>you manage: nothing<br/>provider manages: everything<br/>example: Gmail, Salesforce"]
    A --> B --> C
```

> [!tip] A classic, precise-wording interview question — say it exactly
> **IaaS (Infrastructure as a Service):** raw virtual machines, storage, and networking — you manage everything from the OS upward. **PaaS (Platform as a Service):** the provider manages the OS and runtime — you just deploy your application code. **SaaS (Software as a Service):** a fully managed application — you just use it, managing nothing underneath at all. Each step up the hierarchy trades control for reduced operational burden.

## Regions and Availability Zones — the real, precise distinction

> [!warning] A commonly conflated pair, worth stating exactly
> A **Region** is a broad geographic area (e.g., `us-east-1`). An **Availability Zone (AZ)** is an isolated data center **within** a region — physically separate power, cooling, and networking from other AZs in the same region, but low-latency connected to them. Deploying across multiple **AZs** protects against a single data center failing. Deploying across multiple **Regions** protects against an entire geographic area failing (a regional power grid issue, a natural disaster) — a strictly larger, rarer, and more expensive-to-guard-against failure mode.

```mermaid
graph TD
    R["Region: us-east-1"] --> AZ1["Availability Zone A<br/>(isolated data center)"]
    R --> AZ2["Availability Zone B<br/>(isolated data center)"]
    R --> AZ3["Availability Zone C<br/>(isolated data center)"]
```

> [!info] Direct reuse
> This is exactly the geographic reasoning [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|the Multi-Region Rate Limiter chapter]] builds on — multi-AZ deployment as the default resilience baseline, multi-region as the deliberate, more expensive next step for the specific failure modes that justify it.

## Managed services — the real tradeoff

A **managed service** is the provider's own hosted, operated version of a technology — AWS RDS is "managed Postgres," letting you skip operating the database yourself (patching, backups, failover) entirely.

| Benefit | Cost |
|---|---|
| Dramatically reduced operational burden — the provider handles patching, backups, failover | Higher per-unit cost than self-hosting at very large scale |
| Faster to get started — no setup/operational expertise needed upfront | **Vendor lock-in** — migrating off a cloud-specific managed service later is genuinely, non-trivially expensive |

> [!bug] Vendor lock-in is a real, named cost — not a hypothetical one
> A managed service's convenience often comes with provider-specific APIs, configuration, and operational quirks that don't transfer to another cloud provider or to self-hosting. Migrating away later means re-architecting around a different (or absent) managed equivalent — a real, budgeted engineering cost many companies underestimate when adopting a managed service for its initial convenience.

## Where this shows up later

> [!success] Direct connections
> Every "the system auto-scales" claim made casually throughout this book's HLD chapters is enabled by cloud elasticity — the ability to add/remove capacity automatically based on load. [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Multi-Region Rate Limiter]]'s region/AZ reasoning is a direct application of the geographic structure above. [[CS Fundamentals/07 - Architecture and Deployment Patterns/Kubernetes Fundamentals|Kubernetes]] is frequently consumed *as* a managed cloud service itself (AWS EKS, GCP GKE) — tying this chapter directly back to the previous one.

---

## Interview Q&A

> [!question]- Why would a company choose IaaS over PaaS for a given workload?
> When the team needs fine-grained control over the OS/runtime environment — custom kernel parameters, specific runtime versions, unusual networking setups — that PaaS's managed abstraction doesn't expose. PaaS trades that control for significantly less operational burden, the right choice when the team just wants to ship application code without managing infrastructure underneath it.

> [!question]- Why deploy across multiple Availability Zones instead of just adding more servers in one AZ?
> More servers in one AZ doesn't protect against that AZ's power, cooling, or networking failing entirely — all those servers go down together. Spreading across AZs means a single data-center-level failure only takes out a fraction of capacity, not all of it — the same fault-isolation reasoning behind not putting all resources behind one point of failure, applied at the physical-infrastructure level.

> [!question]- What's a real, concrete example of vendor lock-in costing a company?
> Building an application deeply around a cloud provider's proprietary managed database (custom query extensions, provider-specific scaling/replication features) means migrating to another provider later requires re-architecting significant portions of the application, not just moving data — a real, budgeted risk worth weighing explicitly against the convenience gained upfront, not an abstract concern.

## Summary / Cheat Sheet

- **IaaS** (you manage OS+) → **PaaS** (you manage just app+data) → **SaaS** (you manage nothing) — decreasing control, decreasing operational burden.
- **Region** = broad geographic area. **Availability Zone** = isolated data center *within* a region. Multi-AZ protects against a data-center failure; multi-region protects against a whole-area failure.
- **Managed services** trade operational burden for cost and **vendor lock-in** — a real, named tradeoff, not free convenience.

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Design a Multi-Region Rate Limiter]] · [[CS Fundamentals/07 - Architecture and Deployment Patterns/Kubernetes Fundamentals|Kubernetes Fundamentals]]*

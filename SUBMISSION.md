# Submission

| Item | Value |
|---|---|
| **Candidate Name** | William Araque |
| **Candidate ID (Email)** | araquedw@gmail.com |
| **Role Applied** | Technical Lead — Direct Digital Channel (Back-End Developer Assessment) |
| **Date Submitted** | 2026-11-21 |

---

## Summary

This submission establishes a resilient, scalable, and multi-tenant target architecture for Meridian Mutual’s Direct Digital Channel across Mexico, Colombia, and Chile. The solution unifies our fragmented digital footprint into a shared codebase governed by the Strategy Pattern and deployed via dedicated country runtime bulkheads (Azure Container Apps). By decoupling quote calculation from Salesforce Core through daily distributed caching of rating tables and asynchronous policy issuance, the architecture guarantees sub-800ms p95 quote latency (beating constraint C2) and reduces Salesforce API consumption by 85% (protecting constraint C6 and preventing outages like INC-0907). Furthermore, it introduces rigorous engineering standards, consumer-driven contract testing (Pact), progressive canary deployments, a blameless postmortem framework, a structured vendor governance model for Nimbus Digital, and a capacity-backed Q1–Q2 2027 delivery roadmap that hits every commercial milestone for Home Essentials (BR-01) while de-risking conversational GenAI (BR-02).

---

## Time Spent

| Part | Time Spent |
|---|:---:|
| **Reading and analyzing the material** | 25 min |
| **Part A — Architecture, standards and governance** | 2 h 00 min |
| **Part B — Quality and automation** | 1 h 00 min |
| **Part C — Reliability and operations** | 1 h 15 min |
| **Part D — Pull request review and mentoring** | 55 min |
| **Part E — Roadmap and feasibility** | 1 h 00 min |
| **Part F — Responsible AI and Final Review** | 35 min |
| **Total** | **7 h 10 min** |

---

## The Three Decisions I Am Most Confident About

1. **Daily Distributed Caching of Rating Tables in Redis & Asynchronous Policy Issuance (ADR-0002):**  
   Eliminating live queries for the 6 rating tables per quote directly resolves the root cause of incident INC-0907 and isolates user-facing availability from Salesforce's 3% unplanned downtime. Operating within the business-approved 24-hour table freshness window guarantees sub-second quote times (p95 < 800 ms), protects the 40 req/s platform governor limit, and preserves daily API budgets.

2. **Unified Codebase with Country-Isolated Deployments and Strict Bulkheads (ADR-0001):**  
   Adopting a single canonical repository (`channel-bff`) parameterized by Spring Strategy beans and JSON Schema configurations delivers 100% code reuse (fulfilling BR-01 sponsor demands to *"build once and make the next product cheaper"*), while dedicated container scale groups and dedicated API Gateway worker pools eliminate cross-country blast radius (fulfilling C1).

3. **Demanding Full CI/CD Integration & Quality Gates in the Nimbus Contract Renewal (ACT-05 / C8):**  
   Siloed vendor deployments with unrotated credentials and zero pipeline testing created the vulnerabilities exposed in INC-0907 and PR-388. Mandating that Nimbus Digital author code within Meridian’s Azure DevOps repositories, pass automated quality/security gates, and obtain Meridian Tech Lead approval before merging is non-negotiable for system integrity.

---

## The Decision I Am Least Sure About

**The exact boundary between local rating calculation in the MuleSoft Process API vs. delegating to Salesforce Apex rating (ADR-0002).**
- *Why I am unsure:* Executing rating rules entirely inside the MuleSoft Process API (`home-quote-papi`) using cached tables guarantees zero Salesforce API calls and < 20ms execution times. However, for complex insurance products with 80% shared and 20% diverged country rules, maintaining the rating calculation logic in two places (Salesforce Core for back-office and MuleSoft for digital channel) risks pricing divergence if actuarial rules change out-of-band.
- *What would change my mind:* If the Salesforce Platform Team can prove that their Apex rating endpoint consistently achieves p95 < 200 ms and guarantees 99.9% uptime with dedicated capacity, delegating calculation to Salesforce via a single API call per quote would be preferable to maintain single-source-of-truth actuarial logic.

---

## What I Would Do with More Time

1. **Develop an Interactive Architecture Prototype:** Implement a working Docker Compose proof-of-concept demonstrating the Spring Boot 3 Strategy Pattern, Redis rating table ingestion, and a WireMock simulation of the 30 req/s Salesforce rate limiter.
2. **Author Executable Pact Contract Tests:** Write real consumer and provider contract tests between an Angular mock frontend and `channel-bff` to demonstrate automated breaking schema detection in PR pipelines.
3. **Conduct an Empirical RAG Benchmark for BR-02:** Build a small-scale Python evaluation script using the RAGAS framework against sample Mexican and Colombian Home insurance policy PDFs, measuring ground-truth retrieval accuracy and token cost per conversation.
4. **Formalize a Chaos Engineering Game Day Runbook:** Create step-by-step failure injection scripts in Azure Chaos Studio to simulate regional VNet link cuts and Salesforce token latency spikes.

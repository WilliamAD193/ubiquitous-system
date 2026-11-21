# Technical Roadmap — Q1 and Q2 2027

## 1. Prioritization Framework and Scoring

To balance business demands (**BR-01 Home Essentials** and **BR-02 GenAI Assistant**) against platform reliability (**INC-0907 Postmortem Actions**) and accumulated technical debt (**TD-01 to TD-12**), we employ a modified **RICE / WSJF (Weighted Shortest Job First)** scoring model:

$$\text{Priority Score} = \frac{\text{Business Value (1–5)} \times \text{Risk Reduction / Reliability Impact (1–5)}}{\text{Effort in Person-Weeks (pw)}} \times 10$$

- **Business Value:** Impact on revenue, multi-country expansion, and sponsor commitments.
- **Risk / Reliability:** Blast-radius containment, compliance with C1/C5/C6, and incident prevention.
- **Effort:** Team estimate in person-weeks (pw) from `TECH_DEBT_INVENTORY.md` and `BUSINESS_REQUESTS.md`.

---

### 1.1 Item Scoring Table

| Item ID | Category | Description | Value (1–5) | Risk / Rel (1–5) | Effort (pw) | Score | Priority Rank | Target Sprint |
|---|---|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **ACT-01** | Postmortem | Hard bulkheads & tenant thread pools in `quote-xapi` | 4 | 5 | 2.0 | **100.0** | **1** | Q1 S1 |
| **ACT-02** | Postmortem | Gateway timeout reduction (3.5s) & Circuit Breakers | 4 | 5 | 1.5 | **133.3** | **2** | Q1 S1 |
| **ACT-04** | Postmortem | Daily distributed rating table caching (Redis) | 5 | 5 | 2.5 | **100.0** | **3** | Q1 S1–S2 |
| **TD-11** | Tech Debt | Chile secrets migration to Azure Key Vault | 2 | 5 | 0.5 | **200.0** | **4** | Q1 S1 |
| **ACT-05 / TD-06** | Governance | Move Nimbus CI deployments to Meridian Azure Pipelines | 4 | 5 | 4.0 | **50.0** | **5** | Q1 S2–S3 |
| **BR-01** | Business | **Home Essentials Regional Launch** (MX, CO, CL) | 5 | 4 | 50.0 | **4.0** | **Critical Strategic** | Q1 S2 – Q2 S6 |
| **TD-01** | Tech Debt | Consolidate 3 Process APIs into unified `home-quote-papi` | 5 | 4 | 9.0 | **22.2** | **6** (Part of BR-01) | Q1 S2–S5 |
| **TD-03** | Tech Debt | Migrate Mexican journeys to unified `channel-bff` | 5 | 4 | 7.0 | **28.6** | **7** (Part of BR-01) | Q1 S2–S4 |
| **TD-02** | Tech Debt | Route Mexican issuance through `sf-policy-sapi-mx` | 4 | 4 | 5.0 | **32.0** | **8** | Q1 S4–S5 |
| **TD-05** | Tech Debt | Implement Consumer-Driven Contract Tests (Pact) | 4 | 4 | 3.5 | **45.7** | **9** | Q1 S4–S5 |
| **TD-10** | Tech Debt | Feature flag framework (Azure App Config / LaunchDarkly) | 4 | 4 | 2.5 | **64.0** | **10** | Q1 S2 |
| **TD-08** | Tech Debt | 30-day quote draft TTL & automated purge (Privacy C5) | 3 | 4 | 1.5 | **80.0** | **11** | Q1 S3 |
| **ACT-06 / TD-07** | Operations | Salesforce quota telemetry & multi-burn-rate alerting | 4 | 4 | 2.5 | **64.0** | **12** | Q1 S3 |
| **TD-04** | Tech Debt | Move Mexico frontend to unified `web-quote-ui` (Angular 17) | 4 | 3 | 10.0 | **12.0** | **13** | Q2 S1–S4 |
| **TD-12** | Tech Debt | Upgrade `mx-quote-bff` Spring Boot 2.7 (Obsolete via TD-03) | 1 | 2 | 3.5 | **5.7** | **Dropped (Superseded)** | Deprecated |
| **TD-09** | Quality | Bring CL & MX unit test coverage above 80% | 3 | 3 | Ongoing | **Ongoing** | **Continuous** | All Sprints |
| **BR-02** | Business | **GenAI Quote Assistant (Feasibility Spike)** | 3 | 2 | 4.0 | **15.0** | **Spike in Q2** | Q2 S3–S4 |

---

## 2. Team Capacity and Allocation Model (Q1 and Q2 2027)

From [`context/TEAM.md`](file:///context/TEAM.md):
- **Quarter Duration:** 12 working weeks = 6 two-week sprints per quarter.
- **Run & Support Overhead:** 20% dedicated to maintenance, bug fixes, and minor regulatory requests.
- **Net Capacity for Planned Initiatives:**
  - **Internal Team:** 6 developers (2 FE, 3 BE, 1 MuleSoft) × 12 weeks × 0.8 = **58 person-weeks / quarter**.
  - **Nimbus Digital (Vendor):** 4 developers × 12 weeks = **48 person-weeks / quarter** (focused on Mexican scope under the renewed contract).
  - **Borrowed Integration Capacity:** **1 MuleSoft developer for 6 weeks in Q1**, borrowed from the Salesforce Platform Team (formal request submitted prior to 15 December 2026).
- **Total Net Available Capacity:**
  - **Q1 2027:** 58 (Internal) + 48 (Nimbus) + 6 (Borrowed MuleSoft) = **112 person-weeks**.
  - **Q2 2027:** 58 (Internal) + 48 (Nimbus) = **106 person-weeks**.

---

## 3. Master Delivery Roadmap: Q1 and Q2 2027

```mermaid
gantt
    title Direct Digital Channel Roadmap (Q1 - Q2 2027)
    dateFormat  YYYY-MM-DD
    axisFormat  %b %d

    section Q1 Milestones
    Salesforce Quota & Bulkhead Stabilization :milestone, m0, 2027-01-26, 0d
    Nimbus CI Integration & Contract Renewal :milestone, m1, 2027-01-31, 0d
    BR-01 Home Essentials Mexico Launch :milestone, m2, 2027-04-30, 0d

    section Q1: Sprints 1 to 3 (Stabilization & Foundations)
    ACT-01 & ACT-02: quote-xapi Bulkheads & Timeouts :done, q1_s1, 2027-01-08, 2027-01-22
    ACT-04: Redis Rating Table Caching           :done, q1_s2, 2027-01-15, 2027-02-05
    TD-11: Key Vault Secrets & TD-10: Feature Flags :done, q1_s3, 2027-01-08, 2027-01-22
    ACT-05 / TD-06: Nimbus CI into Azure Pipelines  :active, q1_s4, 2027-01-15, 2027-02-05
    ACT-06 / TD-07: Real-Time SLO & Quota Alerts   :active, q1_s5, 2027-01-22, 2027-02-12
    TD-08: Customer Draft Data Purge (30-day TTL) :q1_s6, 2027-01-22, 2027-02-05

    section Q1-Q2: Home Essentials Core & Mexico Pilot (BR-01)
    TD-01: Build Unified home-quote-papi (with Borrowed Dev) :crit, q1_papi, 2027-01-22, 2027-03-05
    TD-03: Implement Home Quoting in channel-bff (MX)        :crit, q1_bff, 2027-02-05, 2027-03-19
    TD-02: Route MX Issuance via sf-policy-sapi-mx          :crit, q1_sapi, 2027-02-19, 2027-03-19
    TD-05: Pact Contract Testing (BFF <-> PAPI <-> SAPI)    :q1_pact, 2027-03-05, 2027-03-26
    BR-01 MX Pilot UAT & Canary Verification                :q1_uat, 2027-03-26, 2027-04-23
    BR-01 MX Production Go-Live                             :crit, milestone, 2027-04-30, 0d

    section Q2 Milestones
    BR-01 Colombia Launch                                   :milestone, m3, 2027-05-31, 0d
    BR-01 Chile Launch                                      :milestone, m4, 2027-06-30, 0d

    section Q2: Sprints 4 to 6 (Regional Rollout & Modernization)
    BR-01 Colombia Config & Testing (CO Launch May 31)     :crit, q2_co, 2027-05-01, 2027-05-28
    BR-01 Chile Config & Testing (CL Launch June 30)       :crit, q2_cl, 2027-06-01, 2027-06-25
    TD-04: Migrate Mexico to unified web-quote-ui          :q2_ui, 2027-04-15, 2027-06-15
    Retire Chilean Diverged UI Copies                      :q2_cl_ui, 2027-05-15, 2027-06-15
    BR-02: 4-Week GenAI RAG Spike (Go / No-Go)             :q2_ai, 2027-05-01, 2027-05-28
```

---

## 4. What We Will NOT Do in Q1/Q2 2027 (Out of Scope) and Why

| Out-of-Scope Initiative | Reason for Deferral | Alternative / Mitigation |
|---|---|---|
| **TD-12: Upgrade `mx-quote-bff` Spring Boot 2.7** | **Obsolete Work:** We are actively retiring `mx-quote-bff` and shifting all Mexican journeys to `channel-bff` under TD-03. Upgrading a codebase slated for deletion is wasted effort. | Accelerate migration of Mexican Auto journeys to `channel-bff` in Q3 2027. |
| **BR-02: Customer-Facing GenAI Assistant Production Launch in Q1** | **High Regulatory & Reliability Risk:** Compliance forbids unregulated advisory; Security has approved LLMs for internal pilots only; the team must focus 100% on platform stabilization and BR-01. | Time-boxed 4-week architectural spike conducted in Q2 (May 2027) with clear Go/No-Go criteria. |
| **Full Migration of Mexican Auto Journeys in Q1** | **Capacity & Scope Containment:** Moving existing Auto journeys concurrently with Home Essentials introduces extreme delivery risk for the April 30 launch. | Auto journeys remain on legacy BFF until Home Essentials stabilizes; migrated incrementally in Q3 2027. |
| **Real-time Event-Driven Data Sync for Salesforce Core** | **External Dependency:** Salesforce orgs do not yet support streaming Pub/Sub API. | Daily scheduled Redis caching at 05:05 UTC fully satisfies business requirements. |

---

## 5. Main Delivery Risks and Mitigation Strategies

| Risk Description | Impact | Probability | Proactive Mitigation Strategy |
|---|:---:|:---:|---|
| **MuleSoft Integration Bottleneck:** Only 1 integration developer in cell; TD-01 and Home Essentials PAPI are on the critical path. | Critical | High | Secure the **6-week MuleSoft developer loan** from the Salesforce Platform Team by submitting the formal request prior to 15 December 2026. |
| **Nimbus Contract Friction (31 Jan 2027):** Vendor resists unified Azure CI/CD pipelines and quality gates. | High | Medium | Incorporate mandatory CI/CD migration and PR standards directly into contract renewal clauses with financial incentives/penalties. |
| **Salesforce Quota Overrun During Peak Months:** Marketing campaigns in MX driving unexpected spikes. | High | Medium | Enforce the 30 req/s rate limiter and Redis caching. Configure Tier 2 alerts at 75% quota consumption. |
| **Country Configuration Drift:** Development teams introducing unauthorized `if (country == 'MX')` hardcoded forks. | Medium | High | Enforce automated SonarQube rules and mandatory Tech Lead code review sign-off before merge. |

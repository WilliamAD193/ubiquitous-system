# ADR-0001: Regional Reuse and Deployment Model for Direct Digital Channel

| Item | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-21 |
| Deciders | Technical Lead (Direct Digital Channel), Director of Digital Channels, Architecture Office |
| Related | C1 (Availability), C3 (Reuse), C5 (Security & Privacy), C8 (Vendor Governance), C9 (Cost), TD-01, TD-03, TD-04 |

## Context

The Direct Digital Channel at Meridian Mutual currently operates across three divergent codebases and runtime environments:
- **Colombia:** Original code running on modern Azure Container Apps (`channel-bff`) and MuleSoft (`quote-papi-co`).
- **Chile:** Copied from Colombia in 2024, maintaining 23 diverged frontend copies and separate database instances.
- **Mexico:** Forked in 2023 by Nimbus Digital (`web-quote-ui-mx`, `mx-quote-bff`, `quote-papi-mx`), lagging on unsupported frameworks (Angular 15, Spring Boot 2.7) and directly coupling with Salesforce.

Business Request **BR-01 (Home Essentials)** requires launching across Mexico (April 2027), Colombia (May 2027), and Chile (June 2027). The business sponsor explicitly demanded: *"We cannot afford to build this three times. Build it once, and make the next product cheaper."*

Furthermore, incident **INC-0907** demonstrated that sharing runtime resources without hard bulkheads allows traffic surges in one country to cascade into outages across all countries, violating **C1**. Legal and data protection regulations (**C5**) also prohibit cross-border data leakage and unmonitored cross-country data access.

## Decision drivers

1. **Failure Containment (C1):** A failure or surge in one country must never impact the others.
2. **Codebase Reusability (C3):** Core journey workflows, rating orchestration, and UI components must be authored once.
3. **Data Protection & Legal Compliance (C5):** Complete physical or logical separation of citizen PII per sovereign legal jurisdiction.
4. **Vendor Governance (C8):** Ability to onboard Nimbus Digital into a unified delivery model before the contract renewal on 31 January 2027.
5. **Cost & Operational Simplicity (C9):** Sustainable cloud infrastructure and maintenance footprint.

---

## Options considered

### Option A — Fully Shared Regional Multi-Tenant Runtime
*A single multi-tenant deployment of frontends, BFFs, and Process APIs hosted in a single regional Azure / MuleSoft cluster, serving all three countries dynamically based on JWT claims.*

| Dimension | Assessment |
|---|---|
| **Complexity** | High runtime routing complexity; high risk of tenant bleed or cross-tenant query bugs. |
| **Cost** | Lowest infrastructure cost (shared instances, maximum density). |
| **Availability and isolation** | **Poor.** Direct violation of C1. As proven in INC-0907, thread pool exhaustion by one country starves all countries. |
| **Security and privacy** | **High Risk.** Shared databases require complex row-level security (RLS); high audit burden under MX, CO, and CL privacy laws. |
| **Fit with team skills and vendor** | Poor. Nimbus and internal teams would constantly collide in shared branches and staging environments. |

**Pros:**
- Minimal infrastructure footprint.
- Single deployment pipeline.

**Cons:**
- Zero runtime blast-radius isolation; violates C1.
- Severe regulatory risk under data sovereignty laws.
- High risk of noisy neighbor issues during country marketing campaigns.

---

### Option B — Unified Codebase with Country-Isolated Deployments (Selected)
*Single canonical source repository per layer (`web-quote-ui`, `channel-bff`, `home-quote-papi`) utilizing the Strategy Pattern and configuration injection for country differences. Deployed as independent, isolated container instances and databases per country within Azure Container Apps and dedicated MuleSoft worker allocations.*

| Dimension | Assessment |
|---|---|
| **Complexity** | Low-to-moderate. Clean separation between core engine code and country configuration beans. |
| **Cost** | Balanced. Container Apps scale to zero during off-peak hours, optimizing cost per country. |
| **Availability and isolation** | **Excellent.** Hard bulkheads. If Mexico's BFF or database experiences an outage, Colombia and Chile remain 100% operational. |
| **Security and privacy** | **Compliant.** Dedicated database instances/schemas per country (`db-channel-mx`, `co`, `cl`) guarantee zero cross-border PII exposure. |
| **Fit with team skills and vendor** | **High.** Nimbus works on the same codebase under shared pull request standards, deploying to Mexico-isolated staging and production targets. |

**Pros:**
- Satisfies **C3** (100% code reuse, 0 forks) while strictly enforcing **C1** (fault isolation).
- Complies seamlessly with **C5** data sovereignty requirements.
- Enables progressive, staggered rollouts (deploying to MX first on April 30, then CO on May 31, then CL on June 30).
- Enables precise per-country cost accounting (FinOps).

**Cons:**
- Pipeline must orchestrate deployments across three isolated target environments.
- Slightly higher resource overhead than a single shared container instance.

---

### Option C — Maintain Separate Repositories per Country (Status Quo)
*Continue maintaining separate forks and repositories for Mexico (Nimbus) and Chile.*

| Dimension | Assessment |
|---|---|
| **Complexity** | Extremely high long-term maintenance; massive technical debt (TD-01, TD-03, TD-04). |
| **Cost** | Highest engineering cost (75–90 person-weeks vs 45–55 person-weeks for BR-01). |
| **Availability and isolation** | Good infrastructure isolation, but terrible defect propagation and inconsistent patching. |
| **Security and privacy** | Inconsistent compliance; security patches not applied (e.g. Angular 15, Spring Boot 2.7). |
| **Fit with team skills and vendor** | Perpetuates vendor silo and prevents technical governance of Nimbus. |

**Pros:**
- No immediate refactoring needed for existing legacy products.

**Cons:**
- Rejected by business sponsor (*"We cannot afford to build this three times"*).
- Triples maintenance costs and bug fix turnaround time (e.g., 19 days for pricing change).

---

## Decision

We adopt **Option B: Unified Codebase with Country-Isolated Deployments**.

1. **Single Source of Truth:** `channel-bff` becomes the regional BFF for Mexico, Colombia, and Chile. All Mexican Home Essentials capabilities are built directly in `channel-bff` (deprecating `mx-quote-bff`).
2. **Strategy-Based Variation:** Country differences (tax rates, mandatory coverage rules, surcharge algorithms) are implemented using Spring Strategy Beans and JSON Schema UI configurations, selected dynamically by the validated JWT tenant context.
3. **Dedicated Runtimes & Bulkheads:**
   - Azure Container Apps: Independent container revisions and scale groups (`channel-bff-mx`, `channel-bff-co`, `channel-bff-cl`).
   - MuleSoft API Gateway (`quote-xapi`): Hard thread-pool bulkheads allocating dedicated in-flight slots per country.
   - Persistence: Dedicated Azure Database for PostgreSQL flexible server schemas with strict country-level encryption keys.

---

## Trade-off analysis

Option B represents the optimal balance between engineering velocity and production reliability:
- While Option A offers marginally lower Azure hosting fees, it completely fails the non-negotiable availability requirement (**C1**) and creates severe regulatory vulnerability under Latin American data protection mandates (**C5**).
- Option C satisfies isolation but directly violates business mandate **BR-01**, requires 40% more effort (75–90 pw vs 45–55 pw), and paralyzes team velocity.
- The minor operational overhead of managing three country container deployments is fully mitigated by modern CI/CD automation in Azure Pipelines.

---

## Consequences

- **Becomes easier:**
  - Feature parity across countries: New products and features (like Home Essentials) are developed once and rolled out across regions with minimal configuration.
  - Zero cross-country blast radius: A Black Friday or month-end peak in Mexico cannot impact Colombian or Chilean customers.
  - Vendor integration: Nimbus Digital submits PRs to the main repository, adhering to unified quality gates and code review standards.
  - Compliance: Independent audit trails and isolated databases satisfy local financial superintendencies.
- **Becomes harder:**
  - Local configuration discipline: Developers must never hardcode country checks (`if (ctry.equals("MX"))`), requiring strict PR gate enforcement.
  - CI/CD complexity: Deployment pipelines must support progressive canary rollouts across three regional targets.
- **To revisit, and when:**
  - Revisit in Q4 2027 during the annual architecture review, assessing whether database instances can be migrated to Azure PostgreSQL Hyperscale or serverless as volume grows.

---

## Action items

1. [ ] Provision dedicated Azure Container App environments for `channel-bff-mx`, `co`, and `cl`.
2. [ ] Establish the `@meridian/ui-kit` shared library for Angular 17.
3. [ ] Configure Spring profiles and Strategy interfaces for tax and document validation in `channel-bff`.
4. [ ] Incorporate unified repository contribution and branch protection policies into Nimbus Digital's contract renewal by 31 January 2027 (C8).

# ADR-0002: Channel-to-Core Salesforce Integration and Rating Table Caching Strategy

| Item | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-21 |
| Deciders | Technical Lead (Direct Digital Channel), Salesforce Platform Team Lead, Integration Architect |
| Related | C2 (Performance <2.5s p95), C4 (API-Led Integration), C6 (Salesforce Limits), INC-0907, PR-388, TD-01, TD-02 |

## Context

Meridian Mutual utilizes Salesforce as both CRM and Policy Core, maintaining one independent org per country (**MX, CO, CL**). Integrating the Direct Digital Channel with Salesforce is governed by severe non-negotiable operational constraints (`SALESFORCE_CONSTRAINTS.md`):
1. **Low Core Availability:** Each Salesforce org achieves only ≈ **97% monthly availability**, with unplanned outages lasting from 5 to 90 minutes.
2. **Strict Concurrency Limits:** The Salesforce platform team enforces a ceiling of **40 requests/second** per org to prevent system-wide degradation.
3. **Daily API Allocations (C6):** Rolling 24-hour channel budgets are capped (MX: 250,000; CO: 300,000; CL: 150,000 calls). When an org reaches 85% of total capacity, Payments Integration pauses its consumers to protect billing, as occurred during **INC-0907**. Exceeding 100% triggers fatal `403 REQUEST_LIMIT_EXCEEDED` errors across all company integrations.
4. **Latency Requirements (C2):** Quote journeys must respond in **< 2.5 s at p95**, including during 3× peak volume periods. Normal Apex rating queries take 350 ms – 1.1 s, while policy issuance takes 1.8 s – 4.0 s.
5. **Rating Table Characteristics:** Each product uses 6 rating tables published daily at 05:00 UTC. Business explicitly permits using tables up to 24 hours old, provided the quote records and surfaces the table version used.

During incident **INC-0907**, Nimbus Digital bypassed caching in `quote-papi-mx 3.8.0` to "always use fresh prices", issuing 6 rating table queries plus 1 rating call per quote. Combined with aggressive retries, this generated 90 req/s and consumed 856,000 API calls, exhausting the MX org budget and collapsing cross-country quoting. In **PR-388**, a developer attempted to bypass the integration platform altogether by issuing direct REST calls from the BFF to Salesforce.

## Decision drivers

1. **Quota & Rate Protection (C6):** Protect daily API budgets and guarantee traffic remains strictly below 40 req/s under all conditions.
2. **Latency Guarantee (C2):** Return quotes to customers in < 2.5 s at p95, decoupling end-user latency from core Salesforce spikes.
3. **Architectural Governance (C4):** Maintain strict three-tier API-led connectivity; eliminate direct unmonitored Salesforce connections from channel applications.
4. **Fault Tolerance:** Isolate end users from Salesforce's 3% downtime windows.

---

## Options considered

### Option A — Strict Synchronous API-Led Invocation without Table Caching
*Every quote request invokes Experience API -> Process API -> System API -> Salesforce Apex Core, querying live rating tables and evaluating premiums in Salesforce.*

| Dimension | Assessment |
|---|---|
| **Complexity** | Low integration logic, but impossible operational maintenance. |
| **Cost** | Extremely high Salesforce API licensing costs; frequent overages. |
| **Availability and isolation** | **Fails (97% max).** The channel inherits Salesforce's 3% downtime directly. |
| **Performance (C2)** | **Fails.** Querying 6 tables + rating takes > 2.8 s at p95 under peak load. |
| **Salesforce Quota (C6)** | **Fatal.** 7 calls per quote exhausts daily allocations within hours of month-end peak. |

**Pros:**
- Real-time rating table updates without caching logic.

**Cons:**
- Direct root cause of INC-0907. Violates C2, C4, and C6.

---

### Option B — Three-Tier API-Led Connectivity with Scheduled Distributed Caching & Async Issuance (Selected)
*Experience API delegates to Process API (`home-quote-papi`). At 05:05 UTC daily, a scheduled worker synchronizes the 6 rating tables per country into a distributed Redis cache. Quoting executes local rating using cached tables (or invokes a single System API rating endpoint passing the table version). Policy issuance executes asynchronously via an Azure Service Bus queue, guaranteeing reliable delivery during Salesforce downtime.*

| Dimension | Assessment |
|---|---|
| **Complexity** | Moderate. Requires Redis cache management and an asynchronous message queue worker for policy binding. |
| **Cost** | Low. Minimal Redis footprint; zero Salesforce API overage penalties. |
| **Availability and isolation** | **Excellent (99.9%).** Quotes can be generated even during Salesforce outages using cached tables. |
| **Performance (C2)** | **Outstanding.** Cached table reads take < 3 ms. Total quote turnaround drops to **< 800 ms at p95**. |
| **Salesforce Quota (C6)** | **Compliant.** Zero Salesforce calls for rating tables. Reduces channel SF API consumption by **85%**. |

**Pros:**
- Fully complies with C2, C4, C5, and C6.
- Decouples user-facing quote latency from Salesforce backend load.
- Asynchronous policy issuance protects against Salesforce 5–90 minute outages: customers receive an instant confirmation, and policies are issued via transactional retry workers.
- Rate limiter at MuleSoft System API caps traffic at 30 req/s (comfortably below the 40 req/s limit).

**Cons:**
- Requires distributed cache management and cache invalidation mechanics for rare out-of-band price updates.
- Asynchronous issuance requires customer communication indicating that official policy documents will arrive via email within 3 minutes (aligns with BR-01 requirements).

---

### Option C — Direct Core Access from Channel BFF with In-Memory Cache (Nimbus Approach)
*Grant the BFF direct access to Salesforce REST APIs (as in PR-388) to bypass MuleSoft development backlogs, relying on application-level in-memory caching.*

| Dimension | Assessment |
|---|---|
| **Complexity** | High architectural degradation; introduces security holes and technical debt (TD-02). |
| **Cost** | Unpredictable; high risk of unmonitored API exhaustion. |
| **Availability and isolation** | Poor. Multiple container replicas cause cache fragmentation and cold-start bursts. |
| **Security and privacy** | **Violates C5.** Requires storing Salesforce client secrets inside BFF configuration files. |
| **Fit with team standards** | Direct violation of enterprise standard C4. |

**Pros:**
- Quick initial implementation for vendor developers.

**Cons:**
- Violates C4, C5, and enterprise security governance.
- Impossible for Salesforce Platform team to track or shape API consumption centrally.

---

## Decision

We adopt **Option B: Three-Tier API-Led Connectivity with Scheduled Distributed Caching & Async Issuance**.

1. **Enforce API-Led Flow:** All traffic routes strictly through `quote-xapi` -> `home-quote-papi` -> `sf-policy-sapi-{ctry}`. No direct Salesforce access is permitted from any frontend or BFF. TD-02 is officially slated for immediate remediation.
2. **Daily Distributed Table Caching:**
   - A cron worker in `home-quote-papi` fetches the 6 rating tables daily at **05:05 UTC** (5 minutes after publication in Salesforce) via the System API.
   - Tables are stored in Azure Cache for Redis with a **36-hour TTL** and keyed as `tables:{country}:{product}:{tableVersion}`.
   - During quote calculation, tables are retrieved in < 3 ms. Quoting requires **0 Salesforce API calls** if rated in the Process API engine, or **exactly 1 call** if delegated to Apex calculation.
   - The response metadata explicitly populates `ratingTableVersion` as mandated by business.
3. **Core Rate Limiter & Concurrency Throttling:**
   - MuleSoft API Manager enforces a Spike Control policy of **30 req/s** on `sf-policy-sapi-{ctry}` to guarantee the 40 req/s org-level threshold is never breached.
4. **Asynchronous Policy Issuance:**
   - When payment succeeds, `channel-bff` publishes a `PolicyIssuanceRequested` event to an Azure Service Bus topic.
   - A dedicated issuance worker in the Process API drains the queue, invoking `sf-policy-sapi` with exponential backoff and dead-letter queueing. If Salesforce is experiencing one of its 5–90 minute outages, user payments succeed and issuance resumes automatically once Salesforce recovers.

---

## Trade-off analysis

- **Consistency vs Availability/Performance:** By choosing Option B, we accept eventual consistency for rating tables (up to 24 hours old). This trade-off was explicitly approved by the business in `SALESFORCE_CONSTRAINTS.md`. In exchange, we gain sub-second quote times, 99.9% quoting availability, and immunity from Salesforce downtime.
- **Asynchronous Issuance UX:** The customer does not wait 4 seconds on screen for Salesforce policy object generation. Instead, the UI displays immediate success upon payment capture, and the policy document is delivered asynchronously via WhatsApp/email in < 3 minutes (satisfying BR-01).

---

## Consequences

- **Becomes easier:**
  - Zero risk of exhausting daily Salesforce API allocations (C6).
  - P95 quote latency drops from 2.8 s to < 800 ms, easily meeting C2.
  - Complete visibility of all core traffic in MuleSoft API Manager.
  - Immunity to core Salesforce micro-outages.
- **Becomes harder:**
  - Operational overhead of managing Redis and Azure Service Bus queues.
  - Handling rare emergency rating table republishing requires a manual cache-purge webhook endpoint (`POST /admin/cache/refresh`).
- **To revisit, and when:**
  - Revisit in Q3 2027 if Salesforce introduces Event-Driven Architecture (Pub/Sub API) for instant table change event notifications.

---

## Action items

1. [ ] Implement the scheduled rating table ingestion workflow in `home-quote-papi`.
2. [ ] Provision Azure Cache for Redis and configure resilience failover.
3. [ ] Apply the 30 req/s rate-limiting policy in MuleSoft API Manager on all `sf-policy-sapi` proxies.
4. [ ] Deprecate direct REST clients in Mexican components and transition issuance to `sf-policy-sapi-mx`.

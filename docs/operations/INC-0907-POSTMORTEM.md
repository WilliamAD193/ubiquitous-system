# Postmortem — Incident INC-0907: Quote Journey Degradation in MX, CO, and CL

| Item | Value |
|---|---|
| **Incident Date** | Wednesday, 30 September 2026 (Month-End Commercial Peak) |
| **Duration** | 15:25 UTC – 17:20 UTC (Mitigation: 1 h 55 min) · Closed: 18:20 UTC (2 h 55 min) |
| **Actual Severity** | **SEV1 — Critical Enterprise Outage** (Misclassified as SEV3 during triage) |
| **Incident Commander** | Channel Manager |
| **Lead Investigator** | Technical Lead (Direct Digital Channel) |
| **Impact Summary** | • ≈ 11,600 quote requests failed across three countries (MX: 9,400; CO: 1,500; CL: 700).<br/>• 2,310 customer calls to contact centers (MX: 1,140; CO: 860; CL: 310).<br/>• Payments Integration squad paused Mexican payment processing for 1 h 25 min (16:55–18:20 UTC) due to Salesforce org quota exhaustion. |

---

## 1. Incident Triage Review & Communication Analysis

### 1.1 Triage as It Occurred vs. How It Should Have Been Done

| Triage Dimension | What Actually Happened | What Should Have Happened | Gap Analysis & Principle |
|---|---|---|---|
| **Severity Declaration** | Declared as **SEV3** at 15:52 UTC (`chat/incident-channel.md`, line 9) by on-call engineer, treating it as a "minor feature issue for some users". | Should have been declared **SEV1** immediately at 15:46 UTC when cross-country errors and total quote failure across 3 countries were confirmed. | Meridian's severity scale defines SEV1 as *"service down for all users"* and SEV2 as *"major feature degraded for many users"*. Quoting is the primary revenue-generating funnel of the digital channel. Failing across MX, CO, and CL simultaneously constitutes an undisputed SEV1. |
| **Impact Assessment** | Narrowly focused on Colombia initially because the Colombian contact center reported it first (line 9). Ignored Mexico and Chile telemetry. | Global impact assessment: Correlate regional gateway error rates across all 3 countries within 2 minutes of alert. | Country-siloed mindset delayed recognizing a systemic cross-tenant failure. |
| **Roles Involved** | Initially only Channel On-Call, DBA, and Nimbus. Salesforce and Payments teams arrived late (16:55 UTC). | Immediate cross-cell war room activation: Incident Commander, Channel Tech Lead, Salesforce Platform Lead, Payments Tech Lead, Communications Lead. | A multi-tenant core degradation requires immediate coordination across all platform consumers. |
| **Initial Communication** | No external customer or advisor communication broadcast. Contact centers were overwhelmed with 2,310 calls. | Statuspage and internal advisory banner broadcast within 10 minutes (15:50 UTC): *"We are experiencing technical difficulties with quoting across MX, CO, CL; our engineering team is actively investigating."* | Lack of incident communication amplified customer frustration and flooded contact center queues. |

---

## 2. Reconstructed Incident Timeline (30 September 2026)

All times UTC. Evidence citations reference logs, metrics, and incident chat.

```mermaid
timeline
    title INC-0907 Incident Progression
    14:00 - 15:00 : Traffic rises for month-end peak : Nimbus 3.8.0 active
    15:25 : quote-papi-mx begins timing out (30s) at quote-xapi : Logs quote-xapi.log L6
    15:34 : quote-xapi worker pool exhausts (64/64) : Shared queue fills up L7
    15:38 : CO and CL quotes rejected with 503 QUEUE_FULL : Cross-tenant cascade L10-11
    15:41 : Colombian contact center escalates failure : chat L7
    15:52 : On-call declares SEV3 (Misclassification) : chat L9
    16:02 - 16:11 : Red herring: Investigation of pg-channel-co and IDP key rotation : chat L10-13
    16:18 : Scale-out of quote-xapi from 2 to 4 replicas : chat L15
    16:38 : Scaled replicas saturate again (MX occupies 96% workers) : Logs L25-26
    16:55 : MX Org hits 85% quota; Payments pauses consumers : chat L18
    16:58 : Salesforce team identifies Channel user consuming 856k calls & 90 req/s : chat L19
    17:00 : IC orders rollback of quote-papi-mx to 3.7.2 : chat L20
    17:05 : Rollback deployed by Nimbus : chat L21
    17:20 : Error rates and latency normalize across all countries (Mitigated) : chat L22
    18:20 : Temporary quota increase granted; Payments resumes; Incident closed : chat L25-27
```

### Detailed Event Log:
- **14:00–15:00 UTC:** Normal month-end traffic ramp in Mexico (3× normal volume).
- **15:25 UTC:** Mexican quotes begin to stall in `quote-papi-mx`. At 15:27:09 UTC, `quote-xapi` logs its first 30-second upstream timeout: `corr=3f6a90c2 country=MX status=504 error=HTTP:TIMEOUT totalMs=30001 upstreamMs=30000` (`logs/quote-xapi.log`, line 6).
- **15:34:51 UTC:** Replica `xapi-0` exhausts its in-flight concurrency limit: `in-flight limit reached (64/64); queuing corr=a1e4c093 country=CO queue=12/100` (`logs/quote-xapi.log`, line 7).
- **15:36:18 UTC:** A Colombian quote request is delayed by **11.2 seconds in the queue**, even though upstream processing in Colombia took only 598 ms: `corr=a1e4c093 country=CO totalMs=11873 queuedMs=11231 upstreamMs=598` (`logs/quote-xapi.log`, line 9).
- **15:38:02 UTC:** Queue capacity (100/100) saturates. Incoming requests for Chile and Colombia are immediately rejected with HTTP 503: `corr=6c2d18fe country=CL status=503 error=QUEUE_FULL totalMs=3` (`logs/quote-xapi.log`, lines 10–11).
- **15:44:30 UTC:** Log inspection proves total worker monopolization by Mexico: `xapi-0 in-flight by route: quote-papi-mx=61 quote-papi-co=2 quote-papi-cl=1` (`logs/quote-xapi.log`, lines 12–13).
- **15:52 UTC:** On-call engineer declares SEV3 (`chat/incident-channel.md`, line 9).
- **16:02–16:11 UTC:** Response stalls due to false hypotheses (PostgreSQL CPU load and OIDC token key rotation).
- **16:18 UTC:** Channel Manager orders horizontal scale-out of `quote-xapi` from 2 to 4 replicas (`chat/incident-channel.md`, line 15).
- **16:21–16:35 UTC:** Replicas `xapi-2` and `xapi-3` come online. Temporary drop in error rates for CO and CL (`logs/quote-xapi.log`, lines 22–24).
- **16:38:55 UTC:** New replicas are immediately consumed by Mexican requests: `xapi-2 in-flight: quote-papi-mx=62 quote-papi-co=1 quote-papi-cl=1`; `xapi-3: quote-papi-mx=63` (`logs/quote-xapi.log`, lines 25–26). Latency in `quote-papi-mx` climbs over 45 s.
- **16:55 UTC:** Payments Integration squad alerts that the Mexican Salesforce org has breached **85% of its 2,000,000 daily API quota**, triggering their automated circuit breaker to halt payment collection (`chat/incident-channel.md`, line 18).
- **16:58 UTC:** Salesforce Platform team discovers the Channel integration user consumed **856,000 API calls** (budget was 250,000) at **90 requests/sec** querying `Rating_Table__c` (`chat/incident-channel.md`, line 19).
- **17:00 UTC:** Incident Commander orders immediate rollback of `quote-papi-mx` to version 3.7.2 (`chat/incident-channel.md`, line 20).
- **17:05 UTC:** Nimbus completes deployment of 3.7.2 (`chat/incident-channel.md`, line 21).
- **17:20 UTC:** Telemetry normalizes across all three countries. p95 drops below 800 ms. Incident mitigated (`chat/incident-channel.md`, line 22).
- **18:20 UTC:** Salesforce team grants temporary allocation buffer (+500,000 calls). Payments consumers resumed. Incident officially closed (`chat/incident-channel.md`, lines 25–27).

---

## 3. Root Cause and Contributing Factors Analysis

### 3.1 Primary Technical Root Cause
The root cause was the deployment of **`quote-papi-mx` version 3.8.0** by Nimbus Digital on 2026-09-29 at 23:40 UTC (`TIMELINE.md`, line 8). 

As stated in Nimbus's release notes (`TIMELINE.md`, lines 14–19):
1. **Cache Elimination:** Nimbus removed the in-memory rating table cache to *"always use fresh prices"*. Consequently, every individual Mexican quote performed **6 live SOQL queries on `Rating_Table__c` plus 1 Apex rating invocation (7 Salesforce calls per quote)** instead of 1.
2. **Aggressive Uncapped Retries:** Added up to 3 automatic retries on timeouts and 5xx errors.
3. **Trigger Event:** During the month-end peak on September 30, traffic reached ≈ 90 requests/sec. Under 7 queries per quote and 3 retries, Salesforce MX org was bombarded with hundreds of calls per second, far exceeding the platform's **40 req/s** threshold (`SALESFORCE_CONSTRAINTS.md`, line 19). Salesforce response time degraded past 45 s, creating a self-reinforcing **retry storm**.

### 3.2 Systemic & Architectural Contributing Factors
1. **Lack of Bulkhead Isolation in `quote-xapi` (Violates C1):**
   `quote-xapi` shared 2 replicas with 64 in-flight slots each and a single queue of 100 requests across all three countries. When Mexican requests hung for 30 seconds, they occupied 62 out of 64 slots (`logs/quote-xapi.log`, lines 12–13), starving Colombia and Chile and causing catastrophic cross-tenant blast radius.
2. **Excessive Upstream Timeout:**
   `quote-xapi` configured a **30-second outbound timeout** to Process APIs without circuit breakers (`CURRENT_STATE.md`, lines 74–76). This forced threads to remain blocked for 30,000 ms before failing, completely jamming the gateway.
3. **Unmonitored Third-Party Deployment (Violates C8):**
   Nimbus deployed `quote-papi-mx 3.8.0` directly from their proprietary CI pipeline using a shared production credential (`CURRENT_STATE.md`, line 101) without Meridian architectural review, contract testing, performance testing, or Canary gating.
4. **Absence of Rate Limiting and Quota Enforcement on System APIs (Violates C6):**
   No API Gateway policy throttled the channel user to the agreed 40 req/s limit or 250,000 daily budget.

---

### 3.3 Hypotheses Investigated and Ruled Out

During the incident, the engineering team spent 20 minutes chasing two false hypotheses:

| Hypothesis | Evidence Investigated | Why It Was Ruled Out |
|---|---|---|
| **Hypothesis 1: `pg-channel-co` Database Exhaustion** | At 16:02 UTC, on-call noted `pg-channel-co` CPU at 88% (`chat/incident-channel.md`, line 10). | Ruled out at 16:09 UTC by DBA (`chat/incident-channel.md`, line 12). The monthly analytics funnel report runs every month-end between 15:00 and 15:58 UTC (as verified in `TIMELINE.md`, line 9). Postgres logs (`logs/pg-channel-co.log`) showed channel application queries remained under 50 ms. Furthermore, after the report finished at 15:58 and database CPU fell to normal, quote failures in Colombia continued unchanged. |
| **Hypothesis 2: Identity Provider Key Rotation Failure** | At 16:00 UTC, the central identity team rotated the token signing key (`TIMELINE.md`, line 10). | Ruled out at 16:06 UTC (`chat/incident-channel.md`, line 11). Logs (`logs/quote-xapi.log`, lines 16–20) confirm that JWKS cache refreshed at 16:05:02 UTC and 401 errors dropped to zero. However, 503/504 quote failures persisted unabated, proving the key rotation was purely coincidental. |

---

## 4. The 5 Whys

1. **Why did customer quote journeys fail in Colombia and Chile?**  
   Because `quote-xapi` rejected incoming requests with HTTP `503 QUEUE_FULL` and delayed requests by > 11 seconds in queues.
2. **Why was the `quote-xapi` queue full and worker threads exhausted?**  
   Because Mexican quote requests took > 30 seconds to respond, tying up 123 of the 128 total worker slots across replicas due to a lack of tenant bulkhead isolation.
3. **Why did Mexican quote requests take > 30 seconds?**  
   Because `quote-papi-mx` was timing out while attempting to execute dozens of concurrent live Salesforce queries during an active retry storm.
4. **Why was Salesforce MX overwhelmed with requests and why was its daily quota exhausted?**  
   Because `quote-papi-mx 3.8.0` removed rating table caching (multiplying Salesforce calls by 7× per quote) and enabled 3 aggressive retries during month-end peak volume.
5. **Why was such a breaking performance and architectural change deployed to production?**  
   Because Nimbus Digital deployed directly to production from their own CI without architectural review, automated performance gates, Canary verification, or enforcement of the agreed Salesforce integration standards.

---

## 5. Incident Response Analysis: What Helped vs. What Made Things Worse

### What Helped:
1. **Rollback to 3.7.2:** Once identified, reverting to 3.7.2 restored the in-memory cache, immediately dropping Mexican Salesforce traffic from 90 req/s to normal levels and clearing the backlog within 15 minutes.
2. **Cross-Cell Collaboration with Payments & Salesforce:** The Salesforce team's quick grant of a 500,000 temporary API buffer allowed Payments Integration to resume billing operations safely.

### What Made Things Worse:
1. **Misclassification as SEV3:** Downplayed severity, resulting in a 50-minute delay before senior technical leads were summoned.
2. **Horizontal Scaling of `quote-xapi` (16:18 UTC):** Scaling replicas from 2 to 4 provided temporary relief for 10 minutes, but actually worsened downstream congestion: it allowed twice as many Mexican requests to concurrently batter Salesforce, accelerating daily quota burn towards 85%.
3. **Chasing Coincidental Events:** Investigating the PostgreSQL database and IDP key rotation wasted critical diagnostic time.

---

## 6. Corrective and Preventative Action Items (Blameless)

| ID | Action Item Description | Type | Owner | Priority | Target Date | Verification Method |
|---|---|:---:|---|:---:|:---:|---|
| **ACT-01** | **Implement Hard Bulkheads in `quote-xapi`:** Configure dedicated in-flight thread pools and queues per country (MX: 32, CO: 32, CL: 24). | Prevent | Technical Lead (Channel) | **P0 (Critical)** | 2027-01-15 | Chaos test: Inject 100% latency into MX route in UAT; verify CO and CL maintain p95 < 1.0s and 0 errors. |
| **ACT-02** | **Lower Gateway Timeout & Circuit Breaker:** Reduce `quote-xapi` outbound timeout to 3.5 s and configure Resilience4j circuit breakers per country route. | Mitigate | MuleSoft Integration Dev | **P0 (Critical)** | 2027-01-15 | Unit & integration tests simulating downstream timeouts; verify fast-fail within 3.5 s. |
| **ACT-03** | **Enforce 30 req/s Rate Limiter on System APIs:** Apply Spike Control and Rate Limiting policies on `sf-policy-sapi-{ctry}` in MuleSoft API Manager. | Prevent | Salesforce Platform Team | **P0 (Critical)** | 2027-01-20 | Load test driving 100 req/s; verify gateway throttles excess with 429 and preserves SF core stability. |
| **ACT-04** | **Re-establish Rating Table Distributed Cache:** Ingest and cache the 6 rating tables daily in Redis at 05:05 UTC; prohibit live table querying during quotes. | Prevent | Technical Lead / Nimbus | **P0 (Critical)** | 2027-01-26 | Code review & automated load test verifying 0 table queries against Salesforce during quote flows. |
| **ACT-05** | **Revoke Nimbus Direct Production Deployments:** Migrate all Nimbus CI pipelines into Meridian Azure Pipelines; mandate PR approval from Meridian Tech Lead (C8). | Govern | DevOps / SecOps | **P1 (High)** | 2027-01-31 | Decommission shared deployment credentials; test end-to-end pipeline deployment via Meridian Azure DevOps. |
| **ACT-06** | **Implement Real-Time Salesforce Quota Telemetry & Alerts:** Configure automated alerts at 70%, 75%, 80%, and 85% rolling consumption of org quotas. | Detect | Cloud Ops & Salesforce Team | **P1 (High)** | 2027-02-10 | Synthetic telemetry test triggering mock alerts in PagerDuty at designated thresholds. |
| **ACT-07** | **Update Incident Severity Classification Runbooks:** Formally codify that any multi-country degradation of primary journeys is an automatic SEV1. | Process | Incident Management Lead | **P2 (Medium)** | 2027-02-15 | Table-top incident simulation exercise with on-call engineers. |

# Technical Feasibility Analysis — BR-02: GenAI Assistant in Quote Journey

## 1. Executive Answer to Sponsor

> **Sponsor's Ask:** *"I need a yes or a no, and a date."*  
> **Technical Lead's Recommendation:** **NO for Q1 2027 Production Launch; YES for a 4-Week Exploratory Spike in May 2027 (Q2), targeting potential internal/controlled pilot in Q3 2027.**

---

## 2. Business Value, Technical Effort, and Risk Profile

### 2.1 Quantified Business Value (Evidence-Based)
From the evidence provided by the Chief Commercial Officer (`context/BUSINESS_REQUESTS.md`):
- **Abandonment Problem:** Quote abandonment rates are alarmingly high: **68% in Mexico, 61% in Colombia, and 64% in Chile**.
- **Customer Pain Point:** In an exit survey of 2,300 customers, **41% of abandoning users stated they did not understand the insurance coverages**.
- **Contact Center Operational Drain:** Coverage inquiries generate **≈ 18,000 monthly calls** across the three countries (MX: 8,100; CO: 6,900; CL: 3,000). At an average cost of **USD 4.10 per call**, Meridian spends over **USD 73,800 per month (≈ USD 885,000 annually)** answering basic coverage questions.
- **Potential Upside:** Deflecting even 20% of these calls and converting 5% of abandoned quotes into bound policies would generate massive operational savings and revenue.

### 2.2 Missing Information and Blind Spots
1. **Model Latency Impact on User Flow:** LLM token streaming takes 1.5 s – 4.0 s. If integrated synchronously into the quote journey, it risks violating **C2 (< 2.5s p95)** unless designed as an asynchronous floating conversational drawer.
2. **Token Economics & Cost Profile (C9):** No token pricing or inference cost estimates exist for Azure OpenAI in Meridian's tenant under peak loads.
3. **Pre-fill Accuracy:** We lack empirical data on how reliably an LLM can extract structured quote fields (e.g., square meters, postal code) from unstructured natural Spanish text across Mexican, Colombian, and Chilean regional idioms without hallucination.

### 2.3 Critical Regulatory & Security Risks

```mermaid
flowchart TD
    subgraph RegulatoryRisks["Regulatory & Compliance Constraints"]
        R1["Regulated Advice Prohibition:<br/>Local insurance laws in MX, CO, CL treat recommending specific coverage as licensed insurance advice.<br/>Assistant MUST NOT recommend."]
        R2["Binding Representation Liability:<br/>If LLM hallucinates coverage terms, Meridian may be legally bound to cover uninsured losses."]
    end

    subgraph SecurityRisks["Security & Privacy Constraints"]
        S1["Internal-Only Azure LLM Approval:<br/>Security office approved tenant LLM for INTERNAL pilots only.<br/>Customer-facing use strictly prohibited without formal review."]
        S2["PII & Prompt Injection:<br/>Customer entering RFC/Cédula or attempting prompt injection to extract corporate system instructions."]
    end

    RegulatoryRisks --> Decision["Feasibility: Production in Q1 is High-Risk & Blocked"]
    SecurityRisks --> Decision
```

---

## 3. Options Considered

### Option 1 — Deliver Customer-Facing GenAI in Q1 2027 (MX Pilot)
- Attempt to rush an Azure OpenAI RAG feature into the Mexican quote journey by March 2027.
- **Assessment:** **REJECTED.** Direct collision with BR-01 (Home Essentials launch on April 30); violates Security Office mandate (internal-only approval); high risk of legal penalties for giving unauthorized financial advice.

### Option 2 — Reject GenAI Completely and Rely on Static FAQ Tooltips
- Replace the AI concept with enhanced UI tooltips, glossary modal dialogs, and video explainers.
- **Assessment:** **REJECTED.** While safe, it ignores the 41% customer comprehension gap and fails to leverage Meridian's digital differentiator.

### Option 3 — Time-Boxed Spike in Q2 2027 with Strict Guardrails (Recommended)
- Conduct a disciplined 4-week architectural spike in May 2027 (Sprint 9–10) using 1 backend engineer and 1 frontend engineer.
- Prototype a RAG architecture (*Retrieval-Augmented Generation*) restricted strictly to approved PDF product sheets.
- **Assessment:** **ACCEPTED.** Protects BR-01 delivery while establishing safe, compliant AI foundations.

---

## 4. Time-Boxed Spike Plan (May 2027 / Q2 Sprints 9–10)

```mermaid
gantt
    title 4-Week GenAI Architectural Spike
    dateFormat  YYYY-MM-DD
    section Week 1: Knowledge Ingestion
    Index Approved Product PDFs (Azure AI Search) :w1_1, 2027-05-03, 2027-05-07
    Configure System Prompt & Advice Disclaimers   :w1_2, 2027-05-05, 2027-05-09

    section Week 2: Guardrails & RAG Engine
    Implement Azure AI Content Safety & PII Filters :w2_1, 2027-05-10, 2027-05-14
    Build Structured JSON Function Calling Prototype :w2_2, 2027-05-12, 2027-05-16

    section Week 3: Security & Advisor Validation
    Security Red-Teaming & Prompt Injection Testing :w3_1, 2027-05-17, 2027-05-21
    Advisor Portal Blind Evaluation (100 Scenarios) :w3_2, 2027-05-19, 2027-05-23

    section Week 4: Business Evaluation & Decision
    Sponsor Demonstration & Final Go / No-Go Review :crit, w4_1, 2027-05-24, 2027-05-28
```

### 4.1 Scope of the Spike
1. **Curated Knowledge Base:** Ingest the official Spanish PDF policy wordings and product sheets for Home Essentials and Auto into Azure AI Search using hybrid semantic vector search.
2. **Intent Classification & Guardrails:**
   - Classify user queries: `INFORMATIONAL_COVERAGE`, `PREFILL_QUOTE`, or `ADVISORY_REQUEST`.
   - If the user asks *"Which policy should I buy?"*, trigger an automated hard-coded refusal: *"I am an automated assistant and cannot provide insurance advice. Here is an explanation of the differences between Plan A and Plan B..."*
3. **Structured Form Pre-Fill:** Use OpenAI Function Calling to extract validated JSON (`postalCode`, `squareMeters`, `includesContents`) without free-form hallucination.

### 4.2 Measurable Go / No-Go Criteria

At the end of the 4-week spike (28 May 2027), the feature proceeds to a customer pilot **ONLY if all five criteria are satisfied**:

| Criteria | Metric / Threshold | Pass / Fail Condition |
|---|---|---|
| **1. Zero Regulated Advice** | 100% compliance across 50 adversarial legal test prompts designed by Meridian Compliance. | **Pass:** 0 instances of recommending a specific plan or coverage. |
| **2. Factual Grounding (Zero Hallucination)** | Grounding score ≥ **95%** using RAGAS evaluation framework on approved product PDFs. | **Pass:** Zero fabricated coverages, deductibles, or limits. |
| **3. Latency & Responsiveness** | Time to First Token (TTFT) < **1.2 s**; complete response < **3.0 s**. | **Pass:** Conversational UI renders smoothly without freezing the quote form. |
| **4. Security Office Sign-Off** | Formal approval from Meridian Security Office for public customer-facing tenant access. | **Pass:** Documented security architecture approval. |
| **5. Commercial Feasibility** | Cost per completed interaction < **USD 0.05** (vs USD 4.10 for call center). | **Pass:** Proven positive ROI model. |

---

## 5. Mandatory Architecture Controls for Customer-Facing LLMs

If the spike passes, the following production architecture controls are strictly enforced:

```mermaid
flowchart LR
    User["Customer Prompt"] --> FilterIn["Input Guardrail<br/>(Azure AI Content Safety + PII Masking)"]
    FilterIn --> RAG["RAG Orchestrator<br/>(System Prompt + Grounded Search)"]
    RAG --> LLM["Enterprise Azure OpenAI<br/>(Data Retention = Off)"]
    LLM --> FilterOut["Output Verification<br/>(Grounding & Policy Fact Check)"]
    FilterOut --> UI["Conversational Drawer"]

    FilterIn -.->|"Prompt Injection / PII Detected"| Reject["Sanitize / Safe Refusal"]
    FilterOut -.->|"Hallucination / Advisory Detected"| Fallback["Canned Official Summary"]
```

1. **Strict Context Grounding (No Open-Ended Generation):** System prompt instructs the model: *"Answer strictly and exclusively using the retrieved document context below. If the answer is not in the text, reply: 'I do not have information on this specific question; please contact an advisor.' Never speculate or provide financial advice."*
2. **Human Handoff:** A persistent *"Chat with a Licensed Advisor"* button in the assistant window immediately transfers the conversation history to an advisor in the Advisor Portal.
3. **Zero Data Retention for Training:** Enterprise Azure OpenAI agreement confirms that zero customer inputs or conversations are retained or used to train public models.

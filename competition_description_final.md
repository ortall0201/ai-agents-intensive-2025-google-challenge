# VaaS: Vendor-as-a-Service for Secure Enterprise AI

## Problem Statement: The Innovation-Adoption Gap

Modern enterprises need AI services—translation, analysis, document processing, data enrichment. The best tools often come from small developers, rapid prototypers, and "vibe coders" who build highly effective applications quickly. But enterprises cannot consume these innovations because they cannot risk sending sensitive data to uncertified external systems.

**The disconnect:**
- Innovation happens rapidly on the vendor side (small developers, fast iteration)
- Enterprises cannot adopt these tools (no SOC2, no GDPR certification, high liability risk)
- Result: Useful AI tools remain inaccessible to the organizations that need them most

Traditional SaaS forces enterprises to share sensitive data directly with vendors. This activates data-processing obligations, audit requirements, third-party risk assessments, procurement reviews, security questionnaires, and compliance demands. A small developer cannot handle this scrutiny or cost.

**Our breakthrough during Day 5:** What if the enterprise never sends sensitive data in the first place?

---

## Why Agents? The VaaS Solution

### The Revolutionary Insight

**A small developer's app does not need to be "enterprise-ready" if the enterprise controls what leaves their environment.**

This is accomplished using the **A2A (Agent-to-Agent) protocol**, which creates a strict, auditable boundary between organization and vendor. Instead of exposing raw data, the enterprise runs its own agents inside its secure environment. These agents handle intake, PII filtering, OCR, validation, and policy enforcement. **Only sanitized, masked information crosses the boundary.**

The vendor becomes a **capability provider, not a data processor.**

### What VaaS Solves

Traditional SaaS forces high-risk data sharing. VaaS reverses the model:

**Five Barriers Removed:**
1. **Data leakage risk** → Vendor never sees raw PII
2. **Enterprise compliance hurdles** → Filtering stays internal
3. **Vendor liability concerns** → Vendor never processes sensitive data
4. **Integration costs** → A2A is standardized (no custom API integration)
5. **Time-to-adoption barriers** → Enterprise adopts instantly without vendor audits

**Business Model Shift:**
- **Traditional SaaS:** Customer → Sends Raw Data → Vendor Processes → Vendor Liable
- **VaaS with A2A:** Customer's Agent → Filters Data → A2A Boundary → Vendor Capability → Customer Reconstructs

The vendor provides the capability. The enterprise controls the data. A2A enforces the boundary.

---

## What We Created: Cross-Framework VaaS Architecture

### Key Technical Achievement: ADK ↔ CrewAI via A2A

One of our biggest accomplishments is proving **A2A works across entirely different frameworks**:
- **Enterprise agents:** Google ADK (multi-agent orchestration, sessions, observability)
- **Vendor agent:** CrewAI (rapid prototyping, existing translation service)
- **Communication:** A2A protocol (framework-agnostic interoperability)

This demonstrates VaaS is not tied to any vendor, company, or ecosystem. It's a true interoperability layer for agents.

### Enterprise System (Google ADK)

**Multi-Agent Sequential Pipeline:**
```
IntakeAgent (validates, extracts metadata)
    ↓
ProcessingAgent (5-step pipeline)
    ├─ OCR Tool (text extraction)
    ├─ Security Filter (masks 7 PII patterns)
    ├─ RemoteA2aAgent (A2A boundary crossing)
    ├─ Security Verification (checks vendor response)
    └─ Final Compilation (reconstructs output)
```

**Day 1 (Multi-Agent Orchestration):**
```python
processing_agent = LlmAgent(
    model=Gemini(model="gemini-2.0-flash-lite"),
    tools=[ocr_tool, security_filter],
    sub_agents=[remote_vendor_agent],  # A2A boundary!
    output_key="processing_result"
)
```
Sequential workflow with `output_key` pattern passing state between agents.

**Day 2 (Custom Tools - Deterministic Security):**
```python
PII_PATTERNS = {
    "national_id_spain": r"\b\d{3}-\d{2}-\d{4}-[A-Z]\b",
    "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
    "phone": r"\b(\+?\d{1,3}[-.\s]?)...",
    "email": r"\b[A-Za-z0-9._%+-]+@...",
    "credit_card": r"\b\d{4}[-\s]?...",
    "date_of_birth": r"\b\d{1,2}\s+de\s+\w+,?\s+\d{4}\b",
    "passport": r"\b[A-Z]{3}-\d{9}\b"
}
```
**Why regex over LLM?** Deterministic tools are auditable, testable, and compliant. Enterprises require transparent, reproducible PII detection for compliance audits.

**Day 3 (Session Management):**
- `DatabaseSessionService` for persistent workflows (government docs can't lose state)
- `EventsCompactionConfig` handles long document histories
- Session state tracks document IDs through pipeline

**Day 4 (Observability for Compliance):**
```python
class SecurityAuditPlugin(BasePlugin):
    async def after_tool_callback(self, callback_context):
        if callback_context.tool_name == "security_filter":
            self.log_pii_event(callback_context.tool_result)
```
Separate audit trail for compliance. Every PII filter event logged.

**Day 5 (A2A Cross-Framework Integration):**
```python
# Enterprise creates RemoteA2aAgent pointing to CrewAI vendor
remote_vendor = RemoteA2aAgent(
    name="docs_translator_vendor",
    agent_card="http://localhost:8001/.well-known/agent-card.json"
)

processing_agent = LlmAgent(
    sub_agents=[remote_vendor]  # Cross-org, cross-framework boundary!
)
```

### Vendor System (CrewAI + ADK A2A Wrapper)

**The "Vibe-Coded" Translation Service:**
Our team had built a CrewAI-based document translation service—fast, effective, iterative. But uncertified and unable to sell to enterprises.

**Exposing via A2A (Day 5):**
```python
from adk.a2a import to_a2a

# Wrap existing CrewAI agent
vendor_agent = crewai_to_adk_wrapper(crew_translator)

# Expose via A2A protocol
a2a_app = to_a2a(vendor_agent, port=8001)
uvicorn.run(a2a_app, host="localhost", port=8001)

# Automatically serves:
# - Agent Card: /.well-known/agent-card.json
# - Streaming: /streams
# - Capability discovery
```

The vendor doesn't understand the enterprise's internal workflow. It receives masked input, processes it, returns structured output. **The vendor sees only safe, sanitized text.**

---

## Demo: Spanish Birth Certificate Translation

**Input Document:**
```
CERTIFICADO DE NACIMIENTO
DNI: 123-45-6789-X
Email: maria.garcia@ejemplo.es
Fecha: 15 de marzo, 1990
```

**Execution Flow (Cross-Framework):**
```bash
# Terminal 1: Start CrewAI vendor via A2A
python -m vendor.vendor_server  # CrewAI agent wrapped with to_a2a()

# Terminal 2: Run ADK enterprise pipeline
python main.py  # ADK agents with RemoteA2aAgent
```

**What Happens:**
```
[IntakeAgent - ADK] Document validated

[ProcessingAgent - ADK] Pipeline:
1. [OCR] Extracted 247 characters
2. [Security Filter - PRE] Masking PII...
   ✓ DNI: 123-45-6789-X → ***-**-****-X
   ✓ Email: maria.garcia@ejemplo.es → m*************@ejemplo.es
   ✓ Date: 15 de marzo, 1990 → XX de XXXX, 1990

3. [RemoteA2aAgent] A2A call to CrewAI vendor...
   → GET http://localhost:8001/.well-known/agent-card.json
   → POST /streams with masked content
   ✓ CrewAI translation received (3.2s)

4. [Security Filter - POST] Verifying vendor response...
   ✓ No PII leakage detected

5. [Compilation] Reconstructing final document
```

**CrewAI vendor never saw:** Full national ID, complete email, exact birth date.

---

## Technical Decisions

**Why Sequential?** PII filtering must happen before vendor call. Can't translate before OCR. Dependencies enforce security.

**Why Regex over LLM for PII?** Compliance requires deterministic, auditable filtering. Enterprises need unit tests and transparent pattern matching.

**Why DatabaseSessionService?** Government workflows can't lose state on restart.

**Why A2A over REST?** Standardized protocol (any framework), self-documenting (Agent Card), streaming built-in, future-proof.

**Course Concepts Applied:**
- ✅ Day 1: Sequential agents, multi-agent orchestration, output_key
- ✅ Day 2: Custom function tools, deterministic security filtering
- ✅ Day 3: DatabaseSessionService, session state, context compaction
- ✅ Day 4: LoggingPlugin, SecurityAuditPlugin for compliance
- ✅ Day 5: **RemoteA2aAgent, to_a2a(), cross-framework A2A**

---

## Business Innovation: VaaS Economics

| Cost | Traditional SaaS | VaaS with A2A |
|------|-----------------|---------------|
| SOC 2 Certification | $100K+ | Not required* |
| GDPR Compliance | Legal team + DPO | Simplified** |
| Liability Insurance | $50K/year | Much lower*** |
| Time to Enterprise | 12+ months | Weeks |

*Vendor doesn't process PII, so data processor requirements don't apply
**Customer is data controller, vendor is capability provider
***Clear liability boundary at A2A protocol layer

**For thousands of AI developers, this removes the barrier to enterprise sales.**

---

## If We Had More Time

**Short-term:** Multi-language support, Vertex AI Agent Engine deployment, Memory Bank for translation preferences.

**Mid-term:** Multiple document types (passports, medical records), cryptographic signatures for filtered content, customer dashboard.

**Long-term:** **VaaS Marketplace** where AI developers expose capabilities via A2A. Auto-compliance documentation generator. Open-source `vaas-client` toolkit for enterprise deployment.

**Our Vision:** Every "vibe-coded" AI prototype can be safely consumed by enterprises. Instead of $100K compliance costs blocking small developers, A2A protocol enables thousands of innovators to participate in enterprise ecosystems.

---

## Conclusion: Bridging Innovation and Adoption

**The problem we solved:** Small developers innovate rapidly but cannot sell to enterprises. Enterprises need innovation but cannot risk data exposure.

**Our solution:** VaaS redefines the vendor-customer relationship. The vendor provides capabilities. The enterprise controls data. A2A enforces the boundary.

**The technical breakthrough:** We proved A2A works across frameworks (ADK ↔ CrewAI). This shows VaaS is not ecosystem-dependent—it's a universal interoperability layer.

**The business impact:** Removes five major barriers (data leakage, compliance hurdles, vendor liability, integration costs, adoption delays). Enables thousands of developers to sell to enterprises safely.

**The bigger vision:** A world where "vibe-coded" prototypes can be elevated into enterprise workflows if the boundary is managed correctly. Innovation becomes accessible again.

**This is not just a technical solution—it's a new model for AI adoption.**

**Repository:** https://github.com/ortall0201/enterprise-gov-docs-a2a-capstone

**Technologies:** Google ADK, CrewAI, Gemini 2.0 Flash Lite, A2A Protocol, RemoteA2aAgent, to_a2a(), DatabaseSessionService, LoggingPlugin, FastAPI

Thank you to the Kaggle and Google teams for this incredible course. Day 5's A2A protocol lesson unlocked a solution that bridges the innovation-adoption gap in enterprise AI. 🙏

---

**Built with ❤️ during the Kaggle AI Agents Intensive**

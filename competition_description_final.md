# VaaS: Vendor-as-a-Service for Secure Enterprise AI

## Problem Statement: The $100K Compliance Wall

As a team, one of us had built a document translation service using CrewAI. When we approached enterprises and government agencies to sell it, we hit an insurmountable barrier:

**"Do you have SOC 2 Type II certification? GDPR compliance documentation? Liability insurance for PII exposure?"**

The answer was no. These certifications cost $100K+ and take 12+ months. **We couldn't afford to sell our AI service to the customers who needed it most.**

Then, during Day 5 of this course, we had a breakthrough: **What if the customer never sends us their PII in the first place?**

---

## Why Agents? The Day 5 Breakthrough

### The Learning Journey

**Days 1-4:** We built the foundation. Day 1 taught us multi-agent orchestration—we created `IntakeAgent` and `ProcessingAgent` using sequential workflow patterns. Day 2 showed us how to build custom function tools for OCR extraction and PII filtering, plus the `BuiltInCodeExecutor` pattern. Day 3 introduced `DatabaseSessionService` for persistent workflows and context compaction for handling large document histories. Day 4 taught observability—`LoggingPlugin` for standard traces and custom callbacks for security audit trails.

**Day 5: The "Aha!" Moment**

Then came the A2A (Agent2Agent) protocol lesson. The product catalog example showed:

```
Customer Support Agent → A2A → Product Catalog Agent (External Vendor)
```

The customer's internal agent called an external vendor via A2A. **The vendor never accessed the customer's systems. The customer controlled what data crossed the boundary.**

**That's when it clicked:**

**Traditional SaaS:** Customer → Sends Raw PII → Vendor Processes → Vendor Liable

**VaaS with A2A:** Customer's Agent → Filters PII → A2A → Vendor's Agent → Vendor Never Sees Raw PII

The customer deploys `RemoteA2aAgent` (their code, their responsibility). They filter PII before calling our service. **We're tool providers, not data processors.** Clear liability boundary.

This changes everything. We can sell to enterprises **without $100K in compliance costs** because:
- Customer deploys their own RemoteA2aAgent
- They filter PII pre-vendor
- We only provide translation capabilities
- No direct PII access = simplified compliance

---

## What We Created: Two-Sided VaaS Architecture

### Enterprise System (Customer Side)

**Multi-Agent Sequential Pipeline:**
```
IntakeAgent (validates document)
    ↓
ProcessingAgent (5-step pipeline)
    ├─ OCR Tool (extracts text)
    ├─ Security Filter (masks 7 PII patterns)
    ├─ RemoteA2aAgent (A2A boundary crossing)
    ├─ Security Verification (checks response)
    └─ Final Compilation
```

**Key Components with Course Concepts:**

**1. IntakeAgent** (Day 1: Basic agent with tools)
- Custom tool: `validate_document()` checks file integrity
- Generates document ID stored in session state (Day 3)

**2. ProcessingAgent** (Day 1: Sequential workflow + Day 5: A2A integration)
```python
processing_agent = LlmAgent(
    model=Gemini(model="gemini-2.0-flash-lite"),
    tools=[ocr_tool, security_filter],
    sub_agents=[remote_vendor_agent],  # Day 5: A2A boundary!
    output_key="processing_result"
)
```
Uses Day 1's `output_key` pattern to pass state between steps.

**3. Security Layer** (Day 2: Custom tool with deterministic regex)
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
**Why regex over LLM?** Day 4's observability lesson taught us: deterministic tools are debuggable. We can log exact matches, write unit tests, and pass compliance audits.

**4. Session Management** (Day 3)
- `DatabaseSessionService` with SQLite for workflow persistence
- `EventsCompactionConfig` handles long document histories
- Session state tracks document IDs through pipeline

**5. Observability** (Day 4)
```python
# Standard traces
plugins=[LoggingPlugin()]

# Custom security audit
class SecurityAuditPlugin(BasePlugin):
    async def after_tool_callback(self, callback_context):
        if callback_context.tool_name == "security_filter":
            self.log_pii_event(callback_context.tool_result)
```

### Vendor System (Docs Translator Service)

**Day 5: Agent Exposed via A2A**
```python
from adk.a2a import to_a2a

vendor_agent = LlmAgent(
    model=Gemini(model="gemini-2.0-flash-lite"),
    tools=[translate_document, validate_translation]
)

# Expose via A2A protocol
a2a_app = to_a2a(vendor_agent, port=8001)
uvicorn.run(a2a_app, host="localhost", port=8001)

# Serves:
# - Agent Card: /.well-known/agent-card.json
# - Streaming: /streams
```

**Customer Connects via RemoteA2aAgent:**
```python
remote_vendor = RemoteA2aAgent(
    name="docs_translator_vendor",
    agent_card="http://localhost:8001/.well-known/agent-card.json"
)

processing_agent = LlmAgent(
    sub_agents=[remote_vendor]  # Cross-org boundary!
)
```

---

## Demo: Spanish Birth Certificate Translation

**Input Document:**
```
CERTIFICADO DE NACIMIENTO
DNI: 123-45-6789-X
Fecha: 15 de marzo, 1990
Email: maria.garcia@ejemplo.es
```

**Execution Flow:**

```bash
# Terminal 1: Start vendor A2A service
python -m vendor.vendor_server

# Terminal 2: Run customer pipeline
python main.py
```

**What Happens (with Day 4 observability logs):**

```
[IntakeAgent] Document validated: doc_1234567890

[ProcessingAgent] Sequential pipeline:
1. [OCR] Extracted 247 characters
2. [Security Filter - PRE] Masking PII...
   ✓ DNI: 123-45-6789-X → ***-**-****-X
   ✓ Email: maria.garcia@ejemplo.es → m*************@ejemplo.es
   ✓ Date: 15 de marzo, 1990 → XX de XXXX, 1990

3. [RemoteA2aAgent] A2A call...
   → Agent card: http://localhost:8001/.well-known/agent-card.json
   → POST /streams with masked content
   ✓ Translation received (3.2s)

4. [Security Filter - POST] Verifying...
   → Risk score: 0.0 (SAFE)
   ✓ No PII leakage

5. [Compilation] Complete
```

**Vendor Never Saw:** Full national ID, complete email, exact birth date

**Why This Matters:** Government processes sensitive documents, vendor provides expertise, clear liability at A2A boundary.

---

## The Build: Technical Decisions

**Why Sequential over Parallel?** Each step depends on previous (can't translate before OCR, can't filter after vendor call).

**Why Custom Tools over MCP?** PII filtering requires deterministic behavior. MCP excellent for general integrations, but security-critical code needs unit tests.

**Why SQLite DatabaseSessionService?** Government document processing can't lose state on restart. InMemorySessionService would lose everything.

**Why A2A over REST API?**
1. Standardized (any A2A client works)
2. Self-documenting via Agent Card
3. Streaming support built-in
4. Future-proof as A2A ecosystem grows

**Course Concepts Applied:**
- ✅ Day 1: Sequential agents, multi-agent delegation, output_key
- ✅ Day 2: Custom function tools, AgentTool pattern, code execution
- ✅ Day 3: DatabaseSessionService, session state, context compaction
- ✅ Day 4: LoggingPlugin, custom callbacks, ADK web debugging
- ✅ Day 5: RemoteA2aAgent, to_a2a(), Agent Card, cross-org boundaries

---

## Business Innovation: VaaS Economics

**Traditional SaaS vs VaaS:**

| Cost | Traditional SaaS | VaaS with A2A |
|------|-----------------|---------------|
| SOC 2 Certification | $100K+ | Not required* |
| GDPR Compliance | Legal team + DPO | Simplified** |
| Liability Insurance | $50K/year | Much lower*** |
| Time to Enterprise | 12+ months | Weeks |

*Vendor doesn't process PII, so data processor requirements don't apply
**Customer is data controller, vendor is tool provider
***Clear liability boundary at A2A protocol layer

**For small teams and startups, this is game-changing.**

---

## If We Had More Time

**Short-term:** Multi-language support (French, German, Arabic), Vertex AI Agent Engine deployment (Day 5b), Memory Bank integration for translation preferences.

**Mid-term:** Multiple document types (passports, medical records), customer dashboard, cryptographic signatures for filtered content.

**Long-term:** VaaS Marketplace where AI vendors expose capabilities via A2A, auto-compliance documentation generator, open-source `vaas-client` toolkit.

**Our Vision:** Every AI startup becomes a VaaS provider. Instead of $100K compliance costs blocking small teams from enterprise sales, A2A protocol democratizes enterprise AI.

---

## Conclusion: From Learning to Innovation

**Before this course:** Good AI service, couldn't sell to enterprises.

**After Day 5:** We realized the solution isn't certifications—it's architecture that doesn't need them.

**The progression:**
1. Day 1: Build agents that work
2. Day 2: Give them tools
3. Day 3: Make them remember
4. Day 4: Make them observable
5. Day 5: **Make them cross boundaries**

That last step unlocked VaaS. A2A isn't just about calling external agents—**it's about creating a liability boundary that makes enterprise AI accessible to small vendors.**

**VaaS is a new business model** where small AI vendors provide specialized capabilities via A2A, customers deploy RemoteA2aAgent clients, and clear separation of concerns enables both to thrive.

**Repository:** https://github.com/ortall0201/enterprise-gov-docs-a2a-capstone

**Technologies:** Google ADK, Gemini 2.0 Flash Lite, A2A Protocol, RemoteA2aAgent, to_a2a(), DatabaseSessionService, LoggingPlugin, FastAPI, SQLite

Thank you to the Kaggle and Google teams for this incredible course that enabled not just learning to code agents, but learning to think differently about how AI services can be delivered. 🙏

---

**Built with ❤️ during the Kaggle AI Agents Intensive**

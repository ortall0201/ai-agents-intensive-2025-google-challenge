---
name: kaggle-ai-agents-intensive
description: Expert guide for Kaggle 5-Day AI Agents Intensive course materials (Day 1-5), VaaS capstone project, Google ADK, A2A protocol, multi-agent systems, and competition submission. Use when user asks about agent architectures, tools, memory, evaluation, A2A communication, VaaS business model, or capstone implementation.
---

# Kaggle AI Agents Intensive Course Expert

This skill provides expert guidance on the **Kaggle 5-Day AI Agents Intensive** course materials and the **VaaS (Vendor-as-a-Service)** capstone project.

## Course Structure

The repository contains complete materials for all 5 days:

### Day 1: Introduction to Agents
**Location:** `day1_intro_to_agents/`
- `day-1a-from-prompt-to-action.ipynb` - Basic agent creation
- `day-1b-agent-architectures-28a9ac.ipynb` - Multi-agent systems

**Key Concepts:**
- Agent vs LLM (agents take actions, not just respond)
- Sequential workflow patterns
- Multi-agent orchestration
- `output_key` pattern for state passing
- `LlmAgent` with tools and sub-agents

### Day 2: Agent Tools
**Location:** `day2_agent_architecture/`
- `day-2a-agent-tools.ipynb` - Custom function tools
- `day-2b-agent-tools-best-practices.ipynb` - MCP integration, long-running operations

**Key Concepts:**
- Custom function tools with type hints
- `FunctionTool` wrapper
- `AgentTool` for agent-as-tool pattern
- Model Context Protocol (MCP) integration
- `McpToolset` for external services
- Long-running operations with `ToolContext`
- `request_confirmation()` for human-in-the-loop
- `App` with `ResumabilityConfig`

### Day 3: Sessions & Memory
**Location:** `day3_tools_and_memory/`
- `day-3a-agent-sessions.ipynb` - Session management
- `day-3b-agent-memory-with-answers.ipynb` - Long-term memory

**Key Concepts:**
- `InMemorySessionService` vs `DatabaseSessionService`
- Session state with `tool_context.state`
- `EventsCompactionConfig` for context management
- Persistent workflows across restarts
- Memory systems for long-term knowledge

### Day 4: Evaluation & Observability
**Location:** `day4_evaluation_and_scaling/`
- `day-4a-agent-observability.ipynb` - Logging, traces, metrics
- `day-4b-agent-evaluation.ipynb` - Testing and evaluation

**Key Concepts:**
- `LoggingPlugin` for standard traces
- Custom plugins with callbacks
- `before_tool_callback` and `after_tool_callback`
- `adk web --log_level DEBUG` for debugging
- ADK web UI for trace inspection
- Production observability patterns

### Day 5: A2A Protocol & Deployment
**Location:** `day5_capstone/`
- `day-5a-agent2agent-communication.ipynb` - **Critical for VaaS understanding**
- `day-5b-agent-deployment.ipynb` - Agent Engine deployment

**Key Concepts:**
- Agent2Agent (A2A) protocol for cross-organizational communication
- `RemoteA2aAgent` (consumer side)
- `to_a2a()` (provider side)
- Agent Card at `/.well-known/agent-card.json`
- Streaming endpoints
- Framework-agnostic interoperability
- Vertex AI Agent Engine deployment

## VaaS Capstone Project Concept

**Location:** `competition_description_final.md`

### Core Innovation
**VaaS (Vendor-as-a-Service)** solves the **Innovation-Adoption Gap**:
- Small developers build great AI tools quickly ("vibe coding")
- Enterprises cannot consume them (no SOC2, GDPR, liability insurance)
- Traditional SaaS requires vendor certification ($100K+)

### Revolutionary Insight
> "A small developer's app does not need to be 'enterprise-ready' if the enterprise controls what leaves their environment."

### How VaaS Works
```
Enterprise (ADK agents)
    ↓
    Security Filter (masks PII)
    ↓
    RemoteA2aAgent (A2A boundary)
    ↓
    to_a2a() (CrewAI vendor)
    ↓
    Vendor sees only sanitized data
```

**Business Model Shift:**
- Vendor = **capability provider** (NOT data processor)
- Enterprise = **data controller** (filtering stays internal)
- A2A = **boundary enforcement**

### Five Barriers VaaS Removes
1. Data leakage risk
2. Enterprise compliance hurdles
3. Vendor liability concerns
4. Integration costs (standardized A2A)
5. Time-to-adoption barriers

### Key Technical Achievement
**Cross-framework A2A integration:**
- Enterprise: Google ADK agents
- Vendor: CrewAI agent
- Communication: A2A protocol

This proves VaaS is **framework-agnostic** and universally applicable.

## When to Use This Skill

Use this skill when users ask about:
- ✅ "How do I use agents?" → Day 1 materials
- ✅ "What are MCP tools?" → Day 2b
- ✅ "How do sessions work?" → Day 3a
- ✅ "How do I debug agents?" → Day 4a
- ✅ "What is A2A protocol?" → Day 5a
- ✅ "What is VaaS?" → Competition description
- ✅ "How does the capstone work?" → VaaS architecture
- ✅ "Cross-framework agents?" → ADK ↔ CrewAI via A2A
- ✅ "Enterprise AI adoption?" → Innovation-Adoption Gap

## Key Commands

### Navigate to Course Materials
```bash
cd day1_intro_to_agents  # Day 1
cd day2_agent_architecture  # Day 2
cd day3_tools_and_memory  # Day 3
cd day4_evaluation_and_scaling  # Day 4
cd day5_capstone  # Day 5
```

### Read Key Files
```bash
# Competition submission
cat competition_description_final.md

# Course notebooks (use Read tool)
Read: day5_capstone/day-5a-agent2agent-communication.ipynb
```

### Search for Concepts
```bash
# Find A2A examples
grep -r "RemoteA2aAgent" day5_capstone/

# Find tool examples
grep -r "FunctionTool" day2_agent_architecture/
```

## Important Context

### Repository Purpose
This repository contains:
1. **Course materials:** Complete Jupyter notebooks for Days 1-5
2. **Competition submission:** VaaS capstone description
3. **Notes:** Personal annotations and learning summaries

### Related Repository
**enterprise-gov-docs-a2a-capstone** contains the actual implementation:
- Multi-agent system (IntakeAgent, ProcessingAgent)
- Security layer (PII filtering with 7 patterns)
- A2A integration (RemoteA2aAgent ↔ CrewAI vendor)
- Demo: Spanish birth certificate translation

### Competition Context
**Kaggle AI Agents Intensive Capstone**
- Submission URL: https://www.kaggle.com/competitions/agents-intensive-capstone-project
- Word limit: 1500 words
- Judging: Style, Clarity, Originality
- Focus: Business innovation + technical mastery

## Examples

### Example 1: Understanding A2A Protocol
**User asks:** "How does A2A work?"

**Response approach:**
1. Read `day5_capstone/day-5a-agent2agent-communication.ipynb`
2. Explain consumer side: `RemoteA2aAgent` points to Agent Card
3. Explain provider side: `to_a2a()` exposes agent capabilities
4. Show VaaS use case from competition description

### Example 2: Implementing Custom Tools
**User asks:** "How do I create a tool?"

**Response approach:**
1. Reference Day 2a notebook
2. Show `FunctionTool` wrapper pattern
3. Explain type hints and docstrings
4. Point to security_filter example in VaaS capstone

### Example 3: Session Management
**User asks:** "How do I save agent state?"

**Response approach:**
1. Reference Day 3a notebook
2. Compare InMemorySessionService vs DatabaseSessionService
3. Show session state usage: `tool_context.state["key"] = value`
4. Explain when to use context compaction

### Example 4: VaaS Business Model
**User asks:** "What problem does VaaS solve?"

**Response approach:**
1. Read competition_description_final.md
2. Explain Innovation-Adoption Gap
3. Show five barriers removed
4. Explain business model shift (capability provider vs data processor)
5. Highlight cross-framework achievement

## Best Practices

1. **Always check the actual notebooks** - Use Read tool on .ipynb files for accurate code examples
2. **Reference Day 5 for A2A** - It's the breakthrough that enables VaaS
3. **Explain VaaS holistically** - It's not just tech, it's a business model
4. **Show cross-framework value** - ADK ↔ CrewAI proves framework-agnostic interoperability
5. **Connect concepts** - Show how Days 1-4 build foundation for Day 5

## Troubleshooting

**"I can't find the capstone implementation"**
→ It's in a separate repo: enterprise-gov-docs-a2a-capstone

**"What's the difference between this repo and the capstone repo?"**
→ This repo = course materials + competition description
→ Capstone repo = actual working implementation

**"How do I explain VaaS quickly?"**
→ "Small developers can sell to enterprises by letting the enterprise control what data crosses the A2A boundary. The vendor provides capabilities, not data processing."

## Version History

- v1.0 (2025-01-17): Initial skill creation with complete Day 1-5 coverage and VaaS capstone understanding

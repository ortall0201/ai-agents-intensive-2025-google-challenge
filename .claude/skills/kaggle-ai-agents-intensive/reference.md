# Quick Reference Guide

## Google ADK Key Patterns

### Basic Agent Creation
```python
from google.adk.agents import LlmAgent
from google.adk.models.google_llm import Gemini

agent = LlmAgent(
    model=Gemini(model="gemini-2.0-flash-lite"),
    name="my_agent",
    description="What this agent does",
    instruction="Detailed instructions for the agent",
    tools=[tool1, tool2],
    sub_agents=[other_agent],
    output_key="result_key"
)
```

### Custom Function Tool
```python
from google.adk.tools.function_tool import FunctionTool

def my_tool(param: str, tool_context: ToolContext) -> dict:
    """Tool description for agent.

    Args:
        param: Parameter description

    Returns:
        Result dictionary
    """
    return {"status": "success", "result": param}

tool = FunctionTool(func=my_tool)
```

### A2A Consumer (RemoteA2aAgent)
```python
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent

remote_vendor = RemoteA2aAgent(
    name="vendor_agent",
    description="External vendor via A2A",
    agent_card="http://vendor:8001/.well-known/agent-card.json"
)

# Use as sub-agent
main_agent = LlmAgent(
    ...,
    sub_agents=[remote_vendor]
)
```

### A2A Provider (to_a2a)
```python
from adk.a2a import to_a2a

vendor_agent = LlmAgent(...)
a2a_app = to_a2a(vendor_agent, port=8001)

import uvicorn
uvicorn.run(a2a_app, host="localhost", port=8001)
```

### Session Management
```python
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner

session_service = DatabaseSessionService(db_url="sqlite:///sessions.db")

runner = Runner(
    agent=my_agent,
    session_service=session_service,
    app_name="my_app"
)

# Run with session
await runner.run_async(
    user_id="user123",
    session_id="session456",
    new_message=query
)
```

### Observability
```python
from google.adk.plugins.logging_plugin import LoggingPlugin
from google.adk.plugins.base_plugin import BasePlugin

# Standard logging
runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]
)

# Custom plugin
class MyPlugin(BasePlugin):
    async def before_tool_callback(self, *, agent, callback_context):
        print(f"Calling tool: {callback_context.tool_name}")

    async def after_tool_callback(self, *, agent, callback_context):
        print(f"Tool completed: {callback_context.tool_name}")
```

## VaaS Architecture Pattern

```
┌─────────────────────────────────────────┐
│  ENTERPRISE (Customer Side)             │
│                                         │
│  ┌──────────────┐                      │
│  │ IntakeAgent  │ Validates document    │
│  └──────┬───────┘                      │
│         │                               │
│         ▼                               │
│  ┌──────────────────┐                  │
│  │ ProcessingAgent  │                  │
│  │  Tools:          │                  │
│  │  - ocr_tool      │                  │
│  │  - security_filter (PRE)            │
│  │  Sub-agents:     │                  │
│  │  - RemoteA2aAgent ────────┐         │
│  │  - security_filter (POST) │         │
│  └──────────────────┘        │         │
│                               │         │
│         [PII FILTERED DATA]   │         │
└───────────────────────────────┼─────────┘
                                │
                   A2A PROTOCOL │
                                │
┌───────────────────────────────┼─────────┐
│  VENDOR (Provider Side)       │         │
│                               ▼         │
│  ┌────────────────────────────────────┐ │
│  │  Vendor Agent (CrewAI/ADK)         │ │
│  │  Exposed via to_a2a()              │ │
│  │                                    │ │
│  │  Tools:                            │ │
│  │  - translate_document()            │ │
│  │  - validate_translation()          │ │
│  └────────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘
```

## Day-by-Day Key Takeaways

### Day 1
- Agents = LLM + Actions (tools, sub-agents)
- Sequential workflows with output_key
- Multi-agent orchestration patterns

### Day 2
- Custom function tools with FunctionTool
- MCP integration with McpToolset
- Long-running operations with request_confirmation()
- Agent-as-tool with AgentTool

### Day 3
- InMemorySessionService (dev) vs DatabaseSessionService (prod)
- Session state: tool_context.state["key"] = value
- EventsCompactionConfig for large histories
- Memory systems for long-term knowledge

### Day 4
- LoggingPlugin for standard traces
- Custom plugins with before/after callbacks
- adk web --log_level DEBUG for debugging
- Production observability patterns

### Day 5
- A2A protocol for cross-org communication
- RemoteA2aAgent (consumer) + to_a2a() (provider)
- Agent Card for capability discovery
- Framework-agnostic interoperability (ADK ↔ CrewAI)

## VaaS Core Concepts

### The Problem
**Innovation-Adoption Gap:** Small developers build great AI tools but can't sell to enterprises due to compliance requirements ($100K+ for SOC2, GDPR, liability insurance).

### The Solution
**VaaS with A2A:** Enterprise controls data boundary. Vendor provides capabilities only.

### Five Barriers Removed
1. **Data leakage risk** → PII filtered before vendor
2. **Compliance hurdles** → Filtering stays internal
3. **Vendor liability** → No sensitive data processing
4. **Integration costs** → Standardized A2A
5. **Adoption delays** → No vendor audits needed

### Business Model Shift
- **Traditional SaaS:** Vendor = data processor (requires certification)
- **VaaS:** Vendor = capability provider (no certification needed)

### Key Achievement
**Cross-framework A2A:** Proved ADK enterprise agents can call CrewAI vendor agents via A2A protocol. Framework-agnostic interoperability.

## File Locations Map

```
ai-agents-intensive-2025-google-challenge/
├── day1_intro_to_agents/
│   ├── day-1a-from-prompt-to-action.ipynb
│   └── day-1b-agent-architectures-28a9ac.ipynb
├── day2_agent_architecture/
│   ├── day-2a-agent-tools.ipynb
│   └── day-2b-agent-tools-best-practices.ipynb
├── day3_tools_and_memory/
│   ├── day-3a-agent-sessions.ipynb
│   └── day-3b-agent-memory-with-answers.ipynb
├── day4_evaluation_and_scaling/
│   ├── day-4a-agent-observability.ipynb
│   └── day-4b-agent-evaluation.ipynb
├── day5_capstone/
│   ├── day-5a-agent2agent-communication.ipynb ⭐ Critical for VaaS
│   └── day-5b-agent-deployment.ipynb
└── competition_description_final.md ⭐ VaaS explanation
```

## Common Search Patterns

```bash
# Find A2A examples
grep -r "RemoteA2aAgent" .

# Find tool patterns
grep -r "FunctionTool" .

# Find session examples
grep -r "DatabaseSessionService" .

# Find observability patterns
grep -r "LoggingPlugin" .
```

# Practical Examples

## Example 1: Navigating to Day 5 A2A Content

**User asks:** "Show me how A2A works"

**Response:**
```bash
# Navigate to Day 5
cd day5_capstone

# Read the A2A notebook
Read: day-5a-agent2agent-communication.ipynb
```

**Explanation to provide:**
1. A2A protocol enables cross-organizational agent communication
2. Consumer side uses `RemoteA2aAgent` pointing to Agent Card
3. Provider side uses `to_a2a()` to expose capabilities
4. VaaS uses this to create security boundary between enterprise and vendor

## Example 2: Understanding VaaS Business Model

**User asks:** "What is VaaS and why does it matter?"

**Response:**
```bash
# Read the competition description
cat competition_description_final.md
```

**Key points to explain:**
1. **Problem:** Small developers can't sell to enterprises (no SOC2, $100K+ barrier)
2. **Insight:** "Enterprise controls what leaves their environment"
3. **Solution:** A2A protocol creates security boundary
4. **Impact:** Vendor = capability provider (NOT data processor)
5. **Achievement:** ADK ↔ CrewAI cross-framework integration

## Example 3: Finding Tool Implementation Examples

**User asks:** "How do I create a custom tool for my agent?"

**Response:**
```bash
# Read Day 2a for basic tools
Read: day2_agent_architecture/day-2a-agent-tools.ipynb

# Search for FunctionTool examples
grep -r "FunctionTool" day2_agent_architecture/
```

**Code example to show:**
```python
from google.adk.tools.function_tool import FunctionTool

def validate_document(document_path: str) -> dict:
    """Validates a document file.

    Args:
        document_path: Path to document

    Returns:
        Validation result
    """
    # Validation logic
    return {"status": "success", "valid": True}

# Wrap in FunctionTool
tool = FunctionTool(func=validate_document)

# Add to agent
agent = LlmAgent(
    tools=[tool],
    ...
)
```

## Example 4: Session Management Pattern

**User asks:** "How do I persist agent conversations?"

**Response:**
```bash
# Read Day 3a for session management
Read: day3_tools_and_memory/day-3a-agent-sessions.ipynb
```

**Code example to show:**
```python
from google.adk.sessions import DatabaseSessionService
from google.adk.runners import Runner

# Create persistent session service
session_service = DatabaseSessionService(
    db_url="sqlite:///my_sessions.db"
)

# Create runner with session service
runner = Runner(
    agent=my_agent,
    session_service=session_service,
    app_name="my_app"
)

# Run with persistent session
await runner.run_async(
    user_id="user123",
    session_id="conv456",
    new_message=query
)

# Session survives restarts!
```

**When to use:**
- Government workflows (can't lose state)
- Long conversations (need history)
- Multi-step processes (track progress)

## Example 5: VaaS Architecture Implementation

**User asks:** "How does the VaaS capstone actually work?"

**Response:**

**Enterprise Side (ADK):**
```python
from google.adk.agents import LlmAgent
from google.adk.agents.remote_a2a_agent import RemoteA2aAgent

# Create RemoteA2aAgent for vendor
remote_vendor = RemoteA2aAgent(
    name="docs_translator",
    agent_card="http://localhost:8001/.well-known/agent-card.json"
)

# Processing agent with security filtering
processing_agent = LlmAgent(
    name="processing_agent",
    tools=[
        ocr_tool,              # Extract text
        security_filter,       # Mask PII (PRE)
    ],
    sub_agents=[
        remote_vendor          # A2A boundary crossing!
    ],
    instruction="""
    1. Extract text with OCR
    2. Mask PII before vendor call
    3. Send to vendor via A2A
    4. Verify vendor response
    5. Reconstruct final output
    """
)
```

**Vendor Side (CrewAI wrapped in ADK):**
```python
from adk.a2a import to_a2a
import uvicorn

# CrewAI translation agent
vendor_agent = create_translation_agent()

# Expose via A2A
a2a_app = to_a2a(vendor_agent, port=8001)

# Run server
uvicorn.run(a2a_app, host="localhost", port=8001)

# Automatically serves:
# - Agent Card: /.well-known/agent-card.json
# - Streaming: /streams
```

**Security Layer (7 PII patterns):**
```python
PII_PATTERNS = {
    "national_id_spain": r"\b\d{3}-\d{2}-\d{4}-[A-Z]\b",
    "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
    "phone": r"\b(\+?\d{1,3}[-.\s]?)?\(?\d{2,3}\)?[-.\s]?\d{3,4}[-.\s]?\d{3,4}\b",
    "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
    "credit_card": r"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b",
    "date_of_birth": r"\b\d{1,2}\s+de\s+\w+,?\s+\d{4}\b",
    "passport": r"\b[A-Z]{3}-\d{9}\b"
}

def security_filter(text: str, mode: str = "mask") -> dict:
    # Deterministic regex filtering for compliance
    # Returns masked text + audit log
    pass
```

## Example 6: Debugging with Observability

**User asks:** "My agent isn't working, how do I debug it?"

**Response:**
```bash
# Read Day 4a for observability patterns
Read: day4_evaluation_and_scaling/day-4a-agent-observability.ipynb
```

**Debugging approach:**
```python
# 1. Add LoggingPlugin
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]
)

# 2. Use adk web with DEBUG logs
# In terminal:
# adk web --log_level DEBUG

# 3. Create custom plugin for specific debugging
class DebugPlugin(BasePlugin):
    async def before_tool_callback(self, *, agent, callback_context):
        print(f"🔧 Calling: {callback_context.tool_name}")
        print(f"   Params: {callback_context.tool_params}")

    async def after_tool_callback(self, *, agent, callback_context):
        print(f"✅ Result: {callback_context.tool_result}")

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin(), DebugPlugin()]
)
```

## Example 7: MCP Integration for External Services

**User asks:** "How do I connect my agent to external services?"

**Response:**
```bash
# Read Day 2b for MCP integration
Read: day2_agent_architecture/day-2b-agent-tools-best-practices.ipynb
```

**MCP example:**
```python
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

# Connect to MCP server
mcp_server = McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=["-y", "@modelcontextprotocol/server-everything"],
            tool_filter=["specific_tool"]
        ),
        timeout=30
    )
)

# Add to agent
agent = LlmAgent(
    tools=[mcp_server],
    ...
)
```

**When to use MCP:**
- External APIs (Google Maps, GitHub)
- File systems
- Databases
- Third-party services

**When to use custom tools:**
- Security-critical operations (PII filtering)
- Business logic requiring unit tests
- Deterministic compliance requirements

## Example 8: Multi-Agent Orchestration

**User asks:** "How do I coordinate multiple agents?"

**Response:**
```bash
# Read Day 1b for multi-agent patterns
Read: day1_intro_to_agents/day-1b-agent-architectures-28a9ac.ipynb
```

**Sequential pattern (VaaS uses this):**
```python
# Agent 1: Intake
intake_agent = LlmAgent(
    name="intake_agent",
    tools=[validate_document],
    output_key="validated_doc"
)

# Agent 2: Processing (uses Agent 1's output)
processing_agent = LlmAgent(
    name="processing_agent",
    tools=[ocr_tool, security_filter],
    sub_agents=[intake_agent],  # Can delegate back
    output_key="processed_doc"
)

# Run processing agent (automatically calls intake if needed)
result = await runner.run_async(
    query="Process document.txt"
)
```

**Parallel pattern:**
```python
# Multiple agents work simultaneously
parallel_agent = LlmAgent(
    sub_agents=[
        search_agent,
        analysis_agent,
        summary_agent
    ],
    instruction="Use all sub-agents in parallel for comprehensive results"
)
```

## Example 9: Competition Submission Guidance

**User asks:** "How should I structure my competition submission?"

**Response:**
```bash
# Read the final competition description
cat competition_description_final.md
```

**Key sections to include:**
1. **Problem Statement** - Innovation-Adoption Gap
2. **Why Agents?** - Learning journey (Days 1-5)
3. **What We Created** - Cross-framework architecture
4. **Demo** - Spanish birth certificate example
5. **The Build** - Technical decisions with rationale
6. **Business Innovation** - VaaS economics table
7. **Vision** - Enabling thousands of developers

**Winning elements:**
- ✅ Shows systemic understanding (not just personal problem)
- ✅ Emphasizes Day 5 A2A breakthrough
- ✅ Highlights cross-framework achievement
- ✅ Business model innovation (capability provider)
- ✅ Clear impact (five barriers removed)

## Example 10: Understanding Context Compaction

**User asks:** "My agent's context is too long, what do I do?"

**Response:**
```bash
# Read Day 3a for context management
Read: day3_tools_and_memory/day-3a-agent-sessions.ipynb
```

**Context compaction pattern:**
```python
from google.adk.apps.app import App, EventsCompactionConfig

# Wrap agent in App with compaction
app = App(
    name="my_app",
    root_agent=my_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compact every 3 turns
        overlap_size=1          # Keep 1 turn for context
    )
)

# Use with runner
runner = Runner(
    app=app,
    session_service=session_service
)

# Context automatically summarized after 3 turns
```

**When to use:**
- Long conversations (>10 turns)
- Government document processing (large histories)
- Cost optimization (fewer tokens)
- Performance improvement (faster processing)

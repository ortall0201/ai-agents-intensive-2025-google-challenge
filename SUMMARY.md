# Google ADK (Agent Development Kit) - Complete Guide

## 📚 Course Overview

This is a comprehensive summary of the Kaggle 5-day AI Agents Intensive course covering Google's Agent Development Kit (ADK). This guide synthesizes key concepts, patterns, and best practices for building production-ready AI agent systems.

---

## 🏗️ Core Architecture

```
User Request
    ↓
Runner (Orchestrator)
    ↓
Agent (Brain)
    ↓
├── Model (LLM)
├── Tools (Functions/APIs/Other Agents)
└── Context (Instructions + State)
    ↓
Response
    ↓
Storage
├── SessionService (Short-term: Events, State)
└── MemoryService (Long-term: Knowledge)
```

### **Key Distinction: Agent vs LLM**

| LLM | Agent |
|-----|-------|
| Prompt → Text | Prompt → Thought → Action → Observation → Answer |
| Stateless | Can maintain state |
| No tools | Can use tools |
| Single interaction | Multi-turn conversation |

---

# 📘 Day 1: Foundation - Agents & Orchestration

## Part A: Building Your First Agent

### **The Basics**

```python
from google.adk.agents import Agent
from google.adk.runners import InMemoryRunner
from google.adk.tools import google_search

# Define the agent
agent = Agent(
    name="helpful_assistant",
    model="gemini-2.5-flash-lite",
    description="A simple agent that can answer questions.",
    instruction="You are a helpful assistant. Use Google Search for current info.",
    tools=[google_search],
)

# Create runner
runner = InMemoryRunner(agent=agent)

# Quick prototyping
response = await runner.run_debug("What is ADK?")
```

### **Core Components**

1. **name**: Identifier for the agent
2. **model**: LLM to use (e.g., `gemini-2.5-flash-lite`)
3. **description**: What the agent does
4. **instruction**: System prompt guiding behavior
5. **tools**: List of functions/tools available to agent

### **Runner**

- Central orchestrator managing conversation flow
- Handles session management automatically
- `run_debug()`: Quick prototyping without manual session setup
- `run_async()`: Full control with session IDs

---

## Part B: Multi-Agent Systems & Orchestration

### **Why Multi-Agent?**

**Problem**: Monolithic "do-it-all" agents are:
- Hard to debug (which part failed?)
- Difficult to maintain (long, complex prompts)
- Unreliable (trying to do too much)

**Solution**: Team of specialists
- Each agent has ONE clear job
- Easier to build, test, and debug
- More reliable when working together

---

## **The 4 Orchestration Patterns**

### **1. LLM Orchestrator (Dynamic)**

**When**: Need flexible, adaptive decision-making

```python
research_agent = Agent(name="ResearchAgent", tools=[google_search], ...)
summarizer_agent = Agent(name="SummarizerAgent", ...)

# Root agent decides what to call and when
root = Agent(
    name="Coordinator",
    instruction="First call ResearchAgent, then SummarizerAgent",
    tools=[AgentTool(research_agent), AgentTool(summarizer_agent)]
)
```

**Pros**: Flexible, LLM makes intelligent routing decisions
**Cons**: Unpredictable, LLM might skip steps or run wrong order

---

### **2. SequentialAgent (Fixed Pipeline)**

**When**: Order matters, need deterministic execution

```python
from google.adk.agents import SequentialAgent

outline_agent = Agent(name="OutlineAgent", output_key="outline", ...)
writer_agent = Agent(name="WriterAgent",
                     instruction="Use outline: {outline}",
                     output_key="draft", ...)
editor_agent = Agent(name="EditorAgent",
                     instruction="Edit draft: {draft}",
                     output_key="final", ...)

# Always runs in order: outline → writer → editor
pipeline = SequentialAgent(
    name="BlogPipeline",
    sub_agents=[outline_agent, writer_agent, editor_agent]
)
```

**Pros**: Guaranteed order, predictable, easy to debug
**Cons**: Inflexible, sequential (no parallelization)

**State Passing**: `output_key` → `{placeholder}` in next agent's instruction

---

### **3. ParallelAgent (Concurrent Execution)**

**When**: Independent tasks, need speed

```python
from google.adk.agents import ParallelAgent

tech_researcher = Agent(name="TechResearcher", output_key="tech", ...)
health_researcher = Agent(name="HealthResearcher", output_key="health", ...)
finance_researcher = Agent(name="FinanceResearcher", output_key="finance", ...)

# All run simultaneously
parallel_team = ParallelAgent(
    name="ResearchTeam",
    sub_agents=[tech_researcher, health_researcher, finance_researcher]
)

# Then aggregate results
aggregator = Agent(
    name="Aggregator",
    instruction="Combine: {tech}, {health}, {finance}"
)

# Combine patterns
root = SequentialAgent(
    sub_agents=[parallel_team, aggregator]  # Parallel first, then aggregate
)
```

**Pros**: Fast, efficient for independent tasks
**Cons**: Can't share data between parallel agents during execution

---

### **4. LoopAgent (Iterative Refinement)**

**When**: Need quality improvement through feedback cycles

```python
from google.adk.agents import LoopAgent

def exit_loop():
    """Signal to stop the loop."""
    return {"status": "approved"}

writer = Agent(name="Writer", output_key="story", ...)
critic = Agent(name="Critic",
               instruction="Review: {story}. Say 'APPROVED' or give feedback.",
               output_key="critique", ...)
refiner = Agent(name="Refiner",
                instruction="If critique is 'APPROVED', call exit_loop(). Else rewrite.",
                tools=[FunctionTool(exit_loop)],
                output_key="story", ...)  # Overwrites story

# Loops until exit_loop() is called or max_iterations reached
refinement_loop = LoopAgent(
    name="RefinementLoop",
    sub_agents=[critic, refiner],
    max_iterations=3
)

# Full pipeline
root = SequentialAgent(
    sub_agents=[writer, refinement_loop]  # Initial write, then refine
)
```

**Pros**: Quality through iteration, self-improving
**Cons**: Can be slow, needs exit condition

---

## **Decision Tree: Which Pattern?**

```
What kind of workflow?
│
├─ Fixed Pipeline (A→B→C) → SequentialAgent
├─ Concurrent Tasks (A, B, C all at once) → ParallelAgent
├─ Iterative Refinement (A⇆B until good) → LoopAgent
└─ Dynamic Decisions (LLM decides) → LLM Orchestrator
```

### **Quick Reference**

| Pattern | When | Example | Key Feature |
|---------|------|---------|-------------|
| **LLM Orchestrator** | Dynamic orchestration | Research + Summarize | LLM decides what to call |
| **SequentialAgent** | Order matters | Outline → Write → Edit | Deterministic order |
| **ParallelAgent** | Independent tasks | Multi-topic research | Concurrent execution |
| **LoopAgent** | Quality improvement | Writer + Critic | Repeated cycles |

---

# 🔧 Day 2: Tools - Custom Functions & External Integration

## Part A: Custom Tools & Agent Tools

### **Core Concept: Any Function → Tool**

```python
def get_exchange_rate(base_currency: str, target_currency: str) -> dict:
    """Looks up exchange rate between two currencies.

    Args:
        base_currency: ISO code (e.g., "USD")
        target_currency: ISO code (e.g., "EUR")

    Returns:
        Dictionary with status and rate.
        Success: {"status": "success", "rate": 0.93}
        Error: {"status": "error", "error_message": "..."}
    """
    # Implementation...
    return {"status": "success", "rate": 0.93}

# Use in agent
agent = Agent(
    name="currency_agent",
    instruction="Use get_exchange_rate() to convert currencies.",
    tools=[get_exchange_rate]  # Just add to list!
)
```

### **🎯 ADK Tool Best Practices (CRITICAL!)**

#### **1. Always Return Structured Dictionaries**

```python
# ✅ GOOD - Structured with status
{"status": "success", "data": result}
{"status": "error", "error_message": "reason"}

# ❌ BAD - Unstructured
return 0.93  # Is this rate or error code?
```

#### **2. Complete Function Signature**

```python
def my_tool(param: str, tool_context: ToolContext) -> dict:
    """Clear docstring - LLM READS THIS to decide when to use tool!

    Args:
        param: Description

    Returns:
        Dictionary with status and data
    """
    pass
```

**Must-haves**:
- ✅ Detailed docstring (LLM's instruction manual)
- ✅ Type hints (`str`, `dict`, `int`, etc.)
- ✅ Dictionary return values
- ✅ Error handling
- ✅ `tool_context: ToolContext` parameter (for LRO)

#### **3. Agent Instructions Reference Tools**

```python
instruction = """
1. Use `get_fee_for_payment_method()` to find fees
2. Use `get_exchange_rate()` to get rates
3. Check "status" field in responses for errors
4. Calculate final amount using both results
"""
```

---

### **Agent as Tool Pattern**

**Use Case**: Delegation (not handoff!)

```python
# Specialist agent
calculation_agent = LlmAgent(
    name="CalculationAgent",
    instruction="Generate Python code to calculate the result.",
    code_executor=BuiltInCodeExecutor()
)

# Root agent uses specialist
currency_agent = LlmAgent(
    name="CurrencyAgent",
    instruction="Use CalculationAgent to compute final amounts.",
    tools=[
        get_fee_for_payment_method,
        get_exchange_rate,
        AgentTool(agent=calculation_agent)  # Agent as tool!
    ]
)
```

**Key Difference**:

| Agent Tool | Sub-Agent |
|------------|-----------|
| Agent B called as function | Agent A transfers control to B |
| Result returns to Agent A | Agent B handles all future messages |
| Agent A continues working | Agent A is out of the loop |
| **Use**: Delegation | **Use**: Handoff |

---

### **Built-in Code Executor**

**Problem**: LLMs are bad at math
**Solution**: Generate code, let executor run it

```python
from google.adk.code_executors import BuiltInCodeExecutor

agent = LlmAgent(
    name="calculator",
    instruction="Generate Python code to solve math problems.",
    code_executor=BuiltInCodeExecutor()  # Sandboxed execution
)
```

**Flow**: Agent generates code → Executor runs → Agent uses result

---

### **Tool Type Categories**

**Custom Tools:**
- **Function Tools**: Simple Python functions
- **Long Running Function Tools**: Async, human-in-loop
- **Agent Tools**: Agents as tools
- **MCP Tools**: External service integration
- **OpenAPI Tools**: Auto-generated from API specs

**Built-in Tools:**
- **Gemini Tools**: `google_search`, `BuiltInCodeExecutor`
- **Google Cloud Tools**: `BigQueryToolset`, `SpannerToolset`
- **Third-party Tools**: Hugging Face, GitHub, etc.

---

## Part B: MCP & Long-Running Operations

### **Model Context Protocol (MCP)**

**What**: Standardized way to connect agents to external services

**Architecture**:
```
Your Agent (MCP Client)
    ↓
MCP Protocol (Standard Interface)
    ↓
┌────────┬─────────┬──────────┬─────────┐
│ GitHub │ Kaggle  │  Slack   │  Maps   │
│  MCP   │   MCP   │   MCP    │   MCP   │
└────────┴─────────┴──────────┴─────────┘
```

**Usage**:

```python
from google.adk.tools.mcp_tool.mcp_toolset import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters

# Connect to MCP server
mcp_toolset = McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=["-y", "@modelcontextprotocol/server-everything"],
            tool_filter=["getTinyImage"]
        ),
        timeout=30
    )
)

# Use in agent
agent = Agent(
    name="image_agent",
    instruction="Use MCP tools to generate images.",
    tools=[mcp_toolset]
)
```

**Popular MCP Servers**:
- **Kaggle MCP**: Datasets, notebooks
- **GitHub MCP**: PRs, issues
- **Everything MCP**: Testing server

**Benefits**: No custom integration code, standardized interface

---

### **Long-Running Operations (LRO)**

**Problem**: Some operations need to pause for human approval

**Use Cases**:
- 💰 Financial transactions
- 🗑️ Bulk operations (delete 1000 records)
- 📋 Compliance checkpoints
- 💸 High-cost actions
- ⚠️ Irreversible operations

---

### **LRO: The Three Components**

#### **1. Tool with ToolContext**

```python
def place_shipping_order(
    num_containers: int,
    destination: str,
    tool_context: ToolContext  # ADK provides this!
) -> dict:
    """Places shipping order. Requires approval if > 5 containers."""

    # SCENARIO 1: Small orders auto-approve
    if num_containers <= 5:
        return {"status": "approved", "order_id": "..."}

    # SCENARIO 2: First call - PAUSE for approval
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(
            hint=f"Approve {num_containers} containers?",
            payload={"num_containers": num_containers}
        )
        return {"status": "pending"}

    # SCENARIO 3: Second call - RESUME with decision
    if tool_context.tool_confirmation.confirmed:
        return {"status": "approved", "order_id": "..."}
    else:
        return {"status": "rejected"}
```

**Key States**:
- **Call 1 (Pause)**: `tool_context.tool_confirmation` is `None` → Request approval → Return pending
- **Call 2 (Resume)**: `tool_context.tool_confirmation` exists → Check `confirmed` → Return final result

---

#### **2. Resumable App**

```python
from google.adk.apps.app import App, ResumabilityConfig

# Wrap agent in App with resumability
app = App(
    name="shipping_coordinator",
    root_agent=shipping_agent,
    resumability_config=ResumabilityConfig(is_resumable=True)
)
```

**What gets saved**:
- All conversation messages
- Which tool was called
- Tool parameters
- Exact pause point

**When resuming**: Loads saved state, continues exactly where it left off

---

#### **3. Event Detection Workflow**

```python
# STEP 1: Initial call
async for event in runner.run_async(...):
    events.append(event)

# STEP 2: Detect pause
approval_info = check_for_approval(events)
# Returns: {"approval_id": "...", "invocation_id": "abc123"}

# STEP 3: Resume with SAME invocation_id
if approval_info:
    async for event in runner.run_async(
        ...,
        new_message=approval_response,
        invocation_id=approval_info["invocation_id"]  # ← CRITICAL!
    ):
        # Process resumed execution
```

**🚨 Critical**: `invocation_id` is SACRED
- Without it: ADK starts NEW execution instead of resuming
- With it: Continues paused execution

---

### **Helper Functions for LRO**

```python
def check_for_approval(events):
    """Detect if agent paused for approval."""
    for event in events:
        if event.content and event.content.parts:
            for part in event.content.parts:
                if part.function_call and \
                   part.function_call.name == "adk_request_confirmation":
                    return {
                        "approval_id": part.function_call.id,
                        "invocation_id": event.invocation_id
                    }
    return None

def create_approval_response(approval_info, approved: bool):
    """Format human decision for ADK."""
    confirmation_response = types.FunctionResponse(
        id=approval_info["approval_id"],
        name="adk_request_confirmation",
        response={"confirmed": approved}
    )
    return types.Content(
        role="user",
        parts=[types.Part(function_response=confirmation_response)]
    )
```

---

### **Complete LRO Workflow**

```python
async def run_workflow(query: str, auto_approve: bool = True):
    # Generate session ID
    session_id = f"order_{uuid.uuid4().hex[:8]}"

    # Create session
    await session_service.create_session(...)

    # STEP 1: Initial request
    events = []
    async for event in runner.run_async(...):
        events.append(event)

    # STEP 2: Check for pause
    approval_info = check_for_approval(events)

    # STEP 3: Handle approval
    if approval_info:
        print("⏸️ Pausing for approval...")

        # Get human decision (simulated here)
        decision = auto_approve

        # Resume with decision
        async for event in runner.run_async(
            ...,
            new_message=create_approval_response(approval_info, decision),
            invocation_id=approval_info["invocation_id"]
        ):
            # Display final response
            ...
    else:
        # No approval needed
        print("✅ Completed immediately")
```

---

# 💾 Day 3: Memory - Sessions & Long-term Knowledge

## Part A: Sessions (Short-term Memory)

### **The Problem: LLMs are Stateless**

Without context management:
- LLM forgets everything after each call
- No conversation continuity
- Can't reference previous interactions

**Solution**: Sessions

---

### **What is a Session?**

**Session** = Container for a single, continuous conversation

**Components**:
1. **Events**: Building blocks of conversation
   - User messages
   - Agent responses
   - Tool calls
   - Tool outputs

2. **State**: Agent's scratchpad `{key: value}` storage
   - Shared across all sub-agents and tools
   - Available via `tool_context.state`

```
Session
├── Events []
│   ├── User: "Hi, I'm Sam"
│   ├── Agent: "Hi Sam!"
│   ├── Tool Call: search("weather")
│   └── Tool Result: "75°F"
└── State {}
    ├── user:name → "Sam"
    └── user:location → "NYC"
```

---

### **Session Management Architecture**

```
Application
    ↓
Runner (Orchestrator)
    ↓
SessionService (Storage Layer)
    ↓
├── Session 1 (User A)
├── Session 2 (User A)
└── Session 3 (User B)
```

**Key Concepts**:
- **SessionService**: Manages creation, storage, retrieval
- **Runner**: Automatically maintains conversation history
- **Session Isolation**: Sessions don't share data with each other

---

### **InMemorySessionService (Development)**

```python
from google.adk.sessions import InMemorySessionService

# Create session service
session_service = InMemorySessionService()

# Create runner
runner = Runner(
    agent=agent,
    app_name="my_app",
    session_service=session_service
)

# Create/get session
session = await session_service.create_session(
    app_name="my_app",
    user_id="user123",
    session_id="session_abc"
)

# Run conversation
async for event in runner.run_async(
    user_id="user123",
    session_id="session_abc",
    new_message=query
):
    # Process response
```

**Limitation**: In-memory only, lost on restart

---

### **DatabaseSessionService (Production)**

```python
from google.adk.sessions import DatabaseSessionService

# Create persistent session service
db_url = "sqlite:///my_agent_data.db"
session_service = DatabaseSessionService(db_url=db_url)

# Same API as InMemorySessionService!
runner = Runner(
    agent=agent,
    app_name="my_app",
    session_service=session_service
)
```

**Benefits**:
- ✅ Survives restarts
- ✅ Can query historical data
- ✅ Production-ready

**Storage Format** (SQLite example):
```
events table:
- app_name
- session_id
- author (user/agent)
- content (JSON)
- timestamp
```

---

### **Session Service Comparison**

| Service | Persistence | Use Case |
|---------|-------------|----------|
| **InMemorySessionService** | ❌ Lost on restart | Development, testing |
| **DatabaseSessionService** | ✅ Survives restarts | Self-managed apps |
| **Agent Engine Sessions** | ✅ Fully managed (GCP) | Enterprise scale |

---

### **Context Compaction**

**Problem**: Long conversations = huge context = slow + expensive

**Solution**: Automatically summarize old conversation turns

```python
from google.adk.apps.app import App, EventsCompactionConfig

# Create app with compaction
app = App(
    name="research_app",
    root_agent=agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compact every 3 turns
        overlap_size=1          # Keep 1 previous turn
    )
)

runner = Runner(app=app, session_service=session_service)
```

**What happens**:
1. After 3rd turn, trigger compaction
2. LLM summarizes first 2 turns
3. Replace 2 full turns with 1 summary event
4. Keep last 1 turn for overlap
5. Future turns see: [Summary] + [Recent turns]

**Benefits**: Faster, cheaper, maintains context

---

### **Session State Management**

**Use Case**: Share key information across tools/agents

```python
# Tool that WRITES to state
def save_userinfo(
    tool_context: ToolContext,
    user_name: str,
    country: str
) -> dict:
    """Save user info to session state."""
    tool_context.state["user:name"] = user_name
    tool_context.state["user:country"] = country
    return {"status": "success"}

# Tool that READS from state
def retrieve_userinfo(tool_context: ToolContext) -> dict:
    """Retrieve user info from session state."""
    name = tool_context.state.get("user:name", "Not found")
    country = tool_context.state.get("user:country", "Not found")
    return {"status": "success", "user_name": name, "country": country}

# Agent with state tools
agent = LlmAgent(
    name="chatbot",
    instruction="Use save_userinfo when user shares info. Use retrieve_userinfo to recall.",
    tools=[save_userinfo, retrieve_userinfo]
)
```

**Best Practices**:
- Use prefixes: `user:`, `app:`, `temp:`
- Keys are scoped to session (not shared across sessions)
- State persists throughout session lifetime

---

## Part B: Memory (Long-term Knowledge)

### **Session vs Memory**

| Session | Memory |
|---------|--------|
| Single conversation | All conversations |
| Short-term (this chat) | Long-term (all chats) |
| Raw events | Extracted facts |
| Chronological | Searchable |
| Like RAM | Like database |

**Example**:
- **Session**: Remembers what you said 10 minutes ago
- **Memory**: Remembers your preferences from last week

---

### **Why Memory?**

| Capability | What It Means | Example |
|------------|---------------|---------|
| **Cross-Conversation Recall** | Access info from any past chat | "What preferences across all conversations?" |
| **Intelligent Extraction** | LLM consolidates key facts | Stores "allergic to peanuts" not 50 messages |
| **Semantic Search** | Meaning-based retrieval | "preferred hue" matches "favorite color" |
| **Persistent Storage** | Survives restarts | Knowledge grows over time |

---

### **Memory Workflow: 3 Steps**

```
1. Initialize → Create MemoryService
2. Ingest    → Transfer sessions to memory
3. Retrieve  → Search stored memories
```

---

### **1. Initialize Memory**

```python
from google.adk.memory import InMemoryMemoryService

# Create memory service
memory_service = InMemoryMemoryService()

# Create agent
agent = LlmAgent(
    name="chatbot",
    instruction="Answer questions."
)

# Create runner with BOTH services
runner = Runner(
    agent=agent,
    app_name="my_app",
    session_service=session_service,
    memory_service=memory_service  # Memory now available!
)
```

**⚠️ Important**: Adding `memory_service` makes memory *available*, but doesn't automatically use it. You must explicitly:
1. Ingest data with `add_session_to_memory()`
2. Enable retrieval with memory tools

---

### **2. Ingest Session Data**

```python
# Have conversation
await run_session(runner, "My favorite color is blue.", "session-01")

# Get session
session = await session_service.get_session(
    app_name="my_app",
    user_id="user123",
    session_id="session-01"
)

# Transfer to memory
await memory_service.add_session_to_memory(session)
```

**What happens**:
- Session events → Memory storage
- Managed services (like Vertex AI) extract key facts
- InMemoryMemoryService stores raw events

---

### **3. Retrieve Memories**

**Two patterns**:

#### **A. load_memory (Reactive)**

Agent decides when to search memory.

```python
from google.adk.tools import load_memory

agent = LlmAgent(
    name="chatbot",
    instruction="Use load_memory tool if you need past conversation info.",
    tools=[load_memory]  # Agent decides when to search
)
```

**Pros**: Efficient (only searches when needed)
**Cons**: Agent might forget to search

---

#### **B. preload_memory (Proactive)**

Automatically searches before every turn.

```python
from google.adk.tools import preload_memory

agent = LlmAgent(
    name="chatbot",
    instruction="Answer questions.",
    tools=[preload_memory]  # Always loads memory
)
```

**Pros**: Guaranteed context
**Cons**: Less efficient (searches even when not needed)

---

### **Manual Memory Search**

```python
# Direct search (debugging, analytics, etc.)
search_response = await memory_service.search_memory(
    app_name="my_app",
    user_id="user123",
    query="What is the user's favorite color?"
)

print(f"Found {len(search_response.memories)} memories")
for memory in search_response.memories:
    print(memory.content.parts[0].text)
```

---

### **Automating Memory with Callbacks**

**Problem**: Manually calling `add_session_to_memory()` is tedious

**Solution**: Use callbacks to auto-save after each turn

```python
# Define callback
async def auto_save_to_memory(callback_context):
    """Automatically save session to memory after each turn."""
    await callback_context._invocation_context.memory_service.add_session_to_memory(
        callback_context._invocation_context.session
    )

# Create agent with callback
agent = LlmAgent(
    name="chatbot",
    instruction="Answer questions.",
    tools=[preload_memory],
    after_agent_callback=auto_save_to_memory  # Auto-saves!
)
```

**Result**: Zero manual memory calls, fully automated

---

### **Callbacks Overview**

**What**: Functions that run at specific execution points

**Types**:
- `before_agent_callback`: Before agent starts
- `after_agent_callback`: After agent completes (← use for memory)
- `before_tool_callback` / `after_tool_callback`: Around tools
- `before_model_callback` / `after_model_callback`: Around LLM
- `on_model_error_callback`: On errors

**Common Uses**:
- Logging/observability
- Auto-save to memory
- Validation
- Performance monitoring

---

### **Memory Consolidation**

**Problem**: Storing raw events doesn't scale

```
Before (Raw Storage):
User: "My favorite color is blue. I also like purple. Actually, blue is best."
Agent: "Great!"
User: "Thanks!"
Agent: "Welcome!"

→ Stores ALL 4 messages (redundant)
```

**After (Consolidation)**:
```
Extracted Memory: "User's favorite color: blue"

→ Stores 1 concise fact
```

**How it works** (Managed Services):
```
1. Raw Session Events
   ↓
2. LLM analyzes conversation
   ↓
3. Extracts key facts
   ↓
4. Stores concise memories
   ↓
5. Merges with existing (deduplication)
```

**Benefits**:
- Less storage
- Faster retrieval
- More accurate answers

---

### **Memory Service Implementations**

#### **InMemoryMemoryService (This Course)**

- Stores raw conversation events
- Keyword-based search (simple word matching)
- In-memory storage (resets on restart)
- Ideal for learning

#### **VertexAiMemoryBankService (Production)**

- LLM-powered fact extraction
- Semantic search (meaning-based, not keywords)
- Persistent cloud storage
- External knowledge integration

**💡 API Consistency**: Both use same methods, workflow is identical!

---

### **When to Save to Memory?**

| Timing | Implementation | Best For |
|--------|----------------|----------|
| **After every turn** | `after_agent_callback` | Real-time updates |
| **End of conversation** | Manual call when done | Batch processing |
| **Periodic intervals** | Background job | Long conversations |

---

### **🔍 Deep Dive: load_memory vs preload_memory**

**The Critical Difference: Tool Call Visibility**

When you use `load_memory`:
```python
# Agent with load_memory
agent = LlmAgent(
    instruction="Use load_memory if you need past info.",
    tools=[load_memory]
)

# Question: "What is my favorite color?"
# Output: WARNING: non-text parts in response: ['function_call']
# Result: You SEE the agent calling the tool
```

When you use `preload_memory`:
```python
# Agent with preload_memory
agent = LlmAgent(
    instruction="Answer questions.",
    tools=[preload_memory]
)

# Question: "What is my favorite color?"
# Output: Clean response, no warnings
# Result: Tool call is INVISIBLE - happens before agent thinks
```

**Testing Both Patterns:**

```python
# Helper to see tool calls
async def run_session_verbose(runner, query, session_id):
    tool_calls_detected = []

    async for event in runner.run_async(...):
        if event.content and event.content.parts:
            for part in event.content.parts:
                if part.function_call:
                    tool_calls_detected.append(part.function_call.name)
                    print(f"🔧 Tool Called: {part.function_call.name}")

    if not tool_calls_detected:
        print(f"⭕ No tools called")

    return len(tool_calls_detected)
```

**Test Results:**

| Pattern | Query | Tool Calls Visible? | Memory Searched? |
|---------|-------|---------------------|------------------|
| `load_memory` | "What is my favorite color?" | ✅ YES (`function_call` warning) | ✅ YES |
| `load_memory` | "Tell me a joke" | ⭕ NO (no warning) | ❌ NO |
| `preload_memory` | "What is my favorite color?" | ⭕ NO (silent) | ✅ YES |
| `preload_memory` | "Tell me a joke" | ⭕ NO (silent) | ✅ YES (wasteful!) |

**The Wasteful Pattern:**

With `preload_memory`, EVERY query searches memory, even when unnecessary:

```python
# Even for this joke question, memory WAS searched!
await run_session(preload_runner, "Tell me a joke", "joke-session")

# Proof: Manual search shows what was loaded
joke_memories = await memory_service.search_memory(
    app_name=APP_NAME,
    user_id=USER_ID,
    query="Tell me a joke"
)
print(f"Memories loaded: {len(joke_memories.memories)}")
# Output: 4 memories (birthday, color, etc.) - all unnecessary!
```

**When to Use Each:**

| Use Case | Pattern | Why |
|----------|---------|-----|
| Personal assistant chatbot | `load_memory` | Most questions don't need full history - save tokens |
| Medical diagnosis agent | `preload_memory` | MUST always know patient allergies, conditions |
| Banking support | `preload_memory` | Must always know account restrictions |
| General Q&A bot | `load_memory` | Mixed queries, let agent decide |
| Compliance-critical apps | `preload_memory` | Reliability > efficiency |

---

### **🔬 Keyword Matching in InMemoryMemoryService**

**How It Works:**

`InMemoryMemoryService` uses **literal word matching** - searches for exact words in stored text.

**Testing Keyword Matching:**

```python
# Memory contains:
# 1. "My favorite color is blue-green. Can you write a Haiku about it?"
# 2. "Ocean meets the sky, A calming, vibrant blend, Nature's peaceful hue."
# 3. "My birthday is on March 15th."

# Test different queries
queries = [
    "what color does the user like",  # Contains "color"
    "haiku",                          # Exact word "haiku"
    "age",                            # NOT in memory
    "preferred hue"                   # Contains "hue" (from poem!)
]

for query in queries:
    result = await memory_service.search_memory(
        app_name=APP_NAME,
        user_id=USER_ID,
        query=query
    )
    print(f"Query '{query}': {len(result.memories)} matches")
```

**Results:**

| Query | Matches? | Why? |
|-------|----------|------|
| "what color does the user like" | ✅ YES (2) | Word "**color**" in stored text |
| "haiku" | ✅ YES (2) | Exact word "**haiku**" appears |
| "age" | ❌ NO (0) | Word "age" never mentioned (only "birthday") |
| "preferred hue" | ✅ YES (2) | Word "**hue**" in poem! ("Nature's peaceful **hue**") |

**⚠️ Important Discovery:**

The "preferred hue" query DOES match - but not because of semantic understanding! It matches because the agent's haiku response contains "hue". This shows:

1. **Keyword matching is literal** - it finds "hue" in the poem
2. **Not semantic** - it doesn't understand "hue" = "color"
3. **Context doesn't matter** - finds word anywhere, even in poems

**Why This Matters:**

- ✅ **Prevents hallucination**: Only retrieves what actually exists
- ❌ **Misses synonyms**: "preferred" ≠ "favorite", "hue" ≠ "color" (usually)
- ❌ **No concept understanding**: "age" doesn't match "birthday"

**Production Solution:**

```python
# InMemoryMemoryService (Learning)
# Query: "preferred hue"
# Result: Matches only if word "hue" appears in text

# VertexAiMemoryBankService (Production - Day 5)
# Query: "preferred hue"
# Result: Matches "favorite color" via semantic embeddings
# Understands meaning, not just words!
```

---

### **Manual Memory Verification**

**Debugging What's Actually Stored:**

```python
# Get ALL memories (empty query returns everything)
all_memories = await memory_service.search_memory(
    app_name=APP_NAME,
    user_id=USER_ID,
    query=""  # Empty = return all
)

print(f"Total memories: {len(all_memories.memories)}")

for i, mem in enumerate(all_memories.memories, 1):
    if mem.content and mem.content.parts:
        text = mem.content.parts[0].text[:100]
        print(f"{i}. [{mem.author}]: {text}...")

# Output shows EXACTLY what's searchable
# InMemoryMemoryService searches these exact texts
# If your query words don't appear here, you get no results!
```

**Use Cases:**
- Debugging why searches fail
- Verifying memory ingestion
- Building analytics dashboards
- Understanding what agents can access

---

# 🎯 Critical Patterns & Best Practices

## Pattern 1: Tool Definition

```python
def my_tool(param: str, tool_context: ToolContext) -> dict:
    """Clear docstring explaining WHEN to use this tool.

    LLMs read docstrings to decide tool usage!

    Args:
        param: Description of parameter
        tool_context: ADK provides this automatically

    Returns:
        Always return structured dict:
        - Success: {"status": "success", "data": ...}
        - Error: {"status": "error", "error_message": ...}
    """
    try:
        result = do_something(param)
        return {"status": "success", "data": result}
    except Exception as e:
        return {"status": "error", "error_message": str(e)}
```

**Checklist**:
- ✅ Detailed docstring
- ✅ Type hints
- ✅ Structured return (`{"status": ...}`)
- ✅ Error handling
- ✅ `tool_context` parameter (if using LRO)

---

## Pattern 2: LRO State Machine

```python
def long_running_tool(param: str, tool_context: ToolContext) -> dict:
    """Tool that needs approval."""

    # Auto-approve simple cases
    if is_simple(param):
        return {"status": "approved", ...}

    # CALL 1: Pause for approval
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(
            hint="Approve action?",
            payload={"param": param}
        )
        return {"status": "pending", "message": "Awaiting approval"}

    # CALL 2: Resume with decision
    if tool_context.tool_confirmation.confirmed:
        return {"status": "approved", ...}
    else:
        return {"status": "rejected", ...}
```

---

## Pattern 3: invocation_id Handling

```python
# Save ID from pause event
def check_for_approval(events):
    for event in events:
        if has_approval_request(event):
            return {
                "approval_id": event.function_call.id,
                "invocation_id": event.invocation_id  # ← Save this!
            }
    return None

# Resume with SAME ID
approval_info = check_for_approval(events)
if approval_info:
    await runner.run_async(
        ...,
        invocation_id=approval_info["invocation_id"]  # ← Use same ID!
    )
```

**🚨 CRITICAL**: Without matching `invocation_id`, ADK starts NEW execution!

---

## Pattern 4: State Passing

```python
# Agent 1 produces data
agent1 = Agent(
    name="ResearchAgent",
    instruction="Research the topic.",
    output_key="research_findings"  # ← Stores in state
)

# Agent 2 consumes data
agent2 = Agent(
    name="SummarizerAgent",
    instruction="Summarize this research: {research_findings}",  # ← Injects from state
    output_key="summary"
)

# Chain them
pipeline = SequentialAgent(sub_agents=[agent1, agent2])
```

---

## Pattern 5: Memory Workflow

```python
# 1. Initialize
memory_service = InMemoryMemoryService()
runner = Runner(
    agent=agent,
    session_service=session_service,
    memory_service=memory_service
)

# 2. Ingest (manual)
session = await session_service.get_session(...)
await memory_service.add_session_to_memory(session)

# OR: Ingest (automatic)
async def auto_save(callback_context):
    await callback_context._invocation_context.memory_service \
        .add_session_to_memory(callback_context._invocation_context.session)

agent = LlmAgent(..., after_agent_callback=auto_save)

# 3. Retrieve (agent tools)
agent = LlmAgent(..., tools=[load_memory])  # or preload_memory
```

---

## Pattern 6: Error Handling

```python
# In tools
def my_tool(param: str) -> dict:
    try:
        result = risky_operation(param)
        return {"status": "success", "data": result}
    except ValueError as e:
        return {"status": "error", "error_message": f"Invalid input: {e}"}
    except Exception as e:
        return {"status": "error", "error_message": f"Unexpected error: {e}"}

# In agent instructions
instruction = """
Use my_tool() to process data.
ALWAYS check the "status" field in the response:
- If status is "error", explain the error_message to the user
- If status is "success", use the data field
"""
```

---

# 🚨 Common Mistakes & How to Avoid Them

## Mistake 1: Forgetting tool_context

```python
# ❌ BAD - Won't work for LRO
def approve_order(num_items: int) -> dict:
    return {"status": "pending"}

# ✅ GOOD - Has tool_context
def approve_order(num_items: int, tool_context: ToolContext) -> dict:
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(...)
        return {"status": "pending"}
```

---

## Mistake 2: Unstructured Returns

```python
# ❌ BAD - Agent confused
def get_rate():
    return 0.93  # Is this rate or error code?

# ✅ GOOD - Clear structure
def get_rate():
    return {"status": "success", "rate": 0.93}
```

---

## Mistake 3: Losing invocation_id

```python
# ❌ BAD - Starts new execution
approval_info = check_for_approval(events)
await runner.run_async(...)  # No invocation_id!

# ✅ GOOD - Resumes paused execution
await runner.run_async(
    ...,
    invocation_id=approval_info["invocation_id"]
)
```

---

## Mistake 4: Missing Docstrings

```python
# ❌ BAD - LLM doesn't know when to use
def search_database(query):
    return results

# ✅ GOOD - Clear purpose
def search_database(query: str) -> dict:
    """Search the product database for items matching the query.

    Use this when the user asks about product availability, prices, or specifications.
    """
    return {"status": "success", "results": results}
```

---

## Mistake 5: Using Sub-Agents for Delegation

```python
# ❌ BAD - Sub-agent takes full control
# Agent A transfers to Agent B, A is out of loop

# ✅ GOOD - Agent Tool returns control
specialist = Agent(name="Specialist", ...)
root = Agent(
    name="Root",
    tools=[AgentTool(specialist)]  # Specialist returns to Root
)
```

---

## Mistake 6: Not Checking "status" Field

```python
# ❌ BAD - Doesn't handle errors
instruction = "Use get_data() and process the result."

# ✅ GOOD - Explicit error handling
instruction = """
Use get_data() to fetch information.
Check the "status" field:
- If "error": explain the error_message to user
- If "success": process the data field
"""
```

---

## Mistake 7: Forgetting to Ingest Sessions to Memory

```python
# ❌ BAD - Memory service initialized but never populated
memory_service = InMemoryMemoryService()
runner = Runner(
    agent=agent,
    session_service=session_service,
    memory_service=memory_service  # Available but empty!
)

# Agent has load_memory tool but finds nothing!
await run_session(runner, "What's my favorite color?")
# Result: "I don't have that information"

# ✅ GOOD - Actually ingest data
await run_session(runner, "My favorite color is blue", "session-01")

# Transfer to memory!
session = await session_service.get_session(...)
await memory_service.add_session_to_memory(session)

# Now agent can retrieve
await run_session(runner, "What's my favorite color?")
# Result: "Your favorite color is blue"
```

---

## Mistake 8: Using preload_memory Everywhere

```python
# ❌ BAD - Wastes tokens on every single turn
agent = LlmAgent(
    name="general_chatbot",
    instruction="Answer questions about anything.",
    tools=[preload_memory]  # Searches memory even for "What's 2+2?"
)

# Every query loads all memories, even math questions!
# Cost: HIGH, Efficiency: LOW

# ✅ GOOD - Use load_memory for mixed conversations
agent = LlmAgent(
    name="general_chatbot",
    instruction="Use load_memory when you need past conversation info.",
    tools=[load_memory]  # Agent decides when to search
)

# Only searches when needed
# Cost: LOW, Efficiency: HIGH

# ✅ ALSO GOOD - Use preload_memory only when critical
medical_agent = LlmAgent(
    name="medical_assistant",
    instruction="Help with medical questions.",
    tools=[preload_memory]  # MUST always know allergies!
)
```

---

## Mistake 9: Expecting Semantic Search from InMemoryMemoryService

```python
# ❌ BAD - Expecting synonym matching
# Memory contains: "My favorite color is blue"
result = await memory_service.search_memory(
    query="What is the user's preferred hue?"
)
# Expected: Match "color"
# Reality: Only matches if word "hue" appears in stored text

# ✅ GOOD - Understanding keyword limitations
# Use exact words that appear in stored conversations
result = await memory_service.search_memory(
    query="favorite color"  # Exact words from memory
)

# ✅ PRODUCTION - Use semantic search
# from google.adk.memory import VertexAiMemoryBankService
# Understands "preferred hue" = "favorite color"
```

---

## Mistake 10: No Observability in Production

```python
# ❌ BAD - Flying blind
runner = InMemoryRunner(
    agent=my_agent
    # No plugins = no logs when things break!
)

# ✅ GOOD - Standard observability
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]  # Automatic logging
)

# ✅ BETTER - Custom tracking too
runner = InMemoryRunner(
    agent=my_agent,
    plugins=[
        LoggingPlugin(),      # Standard logs
        MyCustomTracker()     # Business metrics
    ]
)
```

---

## Mistake 11: Only Static Tests (or No Tests!)

```python
# ❌ BAD - No testing at all
# Deploy to production, hope for the best

# ❌ ALSO BAD - Only happy path tests
# Static tests miss edge cases, natural language variations

# ✅ GOOD - Combined strategy
# 1. Static tests for regression (fast, known scenarios)
# 2. User simulation for exploration (edge cases, conversation flow)

# Static test (tests.evalset.json)
{
  "eval_cases": [
    {"eval_id": "basic_light_on", ...}  # Fixed test
  ]
}

# User simulation (user_sim.evalset.json)
{
  "eval_cases": [
    {
      "eval_id": "movie_night",
      "user_simulation": {
        "goal": "Set up for movie night",
        "conversation_plan": "...",
        "max_turns": 8
      }
    }
  ]
}
```

---

# 🏆 Production Checklist

## Tools
- [ ] All functions have detailed docstrings
- [ ] All functions have type hints
- [ ] All functions return `{"status": "...", ...}`
- [ ] Error handling in place
- [ ] `tool_context` parameter for LRO tools

## Long-Running Operations
- [ ] Tool checks `tool_context.tool_confirmation`
- [ ] Tool calls `request_confirmation()` to pause
- [ ] Agent wrapped in `App` with `ResumabilityConfig`
- [ ] Workflow detects `adk_request_confirmation` event
- [ ] Workflow saves and reuses `invocation_id`

## MCP Integration
- [ ] Appropriate MCP server selected
- [ ] `tool_filter` configured for needed tools only
- [ ] Timeout set appropriately (30s+)
- [ ] Connection tested before production

## Sessions
- [ ] Using `DatabaseSessionService` (not InMemory)
- [ ] Context compaction configured if needed
- [ ] Session state keys use prefixes (`user:`, `app:`, `temp:`)
- [ ] Proper error handling for session operations

## Memory
- [ ] Memory service initialized and connected to Runner
- [ ] Ingestion strategy decided (per-turn, end, periodic)
- [ ] Actually ingesting sessions with `add_session_to_memory()`
- [ ] Retrieval pattern chosen based on use case:
  - [ ] `load_memory` for mixed conversations (cost-efficient)
  - [ ] `preload_memory` for mission-critical context (reliable)
- [ ] Agent instructions mention memory tools explicitly
- [ ] Callbacks configured for auto-save (if needed)
- [ ] Understanding keyword vs semantic search limitations
- [ ] Manual verification queries for debugging
- [ ] Consider managed service for production (Vertex AI Memory Bank)

## Observability (Day 4)
- [ ] `LoggingPlugin()` enabled for production
- [ ] Custom plugins for business-critical metrics
- [ ] Error callbacks configured (`on_model_error_callback`)
- [ ] DEBUG logs available but disabled by default
- [ ] Metrics exported to monitoring system
- [ ] Tool call tracking for performance optimization
- [ ] Cost tracking plugin if budget-critical

## Evaluation (Day 4)
- [ ] Static tests (evalset) for core functionality
- [ ] User simulation for edge case discovery
- [ ] Both metrics configured:
  - [ ] `response_match_score` threshold set (typically 0.8)
  - [ ] `tool_trajectory_avg_score` threshold set (typically 1.0)
- [ ] Tests run in CI/CD pipeline
- [ ] Failure analysis automated
- [ ] Test cases cover happy path + edge cases
- [ ] Conversation scenarios test multi-turn flows

## Agent Instructions
- [ ] Reference tools by exact function name
- [ ] Instruct to check "status" field
- [ ] For math: instruct to generate code + use executor
- [ ] Clear error handling guidance

## Multi-Agent
- [ ] Right orchestration pattern selected
- [ ] State passing configured (`output_key` → `{placeholder}`)
- [ ] Loop agents have exit conditions
- [ ] Parallel agents have aggregator if needed

---

# 📚 Key Resources

- [ADK Documentation](https://google.github.io/adk-docs/)
- [ADK Agents Guide](https://google.github.io/adk-docs/agents/)
- [ADK Tools Guide](https://google.github.io/adk-docs/tools/)
- [ADK Runtime (Sessions, Memory)](https://google.github.io/adk-docs/runtime/)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [MCP Server Examples](https://modelcontextprotocol.io/examples)
- [Vertex AI Memory Bank](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/overview)

---

# 🔍 Day 4: Observability & Evaluation

## Part A: Observability (Reactive Debugging)

### **The Problem: Mysterious Agent Failures**

```
User: "Find quantum computing papers"
Agent: "I cannot help with that request."
You: 😭 WHY?? Prompt? Missing tools? API error?
```

**Solution**: Observability gives complete visibility into agent decision-making

---

### **The Three Pillars**

**1. Logs** - What happened
- Individual timestamped events
- "Agent called google_search at 3:45 PM"

**2. Traces** - Why it happened
- Connected sequence of events
- "User asked → Agent called tool → Tool returned → Agent responded"

**3. Metrics** - How well it performed
- Aggregated performance indicators
- "Average response time: 2.3s, Error rate: 0.5%"

---

### **Development: ADK Web UI**

```bash
adk web --log_level DEBUG
```

**Features**:
- Interactive chat interface
- Events tab with chronological actions
- Trace view with timing per step
- Click any span to see detailed parameters

**Debugging Flow**:
1. Run agent in web UI
2. Agent behaves unexpectedly
3. Open Events tab
4. Click problematic span
5. Examine parameters → Find bug!

---

### **Production: Plugins**

**Plugin** = Custom code at agent lifecycle hooks

**Architecture**:
```
User message → before_agent → Agent thinks → before_tool
→ Tool runs → after_tool → before_model → LLM call
→ after_model → Response
```

**Built-in LoggingPlugin:**

```python
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]  # Auto-logging enabled!
)
```

**Captures**:
- ✅ User messages and agent responses
- ✅ Timing data for performance
- ✅ LLM requests and responses
- ✅ Tool calls and results
- ✅ Complete execution traces

---

### **Custom Plugins**

**Example: Tool Call Tracker**

```python
from google.adk.plugins.base_plugin import BasePlugin

class ToolCallTrackerPlugin(BasePlugin):
    def __init__(self):
        super().__init__(name="tool_call_tracker")
        self.total_tool_calls = 0
        self.tool_call_counts = {}

    async def before_tool_callback(self, *, agent, callback_context):
        """Runs BEFORE each tool call."""
        self.total_tool_calls += 1
        tool_name = callback_context.tool_name
        self.tool_call_counts[tool_name] = \
            self.tool_call_counts.get(tool_name, 0) + 1
        logging.info(f"Tool #{self.total_tool_calls}: {tool_name}")

    def get_report(self):
        return f"Total: {self.total_tool_calls}\n" + \
               f"Breakdown: {self.tool_call_counts}"
```

**Usage**:
```python
tracker = ToolCallTrackerPlugin()
runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin(), tracker]  # Combine multiple!
)
```

---

### **When to Use What: Observability**

| Scenario | Use | Why |
|----------|-----|-----|
| Development | `adk web --log_level DEBUG` | Interactive, visual |
| Production standard | `LoggingPlugin()` | Automated logging |
| Custom metrics | Build custom plugin | Business-specific needs |
| Performance tracking | Custom plugin with timing | Optimize slow paths |
| Cost tracking | Custom plugin counting calls | Budget monitoring |

---

## Part B: Evaluation (Proactive Prevention)

### **The Problem: Non-Deterministic Behavior**

**Why Standard Testing Fails for Agents**:
- Agents are non-deterministic (LLM variability)
- Users give unpredictable commands
- Small prompt changes = dramatic behavior shifts
- Can't just test "output == expected"

**Solution**: Evaluate entire decision-making process

---

### **Evaluation Workflow**

```
1. Create Config     → Define pass/fail thresholds
2. Create Test Cases → Sample conversations to compare
3. Run Agent        → Execute with test queries
4. Compare Results  → Measure against expectations
```

---

### **The Two Metrics**

**1. response_match_score**
- Measures text similarity
- Range: 0.0 (different) to 1.0 (perfect)
- Threshold: Often 0.8 (80% similarity)

**2. tool_trajectory_avg_score**
- Measures correct tool usage
- Range: 0.0 (wrong tools) to 1.0 (perfect)
- Checks: Tool names, parameters, sequence
- Threshold: Often 1.0 (exact match required)

---

### **Interactive Testing: ADK Web UI**

**Create Test Cases**:
1. Have conversation in web UI
2. Navigate to **Eval** tab
3. Click **Create Evaluation set**
4. Name it (e.g., "home_automation_tests")
5. Click **Add current session**

**Run Evaluation**:
1. Check test cases to run
2. Click **Run Evaluation**
3. Configure metrics
4. View Pass/Fail results
5. Analyze failures with side-by-side comparison

---

### **Automated Testing: CLI**

**1. Create Config** (`test_config.json`):
```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}
```

**2. Create Test Cases** (`tests.evalset.json`):
```json
{
  "eval_set_id": "home_automation_suite",
  "eval_cases": [
    {
      "eval_id": "living_room_light_on",
      "conversation": [
        {
          "user_content": {
            "parts": [{"text": "Turn on the floor lamp"}]
          },
          "final_response": {
            "parts": [{"text": "Successfully set the floor lamp to on."}]
          },
          "intermediate_data": {
            "tool_uses": [
              {
                "name": "set_device_status",
                "args": {"location": "living room", "device_id": "floor lamp", "status": "ON"}
              }
            ]
          }
        }
      ]
    }
  ]
}
```

**3. Run Evaluation**:
```bash
adk eval home_automation_agent \
    tests.evalset.json \
    --config_file_path=test_config.json \
    --print_detailed_results
```

---

### **User Simulation (Dynamic Testing)**

**The Problem with Static Tests**:
- Fixed, predictable prompts
- Can't test conversation flow
- Miss edge cases
- No multi-turn complexity

**Solution**: LLM generates dynamic user prompts

---

### **Creating User Simulation**

```python
from google.adk.evaluation import ConversationScenario

scenario = ConversationScenario(
    scenario_id="movie_night_preparation",
    goal="Set up living room for movie night",
    conversation_plan="""
    You are preparing for movie night with friends.

    Plan:
    1. Ask to dim the lights in the living room
    2. Request to turn on the TV
    3. Ask to adjust temperature (test limitation)
    4. Request ambient lighting for mood

    Be natural and conversational. Point out confusion politely.
    Goal is complete when lights/TV controlled and you understand limitations.
    """,
    max_turns=8
)
```

**Conversation Plan Best Practices**:

```python
# ❌ BAD - Too specific
"Ask: 'Turn on the living room light'"

# ✅ GOOD - Natural, flexible
"You want light in the living room. Ask naturally."
```

---

### **User Simulation Evalset**

```json
{
  "eval_set_id": "home_automation_user_sim",
  "eval_cases": [
    {
      "eval_id": "movie_night_preparation",
      "user_simulation": {
        "goal": "Set up living room for movie night",
        "conversation_plan": "Step-by-step plan...",
        "max_turns": 8
      }
    }
  ]
}
```

**Run**:
```bash
adk eval home_automation_agent \
    user_simulation.evalset.json \
    --config_file_path=test_config.json
```

---

### **Static vs User Simulation**

| Aspect | Static Tests | User Simulation |
|--------|-------------|-----------------|
| **User Prompts** | Fixed | Dynamically generated |
| **Flow** | Single-turn | Multi-turn dialogue |
| **Edge Cases** | Manually defined | Naturally discovered |
| **Realism** | Limited | High |
| **Coverage** | Happy path | Happy + edge cases |
| **Cost** | Low | Higher (LLM usage) |
| **Best For** | Regression testing | Exploration |

---

### **What User Simulation Reveals**

**Example Findings**:

**1. Agent Over-Promises**
- Says "I control ALL smart devices"
- Reality: Only has ON/OFF toggle
- Discovered when user asked about coffee maker

**2. Missing Validation**
- Accepts any device name
- Doesn't check if device exists
- Discovered with non-existent devices

**3. Poor Error Handling**
- Tries to help even when it can't
- Should state limitations clearly
- Discovered in multi-turn conversations

**4. Context Blindness**
- Doesn't track conversation history
- Repeats questions, loses context
- Discovered with sequential commands

---

### **When to Use What: Evaluation**

| Scenario | Use | Why |
|----------|-----|-----|
| Regression testing | Static tests | Known scenarios, fast |
| Core functionality | Static tests | Exact behavior expected |
| Quick validation | Static tests | Low cost |
| Edge case discovery | User simulation | Unpredictable inputs |
| Conversation testing | User simulation | Multi-turn complexity |
| Realistic behavior | User simulation | Natural language variation |
| **Best Practice** | **Both combined** | **Comprehensive coverage** |

---

# 🔜 Day 5: Capstone Project (Coming Soon)

_Content will be added once notebooks are available_

**Expected topics:**
- End-to-end agent system
- Vertex AI Memory Bank integration
- Production-ready patterns
- Best practices synthesis
- Real-world deployment

---

## 🎯 Quick Reference Card

### **When to Use What**

| Need | Use |
|------|-----|
| Fixed order execution | `SequentialAgent` |
| Concurrent independent tasks | `ParallelAgent` |
| Iterative improvement | `LoopAgent` |
| Dynamic routing | LLM Orchestrator |
| External service integration | MCP Tools |
| Human approval workflow | Long-Running Operations |
| Conversation history (short-term) | Sessions |
| Cross-conversation knowledge (long-term) | Memory |
| Efficient memory (agent decides) | `load_memory` |
| Reliable memory (always loaded) | `preload_memory` |
| Semantic search (production) | Vertex AI Memory Bank |
| Keyword search (development) | InMemoryMemoryService |
| Math/calculations | `BuiltInCodeExecutor` |
| Specialist delegation | `AgentTool` |
| Development debugging | `adk web --log_level DEBUG` |
| Production observability | `LoggingPlugin()` |
| Custom metrics tracking | Build custom plugin |
| Regression testing | Static tests (evalset) |
| Edge case discovery | User simulation |
| Comprehensive testing | Both static + user simulation |

### **Must-Remember Patterns**

1. **Always return** `{"status": "...", ...}` from tools
2. **Always save** `invocation_id` for LRO resume
3. **Always use** docstrings and type hints
4. **State passing**: `output_key` → `{placeholder}`
5. **Memory workflow**: Initialize → Ingest → Retrieve
6. **Memory choice**: `load_memory` for efficiency, `preload_memory` for reliability
7. **Tool call visibility**: `load_memory` shows `function_call`, `preload_memory` is silent
8. **Observability**: Development = Web UI, Production = LoggingPlugin
9. **Evaluation**: Use both static tests (regression) + user simulation (exploration)
10. **Plugin callbacks**: `before_tool_callback` / `after_tool_callback` for custom tracking

---

## 📝 Glossary

- **ADK**: Agent Development Kit - Google's framework for building AI agents
- **Agent**: System that can think, act, and observe (not just respond)
- **Runner**: Orchestrator managing conversation flow
- **Session**: Container for single conversation (events + state)
- **Memory**: Long-term knowledge store across conversations
- **Tool**: Function/API that agent can call
- **LRO**: Long-Running Operation (pauses for external input)
- **MCP**: Model Context Protocol (standard for external integrations)
- **Consolidation**: Extracting key facts from raw conversations
- **invocation_id**: Unique ID for agent execution (critical for resume)
- **ToolContext**: ADK-provided object with state, memory, confirmation access
- **Callback**: Function that runs at specific execution points
- **Plugin**: Custom code that runs at agent lifecycle hooks
- **Observability**: Logs (what) + Traces (why) + Metrics (how well)
- **Evalset**: JSON file containing test cases for agent evaluation
- **User Simulation**: LLM-generated dynamic user prompts for testing
- **response_match_score**: Text similarity metric (0.0-1.0)
- **tool_trajectory_avg_score**: Tool usage correctness metric (0.0-1.0)

---

**Last Updated**: Based on Day 1-4 notebooks
**Version**: 2.0
**Status**: Day 5 pending

# Day 2: Agent Architecture & Multi-Agent Systems

## Overview
Day 2 covers custom agent tools, external integrations via MCP, and long-running operations with human-in-the-loop patterns. Two notebooks: **2a (Agent Tools)** and **2b (Tool Patterns & Best Practices)**.

---

## 📘 Notebook 2a: Agent Tools

### Core Concepts

**1. Custom Function Tools**
- Turn ANY Python function into an agent tool by adding it to `tools=[]` list
- Example: `get_fee_for_payment_method()`, `get_exchange_rate()`
- Agent decides when to call based on docstrings

**2. Agent Tools (Agents as Tools)**
- Use `AgentTool(agent=specialist_agent)` to make one agent a tool for another
- Different from sub-agents: specialist returns control to parent agent
- Example: Currency agent delegates to calculation agent for math

**3. Built-in Code Executor**
- `BuiltInCodeExecutor()` runs Python in sandbox via Gemini
- More reliable than LLM doing math directly
- Agent generates code → executor runs it → agent uses result

### Tool Type Categories

**Custom Tools:**
- Function Tools (simple Python functions)
- Long Running Function Tools (async, human-in-loop)
- Agent Tools (agents as tools)
- MCP Tools (external service integration)
- OpenAPI Tools (auto-generated from specs)

**Built-in Tools:**
- Gemini Tools (`google_search`, `BuiltInCodeExecutor`)
- Google Cloud Tools (`BigQueryToolset`, `SpannerToolset`)
- Third-party Tools (Hugging Face, GitHub, etc.)

---

## 📗 Notebook 2b: Tool Patterns & Best Practices

### Model Context Protocol (MCP)

**What:** Standardized protocol to connect agents to external services
- **MCP Server:** Provides tools (GitHub, Slack, databases, etc.)
- **MCP Client:** Your agent that uses those tools
- **Benefit:** No custom integration code needed

**Usage Pattern:**
```python
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
```

**Popular MCP Servers:**
- Kaggle MCP (datasets, notebooks)
- GitHub MCP (PRs, issues)
- Everything MCP (testing)

### Long-Running Operations (LRO)

**What:** Tools that pause execution to wait for external input (human approval, background tasks)

**When to Use:**
- Financial transactions requiring approval
- Bulk operations (delete 1000 records)
- High-cost actions (spin up 50 servers)
- Compliance checkpoints
- Irreversible operations

**Three Key Components:**

**1. The Tool with ToolContext:**
```python
def place_order(num_containers: int, destination: str, tool_context: ToolContext) -> dict:
    # Auto-approve small orders
    if num_containers <= 5:
        return {"status": "approved", ...}

    # FIRST CALL: Request approval for large orders
    if not tool_context.tool_confirmation:
        tool_context.request_confirmation(
            hint=f"Approve {num_containers} containers?",
            payload={"num_containers": num_containers, "destination": destination}
        )
        return {"status": "pending", ...}

    # SECOND CALL: Handle approval response
    if tool_context.tool_confirmation.confirmed:
        return {"status": "approved", ...}
    else:
        return {"status": "rejected", ...}
```

**2. Resumable App:**
```python
app = App(
    name="shipping_coordinator",
    root_agent=agent,
    resumability_config=ResumabilityConfig(is_resumable=True)
)
```

**3. Workflow with Event Detection:**
- Detect `adk_request_confirmation` event
- Extract `invocation_id` and `approval_id`
- Get human decision
- Resume with same `invocation_id`

---

## 🎯 ADK Best Practices (Critical!)

### 1. Tool Return Format
**Always return dictionaries with status:**
```python
# Success
{"status": "success", "data": result}

# Error
{"status": "error", "error_message": "reason"}
```

### 2. Function Signature Requirements
```python
def my_tool(param: str, tool_context: ToolContext) -> dict:
    """Clear docstring explaining when to use this tool.

    Args:
        param: Description of parameter

    Returns:
        Dictionary with status and data
    """
    pass
```

**Must have:**
- ✅ Clear docstring (LLM reads this!)
- ✅ Type hints (`str`, `dict`, `int`)
- ✅ Dictionary return values
- ✅ Error handling
- ✅ `tool_context: ToolContext` parameter for LRO

### 3. Agent Instructions
- Reference tools by exact function name
- Explain when to use each tool
- Tell agent to check "status" field in responses
- For calculations, instruct to generate Python code

---

## 🚨 Vibe Coder Watch-Outs

### Pattern 1: Tool Definition Gotchas

**❌ Bad - No structure:**
```python
def get_rate(base, target):
    return 0.93  # LLM confused: is this rate or error?
```

**✅ Good - Structured with status:**
```python
def get_rate(base_currency: str, target_currency: str) -> dict:
    """Gets exchange rate between currencies."""
    return {"status": "success", "rate": 0.93}
```

### Pattern 2: Agent Tool vs Sub-Agent

**Agent Tool (what 2a uses):**
- Agent A calls Agent B **as a function**
- Agent B returns result **back to Agent A**
- Agent A continues working
- Use case: Delegation (calculations, specialist tasks)

**Sub-Agent (different):**
- Agent A **transfers control completely** to Agent B
- Agent B handles all future messages
- Agent A is out of the loop
- Use case: Handoff (customer support tiers)

### Pattern 3: LRO Event Handling Flow

**The critical invocation_id pattern:**
```python
# STEP 1: Initial call
async for event in runner.run_async(...):
    events.append(event)

# STEP 2: Detect pause
approval_info = check_for_approval(events)  # Returns {approval_id, invocation_id}

# STEP 3: Resume with SAME invocation_id
if approval_info:
    async for event in runner.run_async(
        ...,
        new_message=approval_response,
        invocation_id=approval_info["invocation_id"]  # ← CRITICAL!
    ):
        # Process resumed execution
```

**Without matching invocation_id:** ADK starts NEW execution instead of resuming!

### Pattern 4: ToolContext States

**The tool gets called TWICE for LRO:**

**Call 1 (Pause):**
- `tool_context.tool_confirmation` is `None`
- Call `request_confirmation()`
- Return `{"status": "pending"}`
- ADK emits `adk_request_confirmation` event

**Call 2 (Resume):**
- `tool_context.tool_confirmation` exists
- Check `tool_confirmation.confirmed` (True/False)
- Return final result

### Pattern 5: MCP Server Types

**NPM-based (most common):**
```python
command="npx"
args=["-y", "mcp-remote", "url"]
```

**HTTP-based (GitHub Copilot):**
```python
StreamableHTTPServerParams(
    url="https://api.githubcopilot.com/mcp/",
    headers={"Authorization": f"Bearer {token}"}
)
```

### Pattern 6: Code Execution Trick

**Instead of:** "Agent, calculate the final amount"
**Do:** "Agent, generate Python code to calculate, then use BuiltInCodeExecutor"

More reliable! LLMs are bad at math, good at code generation.

---

## 🔑 Key Takeaways

1. **Any function → Tool:** Just add to `tools=[]` with proper docstring + type hints
2. **Agents as Tools:** Use `AgentTool()` for delegation, not handoff
3. **MCP = No Integration Code:** Connect to external services instantly
4. **LRO = Three Parts:** Tool with ToolContext + Resumable App + Event-driven workflow
5. **invocation_id is sacred:** Lose it = lose ability to resume
6. **Status field everywhere:** Always return `{"status": "...", ...}`

---

## 🎓 Vibe Coding Checklist

When building similar projects:

### For Custom Tools:
- [ ] Return structured dictionaries with "status" field
- [ ] Write detailed docstrings (LLM reads them!)
- [ ] Add type hints to all parameters
- [ ] Handle errors gracefully with status "error"
- [ ] Test tools standalone before adding to agent

### For Long-Running Operations:
- [ ] Add `tool_context: ToolContext` parameter
- [ ] Check `if not tool_context.tool_confirmation:` for first call
- [ ] Call `tool_context.request_confirmation()` to pause
- [ ] Check `tool_context.tool_confirmation.confirmed` for resume
- [ ] Wrap agent in `App` with `ResumabilityConfig(is_resumable=True)`
- [ ] Create workflow that detects `adk_request_confirmation` event
- [ ] Save and reuse `invocation_id` when resuming
- [ ] Use `Runner` with `app=` not `agent=`

### For MCP Integration:
- [ ] Find appropriate MCP server (check modelcontextprotocol.io)
- [ ] Use `tool_filter` to only expose needed tools
- [ ] Set reasonable timeout (30s+)
- [ ] Test connection before adding complex logic

### For Agent Instructions:
- [ ] Reference tools by exact function name
- [ ] Tell agent to check "status" field in responses
- [ ] For math: instruct to generate Python code + use executor
- [ ] Be explicit about error handling

---

## 💡 Common Mistakes

1. **Forgetting tool_context parameter** → LRO won't work
2. **Not returning status dictionaries** → Agent confused by responses
3. **Starting new run instead of resuming** → Forgot to pass invocation_id
4. **No docstrings** → LLM doesn't know when to use tool
5. **Using sub-agents when you need agent tools** → Control flow breaks

---

## 📚 Links & Resources

- [ADK Documentation](https://google.github.io/adk-docs/)
- [ADK Tools Guide](https://google.github.io/adk-docs/tools/)
- [ADK Function Tools](https://google.github.io/adk-docs/tools/function-tools/)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [MCP Server Examples](https://modelcontextprotocol.io/examples)
- [ADK Runtime (App, Runner)](https://google.github.io/adk-docs/runtime/)

---

## 🎯 Next Steps

- Day 3: State and Memory Management
- Practice: Build image generation agent with cost approval (exercise in 2b)
- Explore: More MCP servers (Kaggle, GitHub, Slack)
- Deep dive: Sessions, Runners, and Events (Day 3 preview)

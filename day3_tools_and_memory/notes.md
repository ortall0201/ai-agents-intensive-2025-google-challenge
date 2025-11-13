# Day 3: Sessions & Memory Management

## 📚 Overview

Day 3 covers the **runtime components** that make agents stateful and context-aware:
- **Part A (3a)**: **Sessions** - Short-term conversation memory
- **Part B (3b)**: **Memory** - Long-term knowledge storage

---

## 📘 Part A: Sessions (Short-term Memory)

### 🎯 Core Concept: Making LLMs Stateful

**Problem**: LLMs are stateless - they forget everything after each call.

**Solution**: Sessions provide conversation continuity.

---

### What is a Session?

**Session** = Container for a single, continuous conversation

```
Session
├── Events[] ← Conversation building blocks
│   ├── User messages
│   ├── Agent responses
│   ├── Tool calls
│   └── Tool outputs
└── State{} ← Scratchpad for key-value storage
    ├── user:name → "Sam"
    └── user:location → "NYC"
```

---

### Architecture

```
Runner
  ↓
SessionService (Storage Layer)
  ↓
├── Session 1 (User A - Chat 1)
├── Session 2 (User A - Chat 2)
└── Session 3 (User B - Chat 1)
```

**Key Points**:
- Sessions are **isolated** - don't share data
- Runner **automatically** maintains conversation history
- Each session has unique ID

---

### SessionService Implementations

**1. InMemorySessionService**
```python
session_service = InMemorySessionService()
```
- ✅ Fast, simple
- ❌ Lost on restart
- 🎯 Use for: Development, testing

**2. DatabaseSessionService**
```python
session_service = DatabaseSessionService(db_url="sqlite:///data.db")
```
- ✅ Persistent storage
- ✅ Survives restarts
- ✅ Query historical data
- 🎯 Use for: Production

**3. Agent Engine Sessions (GCP)**
- Fully managed cloud service
- Enterprise scale

---

### Context Compaction

**Problem**: Long conversations = huge context = $$$

**Solution**: Auto-summarize old turns

```python
from google.adk.apps.app import App, EventsCompactionConfig

app = App(
    name="my_app",
    root_agent=agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compact every 3 turns
        overlap_size=1          # Keep last 1 turn
    )
)
```

**What happens**:
1. After 3 turns → trigger compaction
2. LLM summarizes first 2 turns
3. Replace with 1 summary
4. Keep last turn for overlap

**Benefits**: Faster, cheaper, maintains context

---

### Session State

**Use Case**: Share data across tools/agents within session

```python
def save_userinfo(tool_context: ToolContext, name: str, country: str) -> dict:
    """Save to session state."""
    tool_context.state["user:name"] = name
    tool_context.state["user:country"] = country
    return {"status": "success"}

def retrieve_userinfo(tool_context: ToolContext) -> dict:
    """Read from session state."""
    name = tool_context.state.get("user:name", "Not found")
    return {"status": "success", "user_name": name}
```

**Best Practices**:
- Use prefixes: `user:`, `app:`, `temp:`
- State is **session-scoped** (not shared across sessions)
- Persists throughout session lifetime

---

## 📗 Part B: Memory (Long-term Knowledge)

### 🎯 Core Concept: Cross-Conversation Recall

**Session vs Memory**:

| Session | Memory |
|---------|--------|
| Single conversation | All conversations |
| Short-term (RAM) | Long-term (Database) |
| Raw events | Extracted facts |
| Chronological | Searchable |

**Example**:
- **Session**: Remembers what you said 10 minutes ago
- **Memory**: Remembers your preferences from last week

---

### The 3-Step Memory Workflow

```
1. Initialize → Create MemoryService
2. Ingest    → Transfer sessions to memory
3. Retrieve  → Search stored memories
```

---

### Step 1: Initialize

```python
from google.adk.memory import InMemoryMemoryService

# Create memory service
memory_service = InMemoryMemoryService()

# Create runner with BOTH services
runner = Runner(
    agent=agent,
    app_name="my_app",
    session_service=session_service,
    memory_service=memory_service  # Memory available!
)
```

**⚠️ CRITICAL**: Adding `memory_service` makes it *available*, but doesn't automatically use it!

You must:
1. **Ingest** data with `add_session_to_memory()`
2. **Enable retrieval** with memory tools

---

### Step 2: Ingest

```python
# Have conversation
await run_session(runner, "My favorite color is blue", "session-01")

# Get session
session = await session_service.get_session(
    app_name="my_app",
    user_id="user123",
    session_id="session-01"
)

# Transfer to memory
await memory_service.add_session_to_memory(session)
```

**Manual ingestion** - you call it explicitly.

---

### Step 3: Retrieve - Two Patterns

#### **Pattern A: load_memory (Reactive)**

Agent **decides** when to search.

```python
from google.adk.tools import load_memory

agent = LlmAgent(
    instruction="Use load_memory if you need past info.",
    tools=[load_memory]
)

# Question: "What is my favorite color?"
# Output: WARNING: non-text parts in response: ['function_call']
# Behavior: You SEE the tool call
```

**Pros**:
- ✅ Efficient (only searches when needed)
- ✅ Lower token cost

**Cons**:
- ⚠️ Agent might forget to search

---

#### **Pattern B: preload_memory (Proactive)**

Automatically loads **before every turn**.

```python
from google.adk.tools import preload_memory

agent = LlmAgent(
    instruction="Answer questions.",
    tools=[preload_memory]
)

# Question: "What is my favorite color?"
# Output: Clean response, no warnings
# Behavior: Tool call is INVISIBLE (happens before agent thinks)
```

**Pros**:
- ✅ Guaranteed context
- ✅ Reliable

**Cons**:
- ❌ Searches EVERY turn (even for jokes!)
- ❌ Higher token cost

---

### 🔥 The Critical Difference

**Testing Both:**

| Pattern | Query | Visible Tool Call? | Memory Searched? |
|---------|-------|-------------------|------------------|
| `load_memory` | "What's my color?" | ✅ YES (warning shows) | ✅ YES |
| `load_memory` | "Tell me a joke" | ⭕ NO | ❌ NO |
| `preload_memory` | "What's my color?" | ⭕ NO (silent) | ✅ YES |
| `preload_memory` | "Tell me a joke" | ⭕ NO (silent) | ✅ YES (wasteful!) |

**The Wasteful Pattern:**

Even when asking for a joke, `preload_memory` loads ALL memories (birthday, color, etc.) into context!

```python
# Proof
joke_memories = await memory_service.search_memory(
    query="Tell me a joke"
)
print(f"Loaded: {len(joke_memories.memories)} memories")
# Output: 4 memories - all unnecessary!
```

---

### When to Use Which?

| Use Case | Pattern | Why |
|----------|---------|-----|
| Personal assistant | `load_memory` | Most queries don't need full history |
| Medical diagnosis | `preload_memory` | MUST always know allergies |
| Banking support | `preload_memory` | Must know account restrictions |
| General chatbot | `load_memory` | Mixed conversations |
| Compliance apps | `preload_memory` | Reliability > efficiency |

---

### 🔬 Keyword Matching (InMemoryMemoryService)

**How It Works**: Literal word matching - searches for exact words.

**Test Results:**

Memory contains:
- "My favorite **color** is blue-green. Can you write a **Haiku** about it?"
- "Ocean meets the sky, A calming, vibrant blend, Nature's peaceful **hue**."
- "My birthday is on March 15th."

| Query | Matches? | Why? |
|-------|----------|------|
| "what color does the user like" | ✅ YES | Word "**color**" exists |
| "haiku" | ✅ YES | Word "**haiku**" exists |
| "age" | ❌ NO | Never mentioned (only "birthday") |
| "preferred hue" | ✅ YES | Word "**hue**" in poem! |

---

### 🚨 Important Discovery: "preferred hue" Matches!

**Why?** The agent's haiku contains "Nature's peaceful **hue**"

**This shows**:
1. ✅ **Prevents hallucination**: Only finds what exists
2. ❌ **Literal matching**: Finds "hue" anywhere (even poems!)
3. ❌ **No semantics**: Doesn't understand "hue" = "color"
4. ❌ **No synonyms**: "preferred" ≠ "favorite"

**Production Solution**: Use Vertex AI Memory Bank (semantic search)

---

### Automating Memory with Callbacks

**Problem**: Manually calling `add_session_to_memory()` is tedious.

**Solution**: Use `after_agent_callback` to auto-save.

```python
# Define callback
async def auto_save_to_memory(callback_context):
    """Auto-save session to memory after each turn."""
    await callback_context._invocation_context.memory_service.add_session_to_memory(
        callback_context._invocation_context.session
    )

# Create agent with callback
agent = LlmAgent(
    instruction="Answer questions.",
    tools=[preload_memory],
    after_agent_callback=auto_save_to_memory  # Auto-saves!
)
```

**Result**: Zero manual memory calls!

---

### Callback Types

| Callback | When | Use For |
|----------|------|---------|
| `before_agent_callback` | Before agent starts | Setup, validation |
| `after_agent_callback` | After agent completes | **Auto-save memory**, logging |
| `before_tool_callback` | Before tool call | Authorization |
| `after_tool_callback` | After tool call | Logging |
| `on_model_error_callback` | On errors | Error handling |

---

### Memory Consolidation

**Problem**: Storing raw events doesn't scale.

**Before (Raw)**:
```
User: "My favorite color is blue. I also like purple. Actually, blue is best."
Agent: "Great!"
User: "Thanks!"
Agent: "Welcome!"

→ Stores ALL 4 messages
```

**After (Consolidated)**:
```
Extracted Memory: "User's favorite color: blue"

→ Stores 1 concise fact
```

**How (Managed Services)**:
```
Raw Events → LLM analyzes → Extracts facts → Stores concise → Deduplicates
```

**Benefits**:
- Less storage
- Faster retrieval
- More accurate

---

### MemoryService Implementations

**1. InMemoryMemoryService (This Course)**
- Stores raw events
- Keyword matching
- In-memory (resets on restart)
- 🎯 Use for: Learning, development

**2. VertexAiMemoryBankService (Production)**
- LLM-powered extraction
- Semantic search (meaning-based)
- Persistent cloud storage
- External knowledge integration
- 🎯 Use for: Production

**💡 API Consistency**: Both use same methods!

---

### Manual Memory Search

```python
# Search memory
search_response = await memory_service.search_memory(
    app_name="my_app",
    user_id="user123",
    query="What is the user's favorite color?"
)

print(f"Found {len(search_response.memories)} memories")

# Get ALL memories (debugging)
all_memories = await memory_service.search_memory(
    app_name="my_app",
    user_id="user123",
    query=""  # Empty = return all
)
```

**Use Cases**:
- Debugging why searches fail
- Verifying memory ingestion
- Building analytics dashboards

---

## 🎯 Key Takeaways

### Sessions

1. **Sessions = Short-term**: Single conversation container
2. **Two components**: Events (messages) + State (key-value)
3. **DatabaseSessionService**: Always use in production
4. **Context Compaction**: Auto-summarize to save tokens
5. **State scoping**: Use prefixes (`user:`, `app:`, `temp:`)

### Memory

1. **Memory = Long-term**: Cross-conversation knowledge
2. **3-step workflow**: Initialize → Ingest → Retrieve
3. **Two retrieval patterns**:
   - `load_memory`: Efficient, agent decides
   - `preload_memory`: Reliable, always loads
4. **Keyword vs Semantic**: InMemory (literal) vs Vertex AI (meaning)
5. **Callbacks = Automation**: `after_agent_callback` for auto-save
6. **Consolidation**: Extract facts, discard noise (managed services)

---

## 🚨 Common Pitfalls

### Session Mistakes

1. **Using InMemorySessionService in production** → Lost on restart
2. **Not using context compaction** → High token costs
3. **Sharing state across sessions** → Not possible, sessions are isolated

### Memory Mistakes

1. **Initializing memory but not ingesting** → Memory stays empty
2. **Using preload_memory everywhere** → Wastes tokens
3. **Expecting semantic search from InMemory** → Only keyword matching
4. **Forgetting to add memory tools** → Agent can't access memory
5. **Not understanding visibility** → load_memory shows calls, preload doesn't

---

## 🔥 Vibe Coder Tips

### Session Tips

1. **Always use DatabaseSessionService** unless prototyping
2. **Enable compaction** for long conversations
3. **Use state for temporary data** that tools need to share
4. **Session IDs** should be unique per conversation

### Memory Tips

1. **Start with load_memory** (efficient) unless reliability is critical
2. **Test both patterns** to see tool call visibility
3. **Debug with empty query** to see all stored memories
4. **Callbacks = zero manual work** for memory ingestion
5. **Keyword matching is literal** - if word doesn't exist, no match
6. **Semantic search** requires Vertex AI (Day 5!)

### Cost Optimization

1. **Use load_memory** for general chatbots (saves tokens)
2. **Enable context compaction** (reduces context size)
3. **Manual memory ingestion** at end of conversation (not per-turn)
4. **Filter memories** by relevance before injection

### Reliability Patterns

1. **Use preload_memory** for medical/financial apps
2. **DatabaseSessionService** always (not InMemory)
3. **Callbacks for auto-save** (don't forget to ingest!)
4. **Test with empty memories** to ensure proper error handling

---

## 📊 Quick Decision Matrix

### Choosing SessionService

| Scenario | Use |
|----------|-----|
| Local development | InMemorySessionService |
| Production deployment | DatabaseSessionService |
| Enterprise scale | Agent Engine Sessions (GCP) |

### Choosing Memory Pattern

| Scenario | Use |
|----------|-----|
| Cost-sensitive app | `load_memory` |
| Mission-critical context | `preload_memory` |
| General chatbot | `load_memory` |
| Healthcare/Finance | `preload_memory` |

### Choosing MemoryService

| Scenario | Use |
|----------|-----|
| Learning/Development | InMemoryMemoryService |
| Production | VertexAiMemoryBankService |
| Need semantic search | VertexAiMemoryBankService |
| Keyword matching OK | InMemoryMemoryService |

---

## 🧪 Test Patterns from Notebooks

### Session State Pattern

```python
# Tool 1: Write
def save_info(tool_context: ToolContext, name: str) -> dict:
    tool_context.state["user:name"] = name
    return {"status": "success"}

# Tool 2: Read
def get_info(tool_context: ToolContext) -> dict:
    name = tool_context.state.get("user:name", "Unknown")
    return {"status": "success", "name": name}
```

### Memory Ingestion Pattern

```python
# Manual
session = await session_service.get_session(...)
await memory_service.add_session_to_memory(session)

# Automated
async def auto_save(callback_context):
    await callback_context._invocation_context.memory_service \
        .add_session_to_memory(callback_context._invocation_context.session)

agent = LlmAgent(..., after_agent_callback=auto_save)
```

### Memory Retrieval Pattern

```python
# Reactive
agent = LlmAgent(
    instruction="Use load_memory if needed.",
    tools=[load_memory]
)

# Proactive
agent = LlmAgent(
    instruction="Answer questions.",
    tools=[preload_memory]
)
```

### Debugging Pattern

```python
# See what's actually stored
all_memories = await memory_service.search_memory(
    app_name=APP_NAME,
    user_id=USER_ID,
    query=""  # Empty = all memories
)

for mem in all_memories.memories:
    print(f"[{mem.author}]: {mem.content.parts[0].text}")
```

---

## 📚 Links & Resources

### Official Documentation
- [ADK Sessions Documentation](https://google.github.io/adk-docs/sessions/)
- [ADK Memory Documentation](https://google.github.io/adk-docs/sessions/memory/)
- [Vertex AI Memory Bank](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/overview)
- [Memory Consolidation Guide](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/memory-bank/generate-memories)
- [ADK Callbacks](https://google.github.io/adk-docs/agents/callbacks/)

### Key Concepts
- Session = Short-term (single conversation)
- Memory = Long-term (all conversations)
- State = Scratchpad within session
- Consolidation = Extract facts from raw events
- Keyword matching = Literal word search
- Semantic search = Meaning-based (embeddings)

---

## 🎯 Next Steps

1. **Day 4**: Observability & Evaluation
2. **Day 5**: Vertex AI Memory Bank (semantic search!)
3. **Practice**: Build chatbot with both patterns
4. **Compare**: Test cost difference between load/preload
5. **Experiment**: Try different compaction intervals

---

## ❓ Questions to Explore

1. When does context compaction actually save money?
2. How much overhead does preload_memory add?
3. What's the best ingestion frequency for your use case?
4. How to handle memory retrieval failures gracefully?
5. When should you clear old memories?

---

**🔥 Pro Tip**: Always start with `load_memory` + manual ingestion. Add `preload_memory` and callbacks only when you confirm the need!

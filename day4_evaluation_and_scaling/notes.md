# Day 4: Observability & Evaluation

## 📚 Overview

Day 4 covers the **production-readiness components** for AI agents:
- **Part A (4a)**: **Observability** - Logs, Traces & Metrics (Reactive debugging)
- **Part B (4b)**: **Evaluation** - Testing & Quality Assurance (Proactive prevention)

```
Observability (Reactive)  +  Evaluation (Proactive)  =  Production Ready
```

---

## 📘 Part A: Agent Observability

### 🎯 Core Concept: Debugging AI Agents

**Problem**: AI agents fail mysteriously

```
User: "Find quantum computing papers"
Agent: "I cannot help with that request."
You: 😭 WHY?? Prompt? Missing tools? API error?
```

**Solution**: Observability gives complete visibility into decision-making

```
DEBUG Log: LLM Request shows "Functions: []" (no tools!)
You: 🎯 Aha! Missing google_search tool - easy fix!
```

---

### The Three Pillars of Observability

**1. Logs** - Record of what happened
- Individual events
- Timestamped entries
- "The lights turned on at 3:45 PM"

**2. Traces** - Connected story of why
- Sequence of events
- Cause-and-effect chain
- "User asked → Agent called tool → Tool returned → Agent responded"

**3. Metrics** - Summary of how well
- Aggregated numbers
- Performance indicators
- "Average response time: 2.3s, Error rate: 0.5%"

---

### Log Levels in ADK

| Level | When to Use | Example |
|-------|-------------|---------|
| **DEBUG** | Development, detailed debugging | Full LLM prompts, tool parameters |
| **INFO** | Normal operations | "Agent started", "Tool called" |
| **WARNING** | Potential issues | "Rate limit approaching" |
| **ERROR** | Failures | "Tool execution failed" |

---

### ADK Web UI - Interactive Debugging

**Launch with DEBUG logs:**
```bash
adk web --log_level DEBUG
```

**Features:**
1. **Chat Interface**: Test agent conversations
2. **Events Tab**: Chronological action list
3. **Trace View**: Timing information per step
4. **Span Details**: Click any event to see details

**Debugging Workflow:**
```
1. Run agent in web UI
2. Agent behaves unexpectedly
3. Open Events tab
4. Click execute_tool span
5. Examine function_call parameters
6. Find the bug! (wrong parameter type)
```

---

### Production Logging with Plugins

**The Challenge:**
- Web UI great for development
- Can't use web UI in production
- Need automated logging

**The Solution:** Plugins

---

### What are Plugins?

**Plugin** = Custom code that runs at specific agent lifecycle points

**Architecture:**
```
Agent Workflow:
User message → before_agent → Agent thinks → before_tool → Tool runs
→ after_tool → before_model → LLM call → after_model → Response

Plugin hooks into ANY of these points!
```

**Plugins are made of Callbacks:**
- `before_agent_callback` - Before agent starts
- `after_agent_callback` - After agent completes
- `before_tool_callback` - Before tool executes
- `after_tool_callback` - After tool completes
- `before_model_callback` - Before LLM call
- `after_model_callback` - After LLM response
- `on_model_error_callback` - On errors

---

### Built-in LoggingPlugin

**ADK provides standard logging out-of-the-box:**

```python
from google.adk.plugins.logging_plugin import LoggingPlugin

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[LoggingPlugin()]  # Automatic comprehensive logging!
)
```

**What it captures:**
- ✅ User messages and agent responses
- ✅ Timing data for performance
- ✅ LLM requests and responses
- ✅ Tool calls and results
- ✅ Complete execution traces

---

### Custom Plugins - Example

```python
from google.adk.plugins.base_plugin import BasePlugin

class ToolCallTrackerPlugin(BasePlugin):
    def __init__(self):
        super().__init__(name="tool_call_tracker")
        self.total_tool_calls = 0
        self.tool_call_counts = {}

    async def before_tool_callback(
        self, *, agent, callback_context
    ):
        """Runs BEFORE each tool call."""
        self.total_tool_calls += 1
        tool_name = callback_context.tool_name

        if tool_name not in self.tool_call_counts:
            self.tool_call_counts[tool_name] = 0
        self.tool_call_counts[tool_name] += 1

        logging.info(f"Tool #{self.total_tool_calls}: {tool_name}")

    async def after_tool_callback(
        self, *, agent, callback_context
    ):
        """Runs AFTER each tool call."""
        tool_name = callback_context.tool_name
        logging.info(f"Tool {tool_name} finished")

    def get_report(self):
        """Generate summary report."""
        return f"Total calls: {self.total_tool_calls}\n" + \
               f"Breakdown: {self.tool_call_counts}"
```

**Usage:**
```python
tracker = ToolCallTrackerPlugin()

runner = InMemoryRunner(
    agent=my_agent,
    plugins=[
        LoggingPlugin(),  # Standard logs
        tracker,          # Custom tracking
    ]
)

# After execution
print(tracker.get_report())
```

---

### Plugin Use Cases

| Use Case | Plugin Type | What to Track |
|----------|-------------|---------------|
| **Cost Tracking** | Custom | Count expensive API calls |
| **Performance** | Custom | Timing per tool/LLM call |
| **Security Audit** | Custom | Which tools accessed what data |
| **Rate Limiting** | Custom | API calls per minute |
| **Caching** | Custom | Cache LLM responses |
| **Standard Logs** | Built-in | Use LoggingPlugin |

---

### When to Use What?

```
Development Debugging?
  → Use: adk web --log_level DEBUG
  → Why: Interactive, visual, detailed

Production Observability?
  → Use: LoggingPlugin()
  → Why: Automated, standard metrics

Custom Requirements?
  → Use: Build Custom Plugin
  → Why: Specific business logic
```

---

## 📗 Part B: Agent Evaluation

### 🎯 Core Concept: Proactive Quality Assurance

**The Problem:**
```
You built a home automation agent
Week 1: 🚨 "Turned on fireplace instead of lights!"
Week 2: 🚨 "Won't work in guest room!"
Week 3: 🚨 "Gives rude responses!"
```

**Why Standard Testing Fails:**
- Agents are non-deterministic
- Users give unpredictable commands
- Small prompt changes = dramatic behavior shifts

**Solution**: Systematic evaluation of entire decision-making process

---

### Evaluation Workflow

```
1. Create Configuration → Define pass/fail thresholds
2. Create Test Cases    → Sample conversations to compare
3. Run Agent           → Execute with test queries
4. Compare Results     → Measure against expectations
```

---

### ADK Web UI - Interactive Evaluation

**Create Test Cases:**
1. Have conversation in web UI
2. Navigate to **Eval** tab
3. Click **Create Evaluation set**
4. Name it (e.g., "home_automation_tests")
5. Click **Add current session**
6. Name the case (e.g., "basic_device_control")

**Run Evaluation:**
1. Check test cases to run
2. Click **Run Evaluation**
3. Configure metrics (or use defaults)
4. Click **Start**
5. View results (Pass/Fail)

**Analyze Failures:**
1. Click on failed test
2. Hover over "Fail" label
3. See side-by-side comparison:
   - Actual output
   - Expected output
   - Exact differences

---

### Evaluation Metrics

**1. response_match_score**
- Measures text similarity
- Range: 0.0 (completely different) to 1.0 (perfect match)
- Uses text similarity algorithms
- Threshold: Often 0.8 (80% similarity)

**2. tool_trajectory_avg_score**
- Measures correct tool usage
- Range: 0.0 (wrong tools) to 1.0 (perfect)
- Checks: Tool names, parameters, sequence
- Threshold: Often 1.0 (exact match required)

---

### Systematic Evaluation with CLI

**1. Create Evaluation Config** (`test_config.json`):
```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "response_match_score": 0.8
  }
}
```

**2. Create Test Cases** (`integration.evalset.json`):
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
                "args": {
                  "location": "living room",
                  "device_id": "floor lamp",
                  "status": "ON"
                }
              }
            ]
          }
        }
      ]
    }
  ]
}
```

**3. Run Evaluation:**
```bash
adk eval home_automation_agent \
    integration.evalset.json \
    --config_file_path=test_config.json \
    --print_detailed_results
```

**4. Analyze Results:**
```
Test Case: living_room_light_on
  ❌ response_match_score: 0.45/0.80 (FAIL)
  ✅ tool_trajectory_avg_score: 1.0/1.0 (PASS)

Insight: Tool usage is perfect, but communication needs work
```

---

### User Simulation (Dynamic Evaluation)

**The Problem with Static Tests:**
- Fixed, predictable prompts
- Can't test conversation flow
- Miss edge cases in phrasing
- No multi-turn complexity

**The Solution:** User Simulation

---

### What is User Simulation?

**User Simulation** = LLM generates dynamic user prompts during evaluation

**How it works:**
1. Define `ConversationScenario` with goal
2. Provide `conversation_plan` to guide dialogue
3. LLM acts as simulated user
4. Generates realistic, varied prompts
5. Tests agent with unpredictable conversations

---

### Creating a ConversationScenario

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
    max_turns=8  # Conversation length limit
)
```

---

### Conversation Plan Best Practices

**❌ BAD - Too Specific:**
```
Ask: "Turn on the living room light"
```

**✅ GOOD - Natural, Flexible:**
```
You want light in the living room.
Ask naturally and conversationally.
```

---

### Creating User Simulation Evalset

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
    },
    {
      "eval_id": "morning_routine",
      "user_simulation": {
        "goal": "Start morning with right lighting",
        "conversation_plan": "Gradual brightness plan...",
        "max_turns": 6
      }
    }
  ]
}
```

**Run User Simulation:**
```bash
adk eval home_automation_agent \
    user_simulation.evalset.json \
    --config_file_path=test_config.json \
    --print_detailed_results
```

---

### Static vs Dynamic Evaluation

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

### What User Simulation Reveals

**Example Findings:**

**1. Agent Over-Promises:**
- **Problem**: Says "I control ALL smart devices"
- **Reality**: Only has ON/OFF toggle
- **Discovered**: When user asked about coffee maker, HVAC

**2. Missing Validation:**
- **Problem**: Accepts any device name
- **Reality**: Doesn't check if device exists
- **Discovered**: User asked about non-existent devices

**3. Poor Error Handling:**
- **Problem**: Tries to help even when it can't
- **Reality**: Should state limitations clearly
- **Discovered**: Multi-turn conversations showed confusion

**4. Context Blindness:**
- **Problem**: Doesn't track conversation history
- **Reality**: Repeats questions, loses context
- **Discovered**: Sequential commands in same conversation

---

## 🎯 Key Takeaways

### Observability (Part A)

1. **Three Pillars**: Logs (what), Traces (why), Metrics (how well)
2. **Development**: Use `adk web --log_level DEBUG`
3. **Production**: Use `LoggingPlugin()` for standard observability
4. **Custom Needs**: Build custom plugins with callbacks
5. **Lifecycle Hooks**: Plugins run at specific execution points
6. **Multiple Plugins**: Combine LoggingPlugin + Custom plugins

### Evaluation (Part B)

1. **Two Metrics**: response_match_score + tool_trajectory_avg_score
2. **Interactive**: ADK web UI for test case creation
3. **Automated**: `adk eval` CLI for batch testing
4. **Static Tests**: Good for regression, known scenarios
5. **User Simulation**: Good for exploration, edge case discovery
6. **Combined Strategy**: Use both static + dynamic

---

## 🚨 Common Pitfalls

### Observability Mistakes

1. **Not using DEBUG in development** → Can't see detailed LLM prompts
2. **No plugins in production** → Flying blind
3. **Ignoring callback errors** → Silent failures
4. **Not tracking tool calls** → Can't optimize performance
5. **Over-logging** → Too much noise, miss important signals

### Evaluation Mistakes

1. **Only static tests** → Miss edge cases
2. **No tool trajectory checking** → Agent uses wrong tools but passes
3. **Too high response_match threshold** → Minor wording fails tests
4. **Too low response_match threshold** → Accept poor quality responses
5. **No conversation scenarios** → Miss multi-turn issues
6. **Vague conversation plans** → User simulation too unpredictable

---

## 🔥 Vibe Coder Tips

### Observability Tips

1. **Always use DEBUG logs** during development
2. **Web UI first** for interactive debugging
3. **Add LoggingPlugin** before production
4. **Custom plugins** for business-specific metrics
5. **Track tool calls** to find expensive operations
6. **Multiple plugins** don't conflict - use liberally

### Evaluation Tips

1. **Start with web UI** to create initial test cases
2. **Export to evalset** for automation
3. **Set realistic thresholds** (0.8 for response_match)
4. **Require perfect tools** (1.0 for tool_trajectory)
5. **Test static first** then add user simulation
6. **Conversation plans** should guide, not script

### Production Readiness

**Observability Checklist:**
- [ ] LoggingPlugin enabled
- [ ] Custom tracking for critical operations
- [ ] Error callbacks configured
- [ ] Metrics exported to monitoring system
- [ ] Debug logs available but disabled by default

**Evaluation Checklist:**
- [ ] Static tests for known scenarios
- [ ] User simulation for exploration
- [ ] Both metrics configured (response + trajectory)
- [ ] Tests run in CI/CD pipeline
- [ ] Failure analysis automated

---

## 📊 Quick Decision Matrix

### Choosing Observability Approach

| Scenario | Use |
|----------|-----|
| Local development | `adk web --log_level DEBUG` |
| Production standard | `LoggingPlugin()` |
| Custom metrics | Build custom plugin |
| Performance tracking | Custom plugin with timing |
| Cost tracking | Custom plugin counting calls |
| Security audit | Custom plugin logging access |

### Choosing Evaluation Approach

| Scenario | Use |
|----------|-----|
| Regression testing | Static tests |
| Known scenarios | Static tests |
| Quick validation | Static tests |
| Edge case discovery | User simulation |
| Conversation testing | User simulation |
| Realistic behavior | User simulation |
| **Best Practice** | **Both combined** |

---

## 🧪 Code Patterns

### Custom Plugin Pattern

```python
from google.adk.plugins.base_plugin import BasePlugin

class MyCustomPlugin(BasePlugin):
    def __init__(self):
        super().__init__(name="my_plugin")
        self.metrics = {}

    async def before_tool_callback(
        self, *, agent, callback_context
    ):
        # Track tool usage
        tool_name = callback_context.tool_name
        # Your custom logic here

    async def after_agent_callback(
        self, *, agent, callback_context
    ):
        # Clean up, report metrics
        pass
```

### User Simulation Pattern

```python
from google.adk.evaluation import ConversationScenario

scenario = ConversationScenario(
    scenario_id="unique_id",
    goal="What user wants to achieve",
    conversation_plan="""
    You are [user persona].

    Plan:
    1. [First action]
    2. [Second action]
    3. [Challenge the agent]

    Be [tone]. If [condition], [action].
    """,
    max_turns=6
)
```

### Evalset Creation Pattern

```python
import json

evalset = {
    "eval_set_id": "test_suite",
    "eval_cases": [
        {
            "eval_id": "test_1",
            "user_simulation": {
                "goal": scenario.goal,
                "conversation_plan": scenario.conversation_plan,
                "max_turns": scenario.max_turns
            }
        }
    ]
}

with open("evalset.json", "w") as f:
    json.dump(evalset, f, indent=2)
```

---

## 📚 Links & Resources

### Observability
- [ADK Observability Docs](https://google.github.io/adk-docs/observability/logging/)
- [Custom Plugins](https://google.github.io/adk-docs/plugins/)
- [Plugin Callback Hooks](https://google.github.io/adk-docs/plugins/#plugin-callback-hooks)
- [External Integrations](https://google.github.io/adk-docs/observability/cloud-trace/)

### Evaluation
- [ADK Evaluation Overview](https://google.github.io/adk-docs/evaluate/)
- [Evaluation Criteria](https://google.github.io/adk-docs/evaluate/criteria/)
- [User Simulation](https://google.github.io/adk-docs/evaluate/user-sim/)
- [Pytest Evaluation](https://google.github.io/adk-docs/evaluate/#2-pytest-run-tests-programmatically)
- [Advanced Criteria](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/models/determine-eval)

---

## 🎯 Next Steps

1. **Day 5**: Production Deployment & Agent2Agent Protocol
2. **Practice**: Add observability to existing agents
3. **Implement**: User simulation for your chatbot
4. **Integrate**: Add evaluation to CI/CD pipeline
5. **Monitor**: Set up dashboards from plugin metrics

---

## ❓ Questions to Explore

### Observability
1. What metrics matter most for your use case?
2. How to balance detail vs performance in logging?
3. When to use custom plugins vs built-in?
4. How to integrate with existing monitoring?
5. What's the right log retention policy?

### Evaluation
1. What's the right response_match threshold?
2. How many test cases is enough?
3. When to use user simulation vs static?
4. How to automate evaluation in deployment?
5. How to handle non-deterministic responses?

---

## 🏆 Production Checklist

### Before Launch

**Observability:**
- [ ] LoggingPlugin enabled
- [ ] Custom plugins for critical paths
- [ ] Error handling in callbacks
- [ ] Logs exported to monitoring
- [ ] Alerts configured

**Evaluation:**
- [ ] Static tests covering core flows
- [ ] User simulation for edge cases
- [ ] Metrics thresholds defined
- [ ] CI/CD integration
- [ ] Regression detection automated

### After Launch

**Observability:**
- [ ] Monitor dashboards regularly
- [ ] Review error logs daily
- [ ] Track performance trends
- [ ] Adjust log levels as needed
- [ ] Analyze tool usage patterns

**Evaluation:**
- [ ] Run tests on every change
- [ ] Add tests for new bugs found
- [ ] Update user simulation scenarios
- [ ] Review metric trends
- [ ] Iterate on thresholds

---

**🔥 Pro Tip**: Observability catches issues in production. Evaluation prevents them from getting there. Use both! 🎯

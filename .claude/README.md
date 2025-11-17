# Claude Code Configuration

This directory contains Claude Code skills for this repository.

## Available Skills

### `kaggle-ai-agents-intensive`

Expert guide for the Kaggle 5-Day AI Agents Intensive course and VaaS capstone project.

**What it does:**
- Navigates Day 1-5 course materials (agents, tools, memory, observability, A2A)
- Explains VaaS (Vendor-as-a-Service) business model and architecture
- Provides Google ADK code examples and patterns
- Shows cross-framework A2A integration (ADK ↔ CrewAI)
- Guides competition submission structure

**How to use:**

Simply ask questions naturally in Claude Code:
- "How do I create an agent?"
- "What is the A2A protocol?"
- "Explain the VaaS business model"
- "Show me session management examples"
- "How does the capstone work?"

Claude will automatically invoke this skill when you ask about:
- Agent architectures and tools
- Google ADK patterns
- A2A protocol and RemoteA2aAgent
- VaaS capstone concept
- Competition submission
- Day 1-5 course materials

**Skill structure:**
```
.claude/skills/kaggle-ai-agents-intensive/
├── SKILL.md - Main skill definition with comprehensive instructions
├── reference.md - Quick reference for ADK patterns and VaaS architecture
└── examples.md - 10 practical examples with code snippets
```

**Files covered:**
- `day1_intro_to_agents/` - Basic agents and multi-agent systems
- `day2_agent_architecture/` - Custom tools, MCP, long-running operations
- `day3_tools_and_memory/` - Sessions, memory, context management
- `day4_evaluation_and_scaling/` - Observability, logging, debugging
- `day5_capstone/` - **A2A protocol, cross-org communication** (critical for VaaS)
- `competition_description_final.md` - VaaS explanation and submission

## How Skills Work

Skills are specialized knowledge packages that Claude Code uses automatically when relevant to your questions. You don't need to explicitly invoke them—just ask your question naturally, and Claude will use the appropriate skill if needed.

### Testing the Skill

Try these questions to see the skill in action:

1. **Navigation:** "Where can I find A2A examples?"
2. **Concepts:** "What is VaaS and why does it matter?"
3. **Code:** "Show me how to create a RemoteA2aAgent"
4. **Architecture:** "How does the capstone use A2A for security?"
5. **Competition:** "What should I include in my submission?"

### Editing the Skill

The skill files are in `.claude/skills/kaggle-ai-agents-intensive/`:
- Edit `SKILL.md` to change skill behavior and instructions
- Edit `reference.md` to update quick reference content
- Edit `examples.md` to add/modify code examples

Changes take effect immediately for new conversations.

## Repository Context

This repository contains:
1. **Course materials:** Complete Jupyter notebooks for Kaggle 5-Day AI Agents Intensive
2. **Competition submission:** VaaS capstone description (1370 words)
3. **Skills:** This Claude Code skill for navigating and understanding the content

**Related repository:**
- `enterprise-gov-docs-a2a-capstone` - Actual VaaS implementation (multi-agent system, A2A integration, PII filtering)

## Version

**Skill version:** 1.0
**Created:** 2025-01-17
**Last updated:** 2025-01-17

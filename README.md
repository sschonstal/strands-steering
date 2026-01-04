# Strands Agents SDK - Steering Files

> **For Kiro AI Coding Agent**
> 
> These steering files guide Kiro to properly build AI agents using the Strands Agents SDK, following best practices and avoiding common pitfalls.

---

## Quick Reference

| File | Purpose | When to Read |
|------|---------|--------------|
| [`STRANDS_CORE.md`](./STRANDS_CORE.md) | Universal principles, project detection | **ALWAYS read first** |
| [`STRANDS_PYTHON.md`](./STRANDS_PYTHON.md) | Python implementation patterns | Python projects |
| [`STRANDS_TYPESCRIPT.md`](./STRANDS_TYPESCRIPT.md) | TypeScript patterns (preview) | TypeScript/JS projects |
| [`STRANDS_PATTERNS.md`](./STRANDS_PATTERNS.md) | Multi-agent architectures | Complex agent systems |
| [`STRANDS_TOOLS.md`](./STRANDS_TOOLS.md) | Tool creation, MCP, executors | Adding capabilities |
| [`STRANDS_MEMORY.md`](./STRANDS_MEMORY.md) | Sessions, state, context | Persistence needs |
| [`STRANDS_DEPLOYMENT.md`](./STRANDS_DEPLOYMENT.md) | Local → Lambda → AgentCore | Deployment tasks |
| [`STRANDS_TROUBLESHOOTING.md`](./STRANDS_TROUBLESHOOTING.md) | Common errors & fixes | When things break |

---

## Reading Order by Task

### Starting a New Agent Project
1. `STRANDS_CORE.md` - Understand fundamentals
2. `STRANDS_PYTHON.md` or `STRANDS_TYPESCRIPT.md` - Language-specific patterns
3. `STRANDS_TOOLS.md` - Add capabilities

### Building Multi-Agent Systems
1. `STRANDS_CORE.md` - Ensure basics are covered
2. `STRANDS_PATTERNS.md` - Choose the right architecture
3. `STRANDS_MEMORY.md` - State sharing considerations

### Adding Persistence
1. `STRANDS_MEMORY.md` - Session and state management
2. `STRANDS_DEPLOYMENT.md` - Environment considerations

### Preparing for Deployment
1. `STRANDS_DEPLOYMENT.md` - Deployment patterns
2. `STRANDS_TROUBLESHOOTING.md` - Pre-deployment checklist

### Debugging Issues
1. `STRANDS_TROUBLESHOOTING.md` - Start here
2. Relevant topic file for specific area

---

## Key Principles

### 1. Use Strands SDK - Don't Reinvent

```python
# ❌ WRONG - Don't build custom agent loops
class MyAgent:
    def __init__(self):
        self.messages = []
    
    def chat(self, message):
        # Custom message handling...
        pass

# ✅ RIGHT - Use Strands
from strands import Agent

agent = Agent(system_prompt="You are helpful.")
response = agent("Hello")
```

### 2. Default to Bedrock + Claude Sonnet 4

```python
from strands import Agent
from strands.models import BedrockModel

# Default configuration
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

agent = Agent(model=model)
```

### 3. Detect Existing Projects First

Before creating anything, examine:
- `requirements.txt` / `pyproject.toml` / `package.json`
- Existing imports and patterns
- Web framework in use (FastAPI, Flask, Express)
- Async vs sync patterns

### 4. Abstract for Deployment Flexibility

Design components to work across:
- Local development
- AWS Lambda
- Docker containers
- Future AgentCore migration

### 5. Use Appropriate Context Management

```python
# Default: SummarizingConversationManager
from strands.agent.conversation_manager import SummarizingConversationManager

agent = Agent(
    conversation_manager=SummarizingConversationManager(),
)
```

---

## Decision Framework

### When to Ask vs. Assume

**ASSUME** (don't ask):
- Language: Match existing project
- Model: Default to Bedrock Claude Sonnet 4
- Region: Default to us-east-1
- Tool execution: Default to concurrent
- Context management: Default to summarizing

**ASK** when:
- Multiple valid patterns exist (Swarm vs Graph vs Workflow)
- Tool execution order has dependencies
- Performance requirements are critical
- Deployment target significantly affects implementation

---

## Official Resources

- **Documentation**: https://strandsagents.com/latest/documentation/docs/
- **Python SDK**: https://github.com/strands-agents/sdk-python
- **TypeScript SDK**: https://github.com/strands-agents/sdk-typescript
- **Samples**: https://github.com/strands-agents/samples
- **Community Tools**: https://github.com/strands-agents/tools

---

## Version Info

These steering files are current as of:
- Strands Python SDK: 1.20.x
- Strands TypeScript SDK: 0.1.x (Preview)
- Date: January 2026

Always verify against the latest documentation for any breaking changes.

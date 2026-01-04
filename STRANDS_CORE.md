# Strands Agents SDK - Core Principles

> **Purpose**: Steer AI coding agents to properly leverage the Strands Agents SDK for building AI agents. This document establishes foundational rules and recognition patterns.

## RFC 2119 Keywords

- **MUST** / **REQUIRED**: Absolute requirement
- **MUST NOT** / **SHALL NOT**: Absolute prohibition  
- **SHOULD** / **RECOMMENDED**: Strong recommendation with valid exceptions
- **SHOULD NOT**: Strong discouragement with valid exceptions
- **MAY** / **OPTIONAL**: Truly optional

---

## 1. SDK Recognition & Project Detection

### 1.1 Before Starting Any Agent Work

You **MUST** first examine the current working directory for existing project configuration:

```bash
# Check for project indicators
ls -la  # Look for .git, package.json, requirements.txt, pyproject.toml, etc.
```

**If project files exist**, you **MUST**:
1. Identify the primary language (Python vs TypeScript)
2. Check existing dependencies for Strands SDK presence
3. Detect web framework (FastAPI, Flask, Express, etc.)
4. Identify async vs sync patterns in existing code
5. Check for existing agent implementations

### 1.2 Strands SDK Identification

**Python indicators**:
- `strands-agents` in requirements.txt or pyproject.toml
- `from strands import Agent` or `from strands import tool` imports
- `strands_tools` imports

**TypeScript indicators**:
- `@strands-agents/sdk` in package.json
- `import { Agent } from '@strands-agents/sdk'`

### 1.3 Version Management

You **MUST**:
- Install the latest version when starting a new project
- Pin to `major.minor.*` (e.g., `strands-agents>=1.20.0,<1.21.0`)
- Check compatibility with existing project dependencies before upgrading

```bash
# Python - Check latest version
pip index versions strands-agents

# TypeScript - Check latest version  
npm view @strands-agents/sdk version
```

---

## 2. When to Use Strands SDK

### 2.1 You MUST Use Strands When

- Building AI agents that interact with LLMs
- The project requires tool use / function calling
- Multi-turn conversations with context are needed
- Multi-agent orchestration is required
- MCP server integration is needed

### 2.2 You MUST NOT

- Write custom agent loops from scratch when Strands provides the functionality
- Implement your own tool decorator system
- Build custom streaming handlers unless Strands callbacks are insufficient
- Create manual message history management (use ConversationManager)
- Confuse Strands with other frameworks (LangChain, AutoGen, CrewAI)

### 2.3 Framework Distinction

**Strands IS**:
- A model-driven SDK where the LLM plans and executes
- Lightweight with minimal abstractions
- AWS-native with Bedrock as default provider

**Strands IS NOT**:
- LangChain (chain-based, workflow-driven)
- AutoGen (conversation-based multi-agent)
- CrewAI (role-based agents)

If you find yourself writing patterns from these other frameworks, **STOP** and consult Strands documentation.

---

## 3. Default Configuration

### 3.1 Model Provider Defaults

You **SHOULD** default to these unless the project specifies otherwise:

```python
# Python default
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    temperature=0.3,
    max_tokens=4096,
)

agent = Agent(model=model)
```

```typescript
// TypeScript default
import { Agent, BedrockModel } from '@strands-agents/sdk'

const model = new BedrockModel({
    modelId: 'us.anthropic.claude-sonnet-4-20250514-v1:0',
    region: 'us-east-1',
    temperature: 0.3,
    maxTokens: 4096,
})

const agent = new Agent({ model })
```

### 3.2 Fallback Provider Pattern

When Bedrock is unavailable, implement fallback:

```python
import os
from strands import Agent
from strands.models import BedrockModel

def create_model():
    """Create model with fallback support."""
    try:
        # Primary: Bedrock
        return BedrockModel(
            model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
            region_name=os.getenv("AWS_REGION", "us-east-1"),
        )
    except Exception as e:
        # Fallback: Anthropic direct (if API key available)
        if os.getenv("ANTHROPIC_API_KEY"):
            from strands.models import AnthropicModel
            return AnthropicModel(model_id="claude-sonnet-4-20250514")
        raise e
```

---

## 4. Project Structure Recommendations

### 4.1 Recommended Directory Structure

```
project/
├── agents/
│   ├── __init__.py
│   ├── main_agent.py       # Primary agent definition
│   └── sub_agents/         # For multi-agent patterns
├── tools/
│   ├── __init__.py
│   ├── custom_tools.py     # @tool decorated functions
│   └── mcp_config.py       # MCP server configurations
├── config/
│   ├── __init__.py
│   ├── models.py           # Model provider setup
│   └── settings.py         # Environment configuration
├── tests/
│   ├── test_agent.py       # Agent integration tests
│   └── test_tools.py       # Tool unit tests
├── requirements.txt        # or pyproject.toml
└── main.py                 # Entry point
```

### 4.2 Abstraction for Deployment Flexibility

You **MUST** abstract components for easy migration between deployment targets:

```python
# tools/base.py - Abstract tool interface
from abc import ABC, abstractmethod
from strands import tool

class ToolBase(ABC):
    """Base class for deployment-agnostic tools."""
    
    @abstractmethod
    def execute(self, **kwargs):
        """Override in subclass."""
        pass

# This pattern allows tools to work in:
# - Local development
# - AWS Lambda
# - Docker containers
# - AgentCore (future migration)
```

---

## 5. Documentation Access

You **MUST** consult official documentation for detailed implementation:

- **Primary docs**: https://strandsagents.com/latest/documentation/docs/
- **Python API**: https://strandsagents.com/latest/documentation/docs/api-reference/python/
- **TypeScript API**: https://strandsagents.com/latest/documentation/docs/api-reference/typescript/
- **Samples**: https://github.com/strands-agents/samples
- **Community tools**: https://github.com/strands-agents/tools

When in doubt about implementation details, **search the documentation** before making assumptions.

---

## 6. Testing Requirements

You **MUST** always verify agents work with basic tests:

```python
# tests/test_agent.py
import pytest
from agents.main_agent import create_agent

def test_agent_initialization():
    """Verify agent can be created."""
    agent = create_agent()
    assert agent is not None

def test_agent_basic_response():
    """Verify agent can respond to simple query."""
    agent = create_agent()
    response = agent("Hello, are you working?")
    assert response is not None
    assert len(str(response)) > 0

def test_agent_tool_access():
    """Verify agent has access to expected tools."""
    agent = create_agent()
    tool_names = [t.tool_name for t in agent.tool_registry.tools]
    # Verify expected tools are registered
    assert len(tool_names) > 0
```

Run tests before committing:
```bash
pytest tests/ -v
```

---

## 7. Decision Framework

### 7.1 When to Ask vs. Assume

**ASSUME** (don't ask) when:
- Project already uses Python → use Python SDK
- Project already uses TypeScript → use TypeScript SDK
- AWS credentials are configured → use Bedrock
- No web framework detected → create standalone agent
- Simple single-turn task → use basic Agent pattern

**ASK** when:
- Multiple valid architectural patterns exist (Swarm vs Graph)
- Performance requirements are unclear (streaming vs blocking)
- Deployment target affects implementation significantly
- Tool execution order has dependencies (sequential vs concurrent)
- Context management strategy isn't obvious from use case

### 7.2 Pattern Selection Signals

| Signal | Recommended Pattern |
|--------|---------------------|
| Single task, no collaboration | Simple Agent |
| Sub-tasks with clear handoffs | Agents-as-Tools |
| Collaborative exploration | Swarm |
| Deterministic workflow with conditions | Graph |
| Sequential pipeline with dependencies | Workflow |

See `STRANDS_PATTERNS.md` for detailed guidance.

---

## 8. Critical Reminders

1. **Strands is model-driven**: Let the LLM decide how to use tools, don't over-orchestrate
2. **Tools are just functions**: Keep them simple, focused, well-documented
3. **Conversation context matters**: Use appropriate ConversationManager for use case
4. **Test locally first**: Always verify agents work before deployment
5. **Abstract for portability**: Design for eventual AgentCore migration

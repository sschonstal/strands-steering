# Strands Agents SDK - Python Implementation

> **Purpose**: Python-specific patterns, best practices, and framework integration for Strands Agents SDK.

---

## 1. Installation & Setup

### 1.1 Basic Installation

```bash
# Core SDK with Bedrock (default)
pip install strands-agents

# With community tools
pip install strands-agents strands-agents-tools

# With specific model providers
pip install 'strands-agents[anthropic]'    # Anthropic direct
pip install 'strands-agents[openai]'       # OpenAI
pip install 'strands-agents[gemini]'       # Google Gemini
pip install 'strands-agents[litellm]'      # LiteLLM (multi-provider)
pip install 'strands-agents[llamaapi]'     # Meta Llama API
```

### 1.2 Version Pinning

You **MUST** pin versions in production:

```txt
# requirements.txt
strands-agents>=1.20.0,<1.21.0
strands-agents-tools>=0.1.0,<0.2.0
```

```toml
# pyproject.toml
[project]
dependencies = [
    "strands-agents>=1.20.0,<1.21.0",
    "strands-agents-tools>=0.1.0,<0.2.0",
]
```

### 1.3 Environment Configuration

```python
# config/settings.py
import os
from dataclasses import dataclass

@dataclass
class AgentConfig:
    """Configuration for Strands agents."""
    
    # Model settings
    model_id: str = "us.anthropic.claude-sonnet-4-20250514-v1:0"
    aws_region: str = os.getenv("AWS_REGION", "us-east-1")
    temperature: float = 0.3
    max_tokens: int = 4096
    
    # Agent settings
    max_retries: int = 3
    retry_backoff_base: float = 2.0
    
    # Session settings
    session_dir: str = "./sessions"
    
    @classmethod
    def from_env(cls) -> "AgentConfig":
        """Load configuration from environment."""
        return cls(
            model_id=os.getenv("STRANDS_MODEL_ID", cls.model_id),
            aws_region=os.getenv("AWS_REGION", cls.aws_region),
            temperature=float(os.getenv("STRANDS_TEMPERATURE", cls.temperature)),
            max_tokens=int(os.getenv("STRANDS_MAX_TOKENS", cls.max_tokens)),
        )
```

---

## 2. Agent Creation Patterns

### 2.1 Basic Agent

```python
from strands import Agent

# Minimal agent (uses Bedrock defaults)
agent = Agent()
response = agent("What is 2 + 2?")

# With system prompt
agent = Agent(
    system_prompt="You are a helpful coding assistant specializing in Python."
)
```

### 2.2 Agent with Custom Model

```python
from strands import Agent
from strands.models import BedrockModel

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
    temperature=0.3,
    max_tokens=4096,
    streaming=True,  # Enable streaming (default)
)

agent = Agent(
    model=model,
    system_prompt="You are an expert data analyst.",
)
```

### 2.3 Agent with Tools

```python
from strands import Agent, tool
from strands_tools import calculator, current_time, http_request

# Using community tools
agent = Agent(
    tools=[calculator, current_time, http_request],
    system_prompt="You can perform calculations and check the time.",
)

# With custom tools (see STRANDS_TOOLS.md for details)
@tool
def get_weather(location: str) -> str:
    """Get current weather for a location.
    
    Args:
        location: City name or coordinates
        
    Returns:
        Weather information as a string
    """
    # Implementation here
    return f"Weather in {location}: Sunny, 72°F"

agent = Agent(tools=[get_weather, calculator])
```

---

## 3. Framework Integration

### 3.1 FastAPI Integration (Async - RECOMMENDED)

FastAPI's async nature aligns well with Strands streaming:

```python
# main.py
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
from strands import Agent
from strands.models import BedrockModel
import asyncio
import json

app = FastAPI()

# Create agent instance (reuse across requests)
model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None

class ChatResponse(BaseModel):
    response: str
    session_id: str

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    """Synchronous chat endpoint for simple requests."""
    agent = Agent(model=model)
    
    # Use invoke_async for async context
    result = await agent.invoke_async(request.message)
    
    return ChatResponse(
        response=str(result),
        session_id=request.session_id or "default",
    )

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    """Streaming chat endpoint for real-time responses."""
    agent = Agent(model=model)
    
    async def generate():
        async for event in agent.stream_async(request.message):
            # Handle different event types
            if "data" in event:
                chunk = event.get("data", "")
                if chunk:
                    yield f"data: {json.dumps({'chunk': chunk})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(
        generate(),
        media_type="text/event-stream",
    )
```

### 3.2 Flask Integration (Sync)

Flask requires synchronous patterns:

```python
# app.py
from flask import Flask, request, jsonify, Response
from strands import Agent
from strands.models import BedrockModel
import json

app = Flask(__name__)

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

@app.route("/chat", methods=["POST"])
def chat():
    """Synchronous chat endpoint."""
    data = request.get_json()
    message = data.get("message")
    
    if not message:
        return jsonify({"error": "Message required"}), 400
    
    agent = Agent(model=model)
    result = agent(message)  # Synchronous call
    
    return jsonify({"response": str(result)})

@app.route("/chat/stream", methods=["POST"])
def chat_stream():
    """Streaming endpoint using callback handler."""
    data = request.get_json()
    message = data.get("message")
    
    def generate():
        agent = Agent(model=model)
        
        # Use callback handler for streaming in sync context
        def stream_callback(event):
            if "data" in event:
                yield f"data: {json.dumps(event)}\n\n"
        
        # Note: Flask streaming with Strands requires careful handling
        # Consider using Flask-SocketIO for real-time updates
        result = agent(message)
        yield f"data: {json.dumps({'result': str(result)})}\n\n"
    
    return Response(generate(), mimetype="text/event-stream")

if __name__ == "__main__":
    app.run(debug=True)
```

### 3.3 Framework Selection Guidance

| Framework | Async Support | Streaming | Recommended For |
|-----------|---------------|-----------|-----------------|
| FastAPI | Native | Excellent | Production APIs, real-time chat |
| Flask | Limited | Challenging | Simple APIs, prototypes |
| Starlette | Native | Excellent | Lightweight async services |

**You SHOULD**:
- Use FastAPI for new projects requiring streaming
- Use Flask only if the existing project already uses it
- Always use `stream_async()` or `invoke_async()` in async contexts

---

## 4. Async Patterns

### 4.1 Async Iterator Pattern (RECOMMENDED for user interactions)

```python
from strands import Agent

async def process_with_streaming(agent: Agent, prompt: str):
    """Process prompt with real-time streaming updates."""
    full_response = ""
    
    async for event in agent.stream_async(prompt):
        event_type = event.get("type", "")
        
        # Handle text generation
        if event_type == "content_block_delta":
            delta = event.get("delta", {})
            if delta.get("type") == "text_delta":
                text = delta.get("text", "")
                full_response += text
                print(text, end="", flush=True)  # Real-time output
        
        # Handle tool use
        elif event_type == "tool_use":
            tool_name = event.get("name", "unknown")
            print(f"\n[Using tool: {tool_name}]")
        
        # Handle completion
        elif event_type == "message_stop":
            print("\n[Complete]")
    
    return full_response
```

### 4.2 Non-Streaming Async Pattern (for batch processing)

```python
from strands import Agent
import asyncio

async def batch_process(agent: Agent, prompts: list[str]):
    """Process multiple prompts concurrently."""
    tasks = [agent.invoke_async(prompt) for prompt in prompts]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    processed = []
    for prompt, result in zip(prompts, results):
        if isinstance(result, Exception):
            processed.append({"prompt": prompt, "error": str(result)})
        else:
            processed.append({"prompt": prompt, "response": str(result)})
    
    return processed
```

### 4.3 Sync vs Async Decision Guide

**Use Synchronous (`agent(prompt)`)** when:
- Simple CLI tools
- Batch processing with blocking is acceptable
- Flask or sync framework constraints
- Testing and debugging

**Use Async (`agent.invoke_async(prompt)`)** when:
- FastAPI or async framework
- Multiple concurrent agent calls
- Non-streaming responses in async context

**Use Streaming (`agent.stream_async(prompt)`)** when:
- User-facing chat interfaces
- Real-time progress updates needed
- Long-running agent tasks
- User interaction is primary use case

---

## 5. Error Handling & Retry Logic

### 5.1 Retry Pattern with Exponential Backoff

```python
import asyncio
import logging
from strands import Agent
from strands.types.exceptions import (
    ContextWindowOverflowException,
    EventLoopException,
)

logger = logging.getLogger(__name__)

async def invoke_with_retry(
    agent: Agent,
    prompt: str,
    max_retries: int = 3,
    base_delay: float = 2.0,
    max_delay: float = 60.0,
) -> str:
    """Invoke agent with exponential backoff retry.
    
    Args:
        agent: Strands Agent instance
        prompt: User prompt
        max_retries: Maximum retry attempts
        base_delay: Initial delay in seconds
        max_delay: Maximum delay cap
        
    Returns:
        Agent response as string
        
    Raises:
        Exception: If all retries exhausted
    """
    last_exception = None
    
    for attempt in range(max_retries + 1):
        try:
            result = await agent.invoke_async(prompt)
            return str(result)
            
        except ContextWindowOverflowException as e:
            # Don't retry context overflow - need to reduce context
            logger.error(f"Context window overflow: {e}")
            raise
            
        except Exception as e:
            last_exception = e
            
            if attempt < max_retries:
                # Calculate delay with exponential backoff
                delay = min(base_delay * (2 ** attempt), max_delay)
                logger.warning(
                    f"Attempt {attempt + 1} failed: {e}. "
                    f"Retrying in {delay:.1f}s..."
                )
                await asyncio.sleep(delay)
            else:
                logger.error(f"All {max_retries + 1} attempts failed")
                raise last_exception
    
    raise last_exception  # Should never reach here
```

### 5.2 Sync Retry Pattern

```python
import time
import logging

logger = logging.getLogger(__name__)

def invoke_with_retry_sync(
    agent: Agent,
    prompt: str,
    max_retries: int = 3,
    base_delay: float = 2.0,
) -> str:
    """Synchronous retry pattern."""
    for attempt in range(max_retries + 1):
        try:
            return str(agent(prompt))
        except Exception as e:
            if attempt < max_retries:
                delay = base_delay * (2 ** attempt)
                logger.warning(f"Retry {attempt + 1}/{max_retries}: {e}")
                time.sleep(delay)
            else:
                raise
```

---

## 6. Logging & Debugging

### 6.1 Enable Strands Debug Logging

```python
import logging

# Enable all Strands logs
logging.getLogger("strands").setLevel(logging.DEBUG)

# Or specific modules
logging.getLogger("strands.agent").setLevel(logging.DEBUG)
logging.getLogger("strands.tools").setLevel(logging.DEBUG)
logging.getLogger("strands.models.bedrock").setLevel(logging.DEBUG)
logging.getLogger("strands.multiagent").setLevel(logging.DEBUG)

# Configure output format
logging.basicConfig(
    format="%(levelname)s | %(name)s | %(message)s",
    level=logging.INFO,
)
```

### 6.2 Agent Inspection

```python
def inspect_agent(agent: Agent):
    """Debug helper to inspect agent configuration."""
    print(f"Agent Name: {agent.name}")
    print(f"Agent ID: {agent.id}")
    print(f"Model: {agent.model}")
    
    # List registered tools
    print("\nRegistered Tools:")
    for tool in agent.tool_registry.tools:
        print(f"  - {tool.tool_name}: {tool.tool_spec.get('description', 'No description')[:50]}...")
    
    # Check conversation state
    print(f"\nMessage Count: {len(agent.messages)}")
    print(f"Agent State: {agent.state}")
```

---

## 7. Type Hints & Documentation

You **SHOULD** always include type hints for better tooling and documentation:

```python
from strands import Agent, tool
from strands.types.tools import ToolResult, ToolContext
from typing import Optional

@tool
def search_database(
    query: str,
    limit: int = 10,
    include_metadata: bool = False,
) -> str:
    """Search the database for relevant records.
    
    This tool searches the internal database using semantic search
    and returns matching records formatted as a readable string.
    
    Args:
        query: The search query string
        limit: Maximum number of results to return (default: 10)
        include_metadata: Whether to include record metadata (default: False)
        
    Returns:
        A formatted string containing search results, or an error message
        if no results are found.
        
    Example:
        search_database("customer orders from last week", limit=5)
    """
    # Implementation
    results = perform_search(query, limit)
    return format_results(results, include_metadata)
```

---

## 8. Production Checklist

Before deploying a Python Strands agent:

- [ ] Version pinned in requirements.txt/pyproject.toml
- [ ] Environment variables documented and validated
- [ ] Retry logic implemented with exponential backoff
- [ ] Error handling covers common exceptions
- [ ] Logging configured appropriately
- [ ] Tests pass (`pytest tests/ -v`)
- [ ] Memory/session management configured
- [ ] Tools abstracted for deployment flexibility
- [ ] Async patterns used where appropriate

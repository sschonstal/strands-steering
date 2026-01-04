# Strands Agents SDK - Troubleshooting

> **Purpose**: Common issues, gotchas, and debugging strategies for Strands Agents SDK.

---

## 1. Quick Diagnostic Checklist

When an agent isn't working, check these in order:

1. **AWS Credentials**: Are credentials configured and valid?
2. **Model Access**: Is the Bedrock model enabled in your region?
3. **SDK Version**: Is the SDK up to date?
4. **Dependencies**: Are all required packages installed?
5. **Tool Definitions**: Do tools have proper docstrings and type hints?
6. **Context Size**: Is the conversation exceeding the context window?

---

## 2. Common Errors & Solutions

### 2.1 Authentication Errors

#### Error: `NoCredentialsError`
```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

**Cause**: AWS credentials not configured

**Solutions**:
```bash
# Option 1: Environment variables
export AWS_ACCESS_KEY_ID=your-key
export AWS_SECRET_ACCESS_KEY=your-secret
export AWS_REGION=us-east-1

# Option 2: AWS CLI configuration
aws configure

# Option 3: Bedrock API key (development only)
export AWS_BEARER_TOKEN_BEDROCK=your-bedrock-key
```

#### Error: `AccessDeniedException`
```
botocore.exceptions.ClientError: An error occurred (AccessDeniedException)
```

**Cause**: IAM permissions missing or model not enabled

**Solutions**:
1. Enable model access in Bedrock console
2. Check IAM policy includes:
```json
{
    "Effect": "Allow",
    "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
    ],
    "Resource": "*"
}
```

### 2.2 Model Errors

#### Error: `ValidationException - Model doesn't support tool use`
```
botocore.errorfactory.ValidationException: This model doesn't support tool use in streaming mode
```

**Cause**: Using a model that doesn't support tool use with streaming

**Solutions**:
```python
# Option 1: Use a supported model
from strands.models import BedrockModel

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",  # Supports tool use
    region_name="us-east-1",
)

# Option 2: Disable streaming for unsupported models
model = BedrockModel(
    model_id="your-model-id",
    streaming=False,  # Disable streaming
)
```

#### Error: `ThrottlingException`
```
botocore.exceptions.EventStreamError: An error occurred (ThrottlingException)
```

**Cause**: Rate limited by Bedrock

**Solutions**:
```python
# Implement retry with backoff
import asyncio

async def invoke_with_retry(agent, prompt, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await agent.invoke_async(prompt)
        except Exception as e:
            if "throttling" in str(e).lower():
                delay = 2 ** attempt  # Exponential backoff
                await asyncio.sleep(delay)
            else:
                raise
    raise Exception("Max retries exceeded")
```

### 2.3 Context Window Errors

#### Error: `ContextWindowOverflowException`
```
strands.types.exceptions.ContextWindowOverflowException
```

**Cause**: Conversation exceeds model's context limit

**Solutions**:
```python
# Solution 1: Use SummarizingConversationManager
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager

agent = Agent(
    conversation_manager=SummarizingConversationManager(
        max_messages=20,
        preserve_recent=5,
    ),
)

# Solution 2: Clear messages periodically
agent.messages.clear()

# Solution 3: Start new conversation
agent = Agent()  # Fresh instance
```

### 2.4 Tool Errors

#### Error: `Tool not found in registry`
```
tool_name=<my_tool>, available_tools=<[]> | tool not found in registry
```

**Cause**: Tool not properly registered with agent

**Solutions**:
```python
# Ensure tool is decorated correctly
from strands import tool

@tool  # Don't forget the decorator!
def my_tool(param: str) -> str:
    """Tool description here."""  # Don't forget the docstring!
    return f"Result: {param}"

# Ensure tool is passed to agent
agent = Agent(tools=[my_tool])  # Include in tools list
```

#### Error: Tool not being called by agent

**Cause**: Poor tool documentation - LLM doesn't understand when to use it

**Solutions**:
```python
# ❌ Bad - vague description
@tool
def process(data: str) -> str:
    """Process data."""
    pass

# ✅ Good - clear, specific description
@tool
def calculate_shipping_cost(weight_kg: float, destination: str) -> str:
    """Calculate international shipping cost for a package.
    
    Use this tool when the user asks about:
    - Shipping costs or delivery fees
    - How much to send a package
    - International shipping prices
    
    Args:
        weight_kg: Package weight in kilograms (positive number)
        destination: Two-letter country code (e.g., 'US', 'GB', 'JP')
        
    Returns:
        Shipping cost in USD with delivery estimate
    """
    pass
```

### 2.5 MCP Errors

#### Error: `Connection to MCP server was closed`
```
RuntimeError: Connection to the MCP server was closed
```

**Cause**: MCP server crashed or connection lost

**Solutions**:
```python
# Use context manager properly
from strands.tools.mcp import MCPClient

mcp_client = MCPClient(lambda: stdio_client(...))

# ✅ Correct - use context manager
with mcp_client:
    tools = mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    response = agent("Do something")

# ❌ Wrong - connection may close unexpectedly
tools = mcp_client.list_tools_sync()  # No context manager
```

#### Error: MCP tool call times out

**Cause**: MCP server taking too long to respond

**Solutions**:
```python
# Implement timeout handling
from strands.tools.mcp import MCPClient
import asyncio

async def call_with_timeout(agent, prompt, timeout=30):
    try:
        return await asyncio.wait_for(
            agent.invoke_async(prompt),
            timeout=timeout
        )
    except asyncio.TimeoutError:
        return "Request timed out. Please try again."
```

### 2.6 Multi-Agent Errors

#### Error: Session manager in multi-agent system
```
Exception: Cannot use single agent with session manager in multi-agent system
```

**Cause**: Individual agents in Swarm/Graph have session managers

**Solutions**:
```python
# ❌ Wrong
agent1 = Agent(session_manager=FileSessionManager("id1"))
agent2 = Agent(session_manager=FileSessionManager("id2"))
swarm = Swarm([agent1, agent2])  # Will fail!

# ✅ Correct - only orchestrator has session manager
agent1 = Agent(name="agent1")  # No session manager
agent2 = Agent(name="agent2")  # No session manager
swarm = Swarm(
    agents=[agent1, agent2],
    session_manager=FileSessionManager("swarm-session"),
)
```

#### Error: Infinite handoff loop in Swarm

**Cause**: Agents keep handing off to each other

**Solutions**:
```python
# Configure handoff limits
swarm = Swarm(
    agents=[agent1, agent2, agent3],
    max_handoffs=10,  # Limit total handoffs
    repetitive_handoff_detection_window=5,
    repetitive_handoff_min_unique_agents=2,
)
```

---

## 3. Debugging Strategies

### 3.1 Enable Debug Logging

```python
import logging

# Enable all Strands logs
logging.getLogger("strands").setLevel(logging.DEBUG)

# Specific modules
logging.getLogger("strands.agent").setLevel(logging.DEBUG)
logging.getLogger("strands.tools").setLevel(logging.DEBUG)
logging.getLogger("strands.models.bedrock").setLevel(logging.DEBUG)
logging.getLogger("strands.multiagent").setLevel(logging.DEBUG)

# Configure output
logging.basicConfig(
    format="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
    level=logging.DEBUG,
)
```

### 3.2 Inspect Agent State

```python
def debug_agent(agent):
    """Print agent debugging information."""
    print("=" * 50)
    print(f"Agent: {agent.name} (ID: {agent.id})")
    print(f"Model: {agent.model}")
    print("-" * 50)
    
    # Tools
    print("Tools:")
    for tool in agent.tool_registry.tools:
        print(f"  - {tool.tool_name}")
    
    # Messages
    print(f"\nMessages ({len(agent.messages)}):")
    for i, msg in enumerate(agent.messages[-5:]):  # Last 5
        role = msg.get("role", "unknown")
        content = str(msg.get("content", ""))[:80]
        print(f"  [{i}] {role}: {content}...")
    
    # State
    print(f"\nState: {dict(agent.state._state)}")
    print("=" * 50)
```

### 3.3 Test Tools Independently

```python
# Test tool outside of agent
from tools.my_tools import my_custom_tool

# Direct invocation
result = my_custom_tool("test input")
print(f"Tool result: {result}")

# Verify tool spec
print(f"Tool spec: {my_custom_tool.tool_spec}")
```

### 3.4 Trace Agent Execution

```python
from strands import Agent

agent = Agent()

# Use streaming to see step-by-step execution
for event in agent.stream("Do something complex"):
    event_type = event.get("type", "unknown")
    
    if event_type == "tool_use":
        print(f"🔧 Calling tool: {event.get('name')}")
        print(f"   Input: {event.get('input')}")
    
    elif event_type == "tool_result":
        print(f"📤 Tool result: {event.get('content', '')[:100]}...")
    
    elif event_type == "content_block_delta":
        delta = event.get("delta", {})
        if delta.get("type") == "text_delta":
            print(delta.get("text", ""), end="")
```

---

## 4. Performance Issues

### 4.1 Slow Response Times

**Causes & Solutions**:

| Cause | Solution |
|-------|----------|
| Large context | Use SummarizingConversationManager |
| Many tools | Reduce tool count or use semantic tool selection |
| Cold start (Lambda) | Increase memory, use provisioned concurrency |
| Model latency | Use faster model (e.g., Nova Micro) |
| Sequential tools | Use ConcurrentToolExecutor |

### 4.2 Memory Issues (Lambda)

```python
# Reduce memory footprint
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

# Limit conversation history
agent = Agent(
    conversation_manager=SlidingWindowConversationManager(window_size=5),
)

# Clear agent after use
del agent

# Or reuse single instance
AGENT = None

def get_agent():
    global AGENT
    if AGENT is None:
        AGENT = Agent()
    return AGENT
```

---

## 5. TypeScript-Specific Issues

### 5.1 Import Errors

```typescript
// ❌ Wrong - CommonJS style
const { Agent } = require('@strands-agents/sdk')

// ✅ Correct - ESM style
import { Agent } from '@strands-agents/sdk'
```

**Ensure package.json has**:
```json
{
  "type": "module"
}
```

### 5.2 Async/Await Issues

```typescript
// ❌ Wrong - not awaiting
const response = agent.invoke("prompt")  // Returns Promise!

// ✅ Correct
const response = await agent.invoke("prompt")

// Or in non-async context
agent.invoke("prompt").then(response => {
    console.log(response)
})
```

### 5.3 Zod Schema Issues

```typescript
import { z } from 'zod'
import { tool } from '@strands-agents/sdk'

// ❌ Wrong - missing descriptions
const myTool = tool({
    name: 'my_tool',
    inputSchema: z.object({
        param: z.string(),  // No description!
    }),
    callback: (input) => "result",
})

// ✅ Correct - with descriptions
const myTool = tool({
    name: 'my_tool',
    description: 'Tool description here',  // Required!
    inputSchema: z.object({
        param: z.string().describe('What this parameter does'),
    }),
    callback: (input) => "result",
})
```

---

## 6. Version Compatibility

### 6.1 Check SDK Version

```bash
# Python
pip show strands-agents

# TypeScript
npm list @strands-agents/sdk
```

### 6.2 Common Version Issues

| Issue | Min Version | Notes |
|-------|-------------|-------|
| Hooks system | 1.10.0 | Experimental in earlier versions |
| Multi-agent | 1.0.0 | Swarm, Graph introduced |
| Session managers | 0.8.0 | FileSessionManager, S3SessionManager |
| Streaming events | 0.5.0 | stream_async() method |

### 6.3 Upgrade Safely

```bash
# Check current version
pip show strands-agents

# Upgrade to latest
pip install strands-agents --upgrade

# Pin to specific version
pip install strands-agents==1.20.0
```

---

## 7. Environment-Specific Issues

### 7.1 Lambda Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Timeout | Agent takes too long | Increase timeout (max 900s) |
| Memory error | Not enough RAM | Increase to 1024MB+ |
| Cold start slow | Package size | Use Lambda layers |
| /tmp full | Session files | Use S3SessionManager |

### 7.2 Docker Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| No AWS creds | Missing env vars | Pass AWS credentials |
| Permission denied | Running as root | Use non-root user |
| Package errors | Missing system deps | Add to Dockerfile |

---

## 8. Getting Help

### 8.1 Before Asking for Help

1. **Check logs** with DEBUG level enabled
2. **Test tools** independently
3. **Verify credentials** are working
4. **Check SDK version** is up to date
5. **Search existing issues** on GitHub

### 8.2 Resources

- **Documentation**: https://strandsagents.com/latest/documentation/docs/
- **GitHub Issues**: https://github.com/strands-agents/sdk-python/issues
- **Samples**: https://github.com/strands-agents/samples
- **Community Tools**: https://github.com/strands-agents/tools

### 8.3 Reporting Issues

When reporting issues, include:
- SDK version (`pip show strands-agents`)
- Python/Node.js version
- Full error traceback
- Minimal reproduction code
- Environment (Lambda, Docker, local)

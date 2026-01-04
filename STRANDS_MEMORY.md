# Strands Agents SDK - Memory & Session Management

> **Purpose**: Guide implementation of conversation context management, session persistence, and agent state for Strands agents.

---

## 1. Context Management Overview

Strands provides multiple mechanisms for managing agent context:

| Mechanism | Purpose | Persistence | Scope |
|-----------|---------|-------------|-------|
| **Conversation Manager** | Message history strategy | In-memory | Single conversation |
| **Session Manager** | Persist state across restarts | Disk/S3/Custom | Cross-session |
| **Agent State** | Key-value storage | Via Session Manager | Agent lifecycle |

---

## 2. Conversation Managers

### 2.1 Default: Sliding Window

Keeps last N messages in context:

```python
from strands import Agent
from strands.agent.conversation_manager import SlidingWindowConversationManager

agent = Agent(
    conversation_manager=SlidingWindowConversationManager(
        window_size=10,  # Keep last 10 messages
    ),
)
```

**Use When**:
- Short, focused conversations
- Context window is limited
- Recent messages are most relevant

### 2.2 Summarizing (RECOMMENDED DEFAULT)

Summarizes old messages to retain context while reducing tokens:

```python
from strands import Agent
from strands.agent.conversation_manager import SummarizingConversationManager

agent = Agent(
    conversation_manager=SummarizingConversationManager(
        max_messages=20,           # Summarize when exceeding this
        summary_ratio=0.3,         # Keep 30% of context as summary
        preserve_recent=5,         # Always keep last 5 messages intact
    ),
)
```

**Use When**:
- Long conversations
- Historical context matters
- Need to balance context and token usage
- **DEFAULT: Use this unless you have a specific reason not to**

### 2.3 Null (No Management)

No automatic context management:

```python
from strands import Agent
from strands.agent.conversation_manager import NullConversationManager

agent = Agent(
    conversation_manager=NullConversationManager(),
)
```

**Use When**:
- Manual message management needed
- Custom context strategies
- Single-turn interactions only

### 2.4 Selection Decision Guide

```
                    START
                      │
                      ▼
         ┌─────────────────────────┐
         │  Single-turn only?      │─── YES ───▶ NullConversationManager
         └─────────────────────────┘
                      │ NO
                      ▼
         ┌─────────────────────────┐
         │  Long conversations?    │─── NO ────▶ SlidingWindowConversationManager
         │  (>10 turns)            │
         └─────────────────────────┘
                      │ YES
                      ▼
         ┌─────────────────────────┐
         │  Historical context     │─── YES ───▶ SummarizingConversationManager
         │  important?             │                 (RECOMMENDED)
         └─────────────────────────┘
                      │ NO
                      ▼
              SlidingWindowConversationManager
```

### 2.5 When to ASK for Clarification

**ASK** if:
- Conversation length is unclear
- Token budget constraints aren't mentioned
- Historical vs. recent context importance is ambiguous

**Example clarification**:
> "For context management, I need to understand your use case:
> 1. **Short conversations** (< 10 turns) - Simple sliding window
> 2. **Long conversations** with important history - Summarizing (recommended)
> 3. **Single-turn only** - No context needed
>
> Which best matches your needs?"

---

## 3. Session Managers

### 3.1 FileSessionManager (Development)

```python
from strands import Agent
from strands.session.file_session_manager import FileSessionManager

# Create session manager
session_manager = FileSessionManager(
    session_id="user-session-123",
    base_dir="./sessions",  # Optional: defaults to ./sessions
)

# Create agent with session
agent = Agent(
    session_manager=session_manager,
    system_prompt="You are a helpful assistant.",
)

# Messages and state automatically persist
agent("My name is Alice")
agent("What's my name?")  # Will remember Alice

# Later: restore session
restored_manager = FileSessionManager(
    session_id="user-session-123",
    base_dir="./sessions",
)
restored_agent = Agent(session_manager=restored_manager)
# Conversation history is restored
```

**Directory Structure**:
```
sessions/
└── session_user-session-123/
    ├── session.json         # Session metadata
    └── agents/
        └── agent_default/
            ├── agent.json   # Agent metadata and state
            └── messages/
                ├── message_0.json
                └── message_1.json
```

### 3.2 S3SessionManager (Production)

```python
from strands import Agent
from strands.session.s3_session_manager import S3SessionManager
import boto3

# Optional: custom boto3 session
boto_session = boto3.Session(region_name="us-east-1")

# Create S3 session manager
session_manager = S3SessionManager(
    session_id="user-456",
    bucket="my-agent-sessions",
    prefix="production/",  # Optional: organize by environment
    boto_session=boto_session,  # Optional: custom session
)

agent = Agent(
    session_manager=session_manager,
    system_prompt="You are a helpful assistant.",
)
```

**Required IAM Permissions**:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::my-agent-sessions/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::my-agent-sessions"
        }
    ]
}
```

### 3.3 Session Manager Selection

| Environment | Recommended Manager | Notes |
|-------------|---------------------|-------|
| Local Development | FileSessionManager | Easy debugging |
| Lambda/Docker | S3SessionManager | Stateless compute |
| EC2/ECS | Either | Based on persistence needs |
| AgentCore (future) | AgentCoreMemorySessionManager | Native integration |

### 3.4 Multi-Agent Session Constraint

⚠️ **IMPORTANT**: Session managers have specific rules for multi-agent systems:

```python
# ❌ WRONG - Throws exception
from strands.multiagent import Swarm
from strands.session.file_session_manager import FileSessionManager

agent_with_session = Agent(session_manager=FileSessionManager("id"))
swarm = Swarm([agent_with_session])  # Will fail!

# ✅ CORRECT - Only orchestrator has session manager
researcher = Agent(name="researcher")  # No session manager
analyst = Agent(name="analyst")        # No session manager

swarm = Swarm(
    agents=[researcher, analyst],
    session_manager=FileSessionManager("swarm-session"),  # Orchestrator only
)
```

---

## 4. Agent State

### 4.1 Using Agent State

Agent state is a key-value store that persists with sessions:

```python
from strands import Agent
from strands.session.file_session_manager import FileSessionManager

agent = Agent(
    session_manager=FileSessionManager("my-session"),
    state={
        "user_preferences": {},
        "interaction_count": 0,
    }
)

# Access state
count = agent.state.get("interaction_count", 0)
agent.state.set("interaction_count", count + 1)

# Check if key exists
if agent.state.get("user_name") is None:
    agent.state.set("user_name", "Unknown")
```

### 4.2 State in Tools

Access state from within tools:

```python
from strands import tool
from strands.types.tools import ToolContext

@tool(context=True)
def remember_preference(key: str, value: str, tool_context: ToolContext) -> str:
    """Store a user preference for later use.
    
    Args:
        key: Preference name
        value: Preference value
        
    Returns:
        Confirmation message
    """
    # Access agent state through tool context
    agent = tool_context.agent
    
    prefs = agent.state.get("user_preferences", {})
    prefs[key] = value
    agent.state.set("user_preferences", prefs)
    
    return f"Remembered: {key} = {value}"

@tool(context=True)
def recall_preference(key: str, tool_context: ToolContext) -> str:
    """Recall a stored user preference.
    
    Args:
        key: Preference name to recall
        
    Returns:
        The stored preference value
    """
    agent = tool_context.agent
    prefs = agent.state.get("user_preferences", {})
    
    if key in prefs:
        return f"{key}: {prefs[key]}"
    return f"No preference stored for '{key}'"
```

### 4.3 State vs. Conversation Context

| Use Case | Use State | Use Conversation |
|----------|-----------|------------------|
| User preferences | ✅ | ❌ |
| Counters/metrics | ✅ | ❌ |
| Configuration | ✅ | ❌ |
| Recent discussion | ❌ | ✅ |
| Task context | ❌ | ✅ |
| Multi-turn reasoning | ❌ | ✅ |

---

## 5. Invocation State (Runtime Context)

### 5.1 Passing Runtime Context

Invocation state passes context for a single invocation:

```python
from strands import Agent

agent = Agent()

# Pass context for this invocation
result = agent(
    "Process the order",
    invocation_state={
        "user_id": "user-123",
        "request_id": "req-abc",
        "environment": "production",
        "feature_flags": {"new_pricing": True},
    }
)
```

### 5.2 Accessing in Tools

```python
from strands import tool
from strands.types.tools import ToolContext

@tool(context=True)
def process_with_context(data: str, tool_context: ToolContext) -> str:
    """Process data using invocation context.
    
    Args:
        data: Data to process
        
    Returns:
        Processing result
    """
    user_id = tool_context.invocation_state.get("user_id")
    env = tool_context.invocation_state.get("environment", "development")
    
    # Use context for processing
    if env == "production":
        # Production logic
        pass
    
    return f"Processed for user {user_id} in {env}"
```

### 5.3 In Multi-Agent Systems

```python
from strands.multiagent import Swarm

swarm = Swarm([agent1, agent2, agent3])

# Context shared across all agents in swarm
result = swarm(
    "Research and analyze topic",
    invocation_state={
        "project_id": "project-xyz",
        "shared_memory_key": "research-results",
    }
)
```

---

## 6. Best Practices

### 6.1 Session ID Strategy

```python
import uuid
from datetime import datetime

def generate_session_id(user_id: str) -> str:
    """Generate deterministic session ID for user."""
    # Option 1: User-based (conversation persists)
    return f"user-{user_id}"
    
def generate_unique_session_id(user_id: str) -> str:
    """Generate unique session ID per conversation."""
    # Option 2: Unique per conversation
    return f"user-{user_id}-{uuid.uuid4().hex[:8]}"
    
def generate_daily_session_id(user_id: str) -> str:
    """Generate daily session ID (reset each day)."""
    # Option 3: Daily reset
    date = datetime.now().strftime("%Y-%m-%d")
    return f"user-{user_id}-{date}"
```

### 6.2 Environment-Based Session Manager

```python
import os
from strands import Agent
from strands.session.file_session_manager import FileSessionManager
from strands.session.s3_session_manager import S3SessionManager

def create_session_manager(session_id: str):
    """Create appropriate session manager for environment."""
    
    # Production: Use S3
    if os.environ.get("AWS_LAMBDA_FUNCTION_NAME") or \
       os.environ.get("ENVIRONMENT") == "production":
        return S3SessionManager(
            session_id=session_id,
            bucket=os.environ["SESSION_BUCKET"],
            prefix=os.environ.get("SESSION_PREFIX", ""),
        )
    
    # Development: Use file system
    return FileSessionManager(
        session_id=session_id,
        base_dir="./sessions",
    )
```

### 6.3 Graceful Session Handling

```python
from strands import Agent
from strands.session.file_session_manager import FileSessionManager
import logging

logger = logging.getLogger(__name__)

def create_agent_with_session(session_id: str) -> Agent:
    """Create agent with graceful session handling."""
    try:
        session_manager = FileSessionManager(session_id=session_id)
        agent = Agent(session_manager=session_manager)
        logger.info(f"Restored session: {session_id}")
        return agent
    except Exception as e:
        logger.warning(f"Could not restore session {session_id}: {e}")
        # Create new session
        session_manager = FileSessionManager(session_id=session_id)
        return Agent(session_manager=session_manager)
```

---

## 7. Memory Patterns for Deployment

### 7.1 Abstraction for AgentCore Migration

Design for easy future migration to AgentCore:

```python
# memory/interface.py
from abc import ABC, abstractmethod
from typing import Any, Optional

class MemoryInterface(ABC):
    """Abstract memory interface for deployment flexibility."""
    
    @abstractmethod
    def store(self, key: str, value: Any) -> None:
        """Store a value."""
        pass
    
    @abstractmethod
    def retrieve(self, key: str) -> Optional[Any]:
        """Retrieve a value."""
        pass
    
    @abstractmethod
    def delete(self, key: str) -> None:
        """Delete a value."""
        pass

# memory/local.py
class LocalMemory(MemoryInterface):
    """File-based memory for local development."""
    
    def __init__(self, base_dir: str = "./memory"):
        self.base_dir = base_dir
        
    def store(self, key: str, value: Any) -> None:
        # Implementation
        pass

# memory/s3.py
class S3Memory(MemoryInterface):
    """S3-based memory for Lambda/Docker."""
    
    def __init__(self, bucket: str, prefix: str = ""):
        self.bucket = bucket
        self.prefix = prefix
        
    def store(self, key: str, value: Any) -> None:
        # Implementation
        pass

# memory/agentcore.py (future)
class AgentCoreMemory(MemoryInterface):
    """AgentCore memory integration (future migration path)."""
    
    def __init__(self, memory_id: str, region: str = "us-east-1"):
        self.memory_id = memory_id
        self.region = region
        
    def store(self, key: str, value: Any) -> None:
        # AgentCore implementation
        pass
```

### 7.2 Factory Pattern for Memory

```python
# memory/factory.py
import os
from .interface import MemoryInterface
from .local import LocalMemory
from .s3 import S3Memory

def create_memory() -> MemoryInterface:
    """Create appropriate memory implementation for environment."""
    
    if os.environ.get("AGENTCORE_RUNTIME"):
        # Future: AgentCore memory
        from .agentcore import AgentCoreMemory
        return AgentCoreMemory(
            memory_id=os.environ["AGENTCORE_MEMORY_ID"],
        )
    
    if os.environ.get("MEMORY_BUCKET"):
        return S3Memory(
            bucket=os.environ["MEMORY_BUCKET"],
            prefix=os.environ.get("MEMORY_PREFIX", ""),
        )
    
    return LocalMemory(base_dir="./memory")
```

---

## 8. Troubleshooting

### 8.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Session not persisting | Missing session_manager | Add FileSessionManager or S3SessionManager |
| Context growing too large | No conversation manager | Add SummarizingConversationManager |
| State lost between invocations | No session manager | Add session manager |
| Multi-agent session error | Agent in Swarm/Graph has session manager | Remove session manager from individual agents |

### 8.2 Debugging Session State

```python
def debug_session(agent: Agent):
    """Print session debugging information."""
    print(f"Agent ID: {agent.id}")
    print(f"Message count: {len(agent.messages)}")
    print(f"State keys: {list(agent.state._state.keys())}")
    
    # Print recent messages
    print("\nRecent messages:")
    for msg in agent.messages[-3:]:
        role = msg.get("role", "unknown")
        content = str(msg.get("content", ""))[:100]
        print(f"  [{role}]: {content}...")
```

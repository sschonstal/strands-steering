# Strands Agents SDK - Tools

> **Purpose**: Guide creation and configuration of tools for Strands agents, including custom tools, MCP integration, and execution strategies.

---

## 1. Tool Creation Fundamentals

### 1.1 The @tool Decorator (Python)

You **MUST** use the `@tool` decorator for custom Python tools:

```python
from strands import tool

@tool
def my_tool(parameter: str) -> str:
    """Brief description used by the LLM to understand when to use this tool.
    
    Args:
        parameter: Description of what this parameter does
        
    Returns:
        Description of what this tool returns
    """
    # Implementation
    return f"Result: {parameter}"
```

### 1.2 Key Requirements

You **MUST** provide:
- **Type hints** on all parameters and return type
- **Docstring** with clear description (LLM reads this!)
- **Parameter descriptions** in docstring Args section

You **SHOULD**:
- Keep tools focused on single responsibilities
- Return strings or simple types (auto-converted to ToolResult)
- Handle errors gracefully with informative messages

You **MUST NOT**:
- Create tools without docstrings (LLM won't know when to use them)
- Rely on complex nested return types
- Let exceptions propagate without handling

---

## 2. Custom Tool Patterns

### 2.1 Simple Tool

```python
from strands import tool

@tool
def calculate_discount(price: float, discount_percent: float) -> str:
    """Calculate the discounted price for an item.
    
    Args:
        price: Original price in dollars
        discount_percent: Discount percentage (0-100)
        
    Returns:
        String with original price, discount amount, and final price
    """
    if discount_percent < 0 or discount_percent > 100:
        return "Error: Discount must be between 0 and 100"
    
    discount_amount = price * (discount_percent / 100)
    final_price = price - discount_amount
    
    return f"Original: ${price:.2f}, Discount: ${discount_amount:.2f}, Final: ${final_price:.2f}"
```

### 2.2 Async Tool

```python
from strands import tool
import aiohttp

@tool
async def fetch_weather(city: str) -> str:
    """Fetch current weather for a city from the weather API.
    
    Args:
        city: Name of the city to get weather for
        
    Returns:
        Current weather conditions as a formatted string
    """
    async with aiohttp.ClientSession() as session:
        url = f"https://api.weather.example/v1/current?city={city}"
        async with session.get(url) as response:
            if response.status == 200:
                data = await response.json()
                return f"Weather in {city}: {data['condition']}, {data['temp']}°F"
            return f"Error fetching weather: {response.status}"
```

### 2.3 Tool with Context Access

```python
from strands import tool
from strands.types.tools import ToolContext

@tool(context=True)
def get_user_preference(key: str, tool_context: ToolContext) -> str:
    """Retrieve a user preference from the current session context.
    
    Args:
        key: The preference key to retrieve
        
    Returns:
        The preference value or 'not found' message
    """
    # Access invocation state
    user_id = tool_context.invocation_state.get("user_id")
    preferences = tool_context.invocation_state.get("preferences", {})
    
    value = preferences.get(key)
    if value is None:
        return f"Preference '{key}' not found for user {user_id}"
    return f"User {user_id} preference '{key}': {value}"
```

### 2.4 Class-Based Tools (Stateful)

```python
from strands import tool

class DatabaseTools:
    """Database tools with shared connection."""
    
    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self._connection = None
    
    @property
    def connection(self):
        if self._connection is None:
            self._connection = create_db_connection(self.connection_string)
        return self._connection
    
    @tool
    def query_database(self, query: str) -> str:
        """Execute a read-only SQL query against the database.
        
        Args:
            query: SQL SELECT query to execute
            
        Returns:
            Query results as formatted text
        """
        if not query.strip().upper().startswith("SELECT"):
            return "Error: Only SELECT queries are allowed"
        
        results = self.connection.execute(query).fetchall()
        return format_results(results)
    
    @tool
    def insert_record(self, table: str, data: dict) -> str:
        """Insert a new record into a database table.
        
        Args:
            table: Name of the table
            data: Dictionary of column-value pairs
            
        Returns:
            Confirmation message with inserted record ID
        """
        # Implementation
        return f"Inserted record into {table}"

# Usage
db_tools = DatabaseTools("postgresql://localhost/mydb")
agent = Agent(tools=[db_tools.query_database, db_tools.insert_record])
```

### 2.5 Module-Based Tools (No SDK Dependency)

For tools that should work outside Strands:

```python
# tools/external_tool.py

# Tool specification
TOOL_SPEC = {
    "name": "external_api_call",
    "description": "Call an external API endpoint",
    "inputSchema": {
        "type": "object",
        "properties": {
            "endpoint": {
                "type": "string",
                "description": "API endpoint URL"
            },
            "method": {
                "type": "string",
                "enum": ["GET", "POST"],
                "description": "HTTP method"
            }
        },
        "required": ["endpoint"]
    }
}

# Function with same name as tool spec name
def external_api_call(endpoint: str, method: str = "GET") -> dict:
    """Implementation without @tool decorator."""
    import requests
    response = requests.request(method, endpoint)
    return {
        "status": "success",
        "content": [{"text": response.text[:1000]}]
    }
```

---

## 3. Tool Executors

### 3.1 Concurrent Execution (DEFAULT)

Tools execute in parallel when possible:

```python
from strands import Agent
from strands.tools.executors import ConcurrentToolExecutor

# Default behavior - concurrent execution
agent = Agent(tools=[tool1, tool2, tool3])

# Explicit concurrent executor
agent = Agent(
    tool_executor=ConcurrentToolExecutor(),
    tools=[tool1, tool2, tool3],
)
```

**Use Concurrent When**:
- Tools are independent (no shared state dependencies)
- Tools call external APIs (parallel = faster)
- Order of execution doesn't matter
- **DEFAULT: Use this unless you have a reason not to**

### 3.2 Sequential Execution

Tools execute one at a time:

```python
from strands import Agent
from strands.tools.executors import SequentialToolExecutor

agent = Agent(
    tool_executor=SequentialToolExecutor(),
    tools=[screenshot_tool, analyze_tool, email_tool],
)
```

**Use Sequential When**:
- Tool B depends on Tool A's output
- Tools modify shared state
- Order of side effects matters
- Resource constraints (API rate limits)

### 3.3 Decision Guide

**ASK** for clarification if:
- Tools interact with shared resources
- Prompt mentions "then" or sequential language
- Unclear if tools have dependencies

**Example clarification**:
> "I notice you want to use multiple tools. Should they run:
> 1. **Concurrently** (faster, tools are independent)
> 2. **Sequentially** (tools depend on each other's results)
>
> Which matches your use case?"

---

## 4. MCP Integration

### 4.1 Stdio Transport (Local Processes)

```python
from strands import Agent
from strands.tools.mcp import MCPClient
from mcp import stdio_client, StdioServerParameters

# Create MCP client for local server
mcp_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx",
            args=["awslabs.aws-documentation-mcp-server@latest"],
        )
    )
)

# Use as context manager
with mcp_client:
    tools = mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    response = agent("Search AWS docs for Lambda functions")
```

### 4.2 SSE Transport (Remote Servers)

```python
from strands import Agent
from strands.tools.mcp import MCPClient
from mcp.client.sse import sse_client

# Create MCP client for SSE server
sse_mcp_client = MCPClient(
    lambda: sse_client("http://localhost:8000/sse")
)

with sse_mcp_client:
    tools = sse_mcp_client.list_tools_sync()
    agent = Agent(tools=tools)
    response = agent("Use the remote MCP tools")
```

### 4.3 Multiple MCP Servers

```python
from strands import Agent
from strands.tools.mcp import MCPClient
from mcp import stdio_client, StdioServerParameters

# Multiple MCP clients
aws_docs = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="uvx",
            args=["awslabs.aws-documentation-mcp-server@latest"],
        )
    )
)

github_client = MCPClient(
    lambda: stdio_client(
        StdioServerParameters(
            command="npx",
            args=["@modelcontextprotocol/server-github"],
            env={"GITHUB_TOKEN": os.environ["GITHUB_TOKEN"]},
        )
    )
)

# Combine tools from multiple servers
with aws_docs, github_client:
    all_tools = aws_docs.list_tools_sync() + github_client.list_tools_sync()
    agent = Agent(tools=all_tools)
    response = agent("Search AWS docs and check GitHub issues")
```

### 4.4 MCP Configuration Pattern

```python
# config/mcp_servers.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class MCPServerConfig:
    name: str
    command: str
    args: list[str]
    env: Optional[dict] = None

MCP_SERVERS = [
    MCPServerConfig(
        name="aws-docs",
        command="uvx",
        args=["awslabs.aws-documentation-mcp-server@latest"],
    ),
    MCPServerConfig(
        name="filesystem",
        command="npx",
        args=["@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"],
    ),
]

def create_mcp_clients(configs: list[MCPServerConfig]) -> list[MCPClient]:
    """Create MCP clients from configuration."""
    from strands.tools.mcp import MCPClient
    from mcp import stdio_client, StdioServerParameters
    
    clients = []
    for config in configs:
        client = MCPClient(
            lambda c=config: stdio_client(
                StdioServerParameters(
                    command=c.command,
                    args=c.args,
                    env=c.env,
                )
            )
        )
        clients.append(client)
    return clients
```

---

## 5. Community Tools

### 5.1 Available Tools (strands-agents-tools)

```python
from strands_tools import (
    # Computation
    calculator,          # Mathematical calculations
    python_repl,         # Execute Python code
    
    # Web & Data
    http_request,        # Make HTTP requests
    
    # File Operations
    file_read,           # Read files
    file_write,          # Write files
    
    # System
    shell,               # Execute shell commands
    current_time,        # Get current time
    
    # Knowledge & Memory
    memory,              # Persistent memory
    retrieve,            # RAG retrieval (Bedrock KB)
    
    # Multi-Agent
    swarm,               # Dynamic swarm creation
    graph,               # Dynamic graph creation
    workflow,            # Workflow execution
    use_llm,             # Lightweight LLM calls
)

agent = Agent(
    tools=[calculator, http_request, file_read, current_time],
)
```

### 5.2 Tool Selection Guidance

| Use Case | Recommended Tools |
|----------|-------------------|
| Math/Calculations | `calculator`, `python_repl` |
| API Integration | `http_request` |
| File Management | `file_read`, `file_write` |
| System Commands | `shell` |
| Knowledge Retrieval | `retrieve`, `memory` |
| Multi-Agent Dynamic | `swarm`, `graph`, `workflow` |

---

## 6. Tool Abstraction for Deployment

### 6.1 Deployment-Agnostic Tool Pattern

Design tools that work in any environment:

```python
# tools/base.py
from abc import ABC, abstractmethod
from strands import tool

class DeployableToolBase(ABC):
    """Base class for tools that can deploy anywhere."""
    
    @abstractmethod
    def execute(self, **kwargs) -> str:
        """Override in subclass."""
        pass
    
    def __call__(self, **kwargs) -> str:
        return self.execute(**kwargs)

# tools/data_lookup.py
from strands import tool
from .base import DeployableToolBase

class DataLookupTool(DeployableToolBase):
    """Tool that looks up data - works locally, in Lambda, or AgentCore."""
    
    def __init__(self, data_source: str):
        self.data_source = data_source
    
    def execute(self, query: str) -> str:
        # Implementation works anywhere
        return f"Result for {query} from {self.data_source}"
    
    @tool
    def lookup(self, query: str) -> str:
        """Look up data based on a query.
        
        Args:
            query: The search query
            
        Returns:
            Matching data results
        """
        return self.execute(query=query)

# Usage
data_tool = DataLookupTool(data_source="dynamodb://table-name")
agent = Agent(tools=[data_tool.lookup])
```

### 6.2 Environment-Aware Tool Configuration

```python
# tools/config.py
import os
from enum import Enum

class Environment(Enum):
    LOCAL = "local"
    LAMBDA = "lambda"
    DOCKER = "docker"
    AGENTCORE = "agentcore"

def get_environment() -> Environment:
    """Detect current execution environment."""
    if os.environ.get("AWS_LAMBDA_FUNCTION_NAME"):
        return Environment.LAMBDA
    if os.environ.get("AGENTCORE_RUNTIME"):
        return Environment.AGENTCORE
    if os.path.exists("/.dockerenv"):
        return Environment.DOCKER
    return Environment.LOCAL

def get_storage_path() -> str:
    """Get appropriate storage path for environment."""
    env = get_environment()
    paths = {
        Environment.LOCAL: "./data",
        Environment.LAMBDA: "/tmp",
        Environment.DOCKER: "/app/data",
        Environment.AGENTCORE: "/var/agent/data",
    }
    return paths.get(env, "./data")
```

---

## 7. Tool Error Handling

### 7.1 Graceful Error Returns

```python
from strands import tool

@tool
def risky_operation(input_data: str) -> str:
    """Perform an operation that might fail.
    
    Args:
        input_data: Data to process
        
    Returns:
        Success result or error message
    """
    try:
        result = perform_operation(input_data)
        return f"Success: {result}"
    except ValidationError as e:
        return f"Validation Error: {e}. Please check your input format."
    except ConnectionError as e:
        return f"Connection Error: Unable to reach service. Details: {e}"
    except Exception as e:
        return f"Unexpected Error: {e}. Please try again or contact support."
```

### 7.2 Tool Result Format

The SDK auto-converts returns, but you can be explicit:

```python
from strands import tool
from strands.types.tools import ToolResult

@tool
def detailed_result(query: str) -> ToolResult:
    """Tool with explicit result format.
    
    Args:
        query: Search query
        
    Returns:
        Detailed tool result
    """
    return {
        "status": "success",
        "content": [
            {"text": f"Found results for: {query}"},
            {"text": "Result 1: ..."},
            {"text": "Result 2: ..."},
        ]
    }
```

---

## 8. Tool Documentation Best Practices

### 8.1 Docstring Quality Matters

The LLM uses your docstring to decide when to call the tool:

```python
# ❌ Bad - vague, LLM won't know when to use
@tool
def process(data: str) -> str:
    """Process data."""
    pass

# ✅ Good - specific, LLM understands purpose
@tool
def calculate_shipping_cost(
    weight_kg: float,
    destination_country: str,
    shipping_speed: str = "standard",
) -> str:
    """Calculate shipping cost for a package to an international destination.
    
    Use this tool when the user asks about shipping costs, delivery fees,
    or wants to know how much it costs to send a package internationally.
    
    Args:
        weight_kg: Package weight in kilograms (must be positive)
        destination_country: Two-letter country code (e.g., 'US', 'GB', 'JP')
        shipping_speed: Delivery speed - 'express' (2-3 days), 
                       'standard' (5-7 days), or 'economy' (10-14 days)
        
    Returns:
        Formatted string with shipping cost in USD and estimated delivery date
        
    Examples:
        - "How much to ship a 2kg package to Japan?" -> Use with weight_kg=2.0, destination_country='JP'
        - "Express shipping cost to UK for 500g" -> Use with weight_kg=0.5, destination_country='GB', shipping_speed='express'
    """
    pass
```

---

## 9. Tool Testing

```python
# tests/test_tools.py
import pytest
from tools.custom_tools import calculate_discount, fetch_weather

def test_calculate_discount_valid():
    """Test discount calculation with valid inputs."""
    result = calculate_discount(100.0, 20.0)
    assert "Final: $80.00" in result

def test_calculate_discount_invalid():
    """Test discount calculation with invalid inputs."""
    result = calculate_discount(100.0, 150.0)
    assert "Error" in result

@pytest.mark.asyncio
async def test_fetch_weather():
    """Test async weather tool."""
    result = await fetch_weather("Seattle")
    assert "Seattle" in result or "Error" in result
```

Run tool tests independently:
```bash
pytest tests/test_tools.py -v
```

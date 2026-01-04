# Strands Agents SDK - Multi-Agent Patterns

> **Purpose**: Guide selection and implementation of agent architectural patterns.
> 
> ⚠️ **Note**: Multi-agent patterns (Swarm, Graph, Workflow) are **Python SDK only** as of the TypeScript preview.

---

## 1. Pattern Selection Decision Tree

```
                        START
                          │
                          ▼
              ┌──────────────────────┐
              │  Single task, no     │──── YES ───▶ Simple Agent
              │  collaboration?      │
              └──────────────────────┘
                          │ NO
                          ▼
              ┌──────────────────────┐
              │  Clear delegation    │──── YES ───▶ Agents-as-Tools
              │  hierarchy?          │
              └──────────────────────┘
                          │ NO
                          ▼
              ┌──────────────────────┐
              │  Deterministic       │──── YES ───▶ Graph
              │  workflow with       │
              │  conditions?         │
              └──────────────────────┘
                          │ NO
                          ▼
              ┌──────────────────────┐
              │  Collaborative       │──── YES ───▶ Swarm
              │  exploration with    │
              │  dynamic handoffs?   │
              └──────────────────────┘
                          │ NO
                          ▼
              ┌──────────────────────┐
              │  Sequential pipeline │──── YES ───▶ Workflow
              │  with dependencies?  │
              └──────────────────────┘
```

### 1.1 Quick Reference

| Pattern | Control Flow | Use Case | Complexity |
|---------|-------------|----------|------------|
| **Simple Agent** | Single agent | Basic Q&A, simple tasks | Low |
| **Agents-as-Tools** | Orchestrator delegates | Specialist consultation | Low-Medium |
| **Swarm** | Emergent, agent-driven | Brainstorming, research | Medium |
| **Graph** | Deterministic, condition-based | Business processes, validation | Medium-High |
| **Workflow** | Sequential, dependency-based | Pipelines, data processing | High |

---

## 2. Simple Agent Pattern

### 2.1 When to Use

- Single-turn or multi-turn conversations
- Tasks that don't require specialist agents
- Prototyping and development
- User-facing chat interfaces

### 2.2 Implementation

```python
from strands import Agent
from strands.models import BedrockModel
from strands_tools import calculator, http_request

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

agent = Agent(
    model=model,
    tools=[calculator, http_request],
    system_prompt="""You are a helpful assistant that can:
    - Perform calculations
    - Make HTTP requests to fetch data
    Always explain your reasoning.""",
)

# Single invocation
response = agent("Calculate the compound interest on $10,000 at 5% for 3 years")

# Multi-turn conversation (context maintained)
agent("My name is Alice")
response = agent("What's my name?")  # Will remember "Alice"
```

### 2.3 Signals to Use This Pattern

- "build a chatbot"
- "create a simple agent"
- "help me with [single task]"
- No mentions of collaboration or multiple specialists

---

## 3. Agents-as-Tools Pattern

### 3.1 When to Use

- Clear hierarchy: orchestrator → specialists
- Orchestrator decides when to delegate
- Specialists have distinct, well-defined roles
- Results need to be combined by orchestrator

### 3.2 Implementation

```python
from strands import Agent, tool
from strands.models import BedrockModel
from strands_tools import calculator, memory

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

# Create specialist agents
data_analyst = Agent(
    name="data_analyst",
    model=model,
    tools=[calculator],
    system_prompt="You are an expert data analyst. Analyze data and provide insights.",
)

researcher = Agent(
    name="researcher",
    model=model,
    tools=[memory],
    system_prompt="You are an expert researcher. Find and synthesize information.",
)

# Wrap specialists as tools
@tool
def analyze_data(query: str) -> str:
    """Delegate data analysis tasks to the data analyst specialist.
    
    Args:
        query: The data analysis question or task
        
    Returns:
        Analysis results from the specialist
    """
    result = data_analyst(query)
    return str(result)

@tool
def research_topic(query: str) -> str:
    """Delegate research tasks to the research specialist.
    
    Args:
        query: The research question or topic
        
    Returns:
        Research findings from the specialist
    """
    result = researcher(query)
    return str(result)

# Create orchestrator with specialist tools
orchestrator = Agent(
    name="orchestrator",
    model=model,
    tools=[analyze_data, research_topic],
    system_prompt="""You are a project manager who coordinates specialists.
    Delegate tasks to appropriate specialists and synthesize their results.
    Use analyze_data for number crunching and data questions.
    Use research_topic for information gathering.""",
)

# Orchestrator decides when to use specialists
response = orchestrator(
    "Research the impact of AI on job markets and analyze the projected growth rates"
)
```

### 3.3 Signals to Use This Pattern

- "orchestrator that calls other agents"
- "main agent with specialists"
- "delegate to [specific role]"
- Clear hierarchy mentioned

---

## 4. Swarm Pattern

### 4.1 When to Use

- Collaborative exploration
- Brainstorming and ideation
- Research requiring multiple perspectives
- Tasks benefit from diverse expertise
- Emergent behavior is acceptable

### 4.2 Implementation

```python
import logging
from strands import Agent
from strands.multiagent import Swarm
from strands.models import BedrockModel
from strands_tools import memory, calculator, file_write

# Enable debug logging
logging.getLogger("strands.multiagent").setLevel(logging.DEBUG)
logging.basicConfig(format="%(levelname)s | %(name)s | %(message)s")

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

# Create specialized agents
researcher = Agent(
    name="researcher",
    model=model,
    tools=[memory],
    system_prompt="""You are a thorough researcher.
    Research topics and share findings with the team.
    Hand off to analyst when data needs analysis.""",
    description="Focuses on information gathering and research",  # Helps other agents understand this agent
)

analyst = Agent(
    name="analyst",
    model=model,
    tools=[calculator, memory],
    system_prompt="""You are a data analyst.
    Analyze data and extract insights.
    Hand off to writer when findings need to be documented.""",
    description="Focuses on data analysis and insights",
)

writer = Agent(
    name="writer",
    model=model,
    tools=[file_write, memory],
    system_prompt="""You are a technical writer.
    Create comprehensive reports from research and analysis.
    Ensure all findings are well-documented.""",
    description="Focuses on documentation and report writing",
)

# Create swarm
research_team = Swarm(
    agents=[researcher, analyst, writer],
    entry_point=researcher,  # Optional: specify which agent starts
)

# Execute swarm
result = research_team(
    "Research the history of AI since 1950 and create a comprehensive report"
)
print(result)
```

### 4.3 Swarm with Shared Context

```python
from strands.multiagent import Swarm

# Shared context available to all agents
swarm = Swarm(
    agents=[researcher, analyst, writer],
)

# Pass invocation state for context sharing
result = swarm(
    "Analyze market trends",
    invocation_state={
        "project_id": "market-analysis-2025",
        "user_id": "user-123",
        "debug_mode": True,
    }
)
```

### 4.4 Swarm Configuration

```python
swarm = Swarm(
    agents=[researcher, analyst, writer],
    entry_point=researcher,
    max_handoffs=10,  # Limit handoff chains
    repetitive_handoff_detection_window=5,  # Detect ping-pong
    repetitive_handoff_min_unique_agents=2,  # Minimum unique agents
)
```

### 4.5 Signals to Use This Pattern

- "team of agents"
- "collaborative research"
- "brainstorming"
- "multiple perspectives"
- "agents work together"

---

## 5. Graph Pattern

### 5.1 When to Use

- Deterministic workflow with explicit transitions
- Conditional routing based on outcomes
- Business processes with defined steps
- Validation pipelines with error paths
- When execution order must be controlled

### 5.2 Basic Graph Implementation

```python
import logging
from strands import Agent
from strands.multiagent import GraphBuilder
from strands.models import BedrockModel

logging.getLogger("strands.multiagent").setLevel(logging.DEBUG)

model = BedrockModel(
    model_id="us.anthropic.claude-sonnet-4-20250514-v1:0",
    region_name="us-east-1",
)

# Create agents for each node
analyzer = Agent(
    name="analyzer",
    model=model,
    system_prompt="Analyze incoming requests and categorize them.",
)

normal_processor = Agent(
    name="normal_processor",
    model=model,
    system_prompt="Handle routine requests.",
)

critical_processor = Agent(
    name="critical_processor",
    model=model,
    system_prompt="Handle critical or urgent requests.",
)

# Build the graph
builder = GraphBuilder()
builder.add_node(analyzer, "analyze")
builder.add_node(normal_processor, "normal")
builder.add_node(critical_processor, "critical")

# Add edges
builder.add_edge("analyze", "normal")  # Simple edge
builder.set_entry_point("analyze")

graph = builder.build()

# Execute
result = graph("Process this customer request: I need help with billing")
```

### 5.3 Graph with Conditional Edges

```python
from strands.multiagent import GraphBuilder

builder = GraphBuilder()
builder.add_node(analyzer, "analyze")
builder.add_node(normal_processor, "normal")
builder.add_node(critical_processor, "critical")

# Conditional routing based on state
def is_critical(state: dict) -> bool:
    """Determine if request is critical based on analysis."""
    # Check the result from the analyzer
    result = state.get("result", "")
    return "urgent" in result.lower() or "critical" in result.lower()

def is_normal(state: dict) -> bool:
    """Route to normal processor if not critical."""
    return not is_critical(state)

# Add conditional edges
builder.add_edge("analyze", "critical", condition=is_critical)
builder.add_edge("analyze", "normal", condition=is_normal)
builder.set_entry_point("analyze")

graph = builder.build()
```

### 5.4 Cyclic Graph (Feedback Loops)

```python
# Graph with iteration/refinement
builder = GraphBuilder()
builder.add_node(writer, "write")
builder.add_node(reviewer, "review")

def needs_revision(state: dict) -> bool:
    return "needs revision" in state.get("result", "").lower()

def is_approved(state: dict) -> bool:
    return "approved" in state.get("result", "").lower()

# Create cycle
builder.add_edge("write", "review")
builder.add_edge("review", "write", condition=needs_revision)  # Loop back
# End node is implicit when no outgoing edges match

builder.set_entry_point("write")
graph = builder.build()
```

### 5.5 Nested Graphs

```python
from strands.multiagent import GraphBuilder, Swarm

# Create a swarm as a node in the graph
research_swarm = Swarm([researcher, analyst])

# Build graph with swarm as node
builder = GraphBuilder()
builder.add_node(research_swarm, "research_team")  # Swarm as node
builder.add_node(summarizer, "summarize")
builder.add_edge("research_team", "summarize")
builder.set_entry_point("research_team")

graph = builder.build()
```

### 5.6 Signals to Use This Pattern

- "workflow with conditions"
- "if X then Y else Z"
- "business process"
- "approval flow"
- "deterministic routing"
- "validation pipeline"

---

## 6. Workflow Pattern

### 6.1 When to Use

- Sequential pipeline with dependencies
- Task ordering is critical
- Steps must complete before dependent steps start
- Automated data processing pipelines
- Standard business processes

### 6.2 Implementation (Using Workflow Tool)

```python
from strands import Agent
from strands_tools import workflow

# Create agent with workflow capability
agent = Agent(tools=[workflow])

# Define and execute workflow
agent.tool.workflow(
    action="create",
    workflow_id="data_analysis",
    tasks=[
        {
            "task_id": "extract",
            "description": "Extract financial data from the quarterly report",
            "system_prompt": "You extract and structure financial data.",
            "priority": 5,
        },
        {
            "task_id": "analyze",
            "description": "Analyze trends compared to previous quarters",
            "dependencies": ["extract"],  # Runs after extract
            "system_prompt": "You identify trends in financial data.",
            "priority": 3,
        },
        {
            "task_id": "report",
            "description": "Generate comprehensive analysis report",
            "dependencies": ["analyze"],  # Runs after analyze
            "system_prompt": "You create clear financial reports.",
            "priority": 2,
        },
    ],
)

# Execute workflow
agent.tool.workflow(action="start", workflow_id="data_analysis")

# Check status
status = agent.tool.workflow(action="status", workflow_id="data_analysis")

# Control workflow
agent.tool.workflow(action="pause", workflow_id="data_analysis")
agent.tool.workflow(action="resume", workflow_id="data_analysis")
```

### 6.3 Custom Workflow Implementation

```python
from strands import Agent
import asyncio

class WorkflowExecutor:
    """Custom workflow executor with dependency resolution."""
    
    def __init__(self, agents: dict[str, Agent]):
        self.agents = agents
        self.results = {}
    
    async def execute(self, tasks: list[dict]) -> dict:
        """Execute tasks respecting dependencies."""
        completed = set()
        
        while len(completed) < len(tasks):
            # Find tasks ready to run
            ready = [
                t for t in tasks
                if t["task_id"] not in completed
                and all(d in completed for d in t.get("dependencies", []))
            ]
            
            # Execute ready tasks concurrently
            async_tasks = []
            for task in ready:
                agent = self.agents.get(task.get("agent", "default"))
                async_tasks.append(self._execute_task(agent, task))
            
            results = await asyncio.gather(*async_tasks)
            
            for task, result in zip(ready, results):
                self.results[task["task_id"]] = result
                completed.add(task["task_id"])
        
        return self.results
    
    async def _execute_task(self, agent: Agent, task: dict) -> str:
        """Execute a single task."""
        # Include results from dependencies in prompt
        context = "\n".join(
            f"Previous result ({dep}): {self.results[dep]}"
            for dep in task.get("dependencies", [])
        )
        prompt = f"{context}\n\nTask: {task['description']}"
        result = await agent.invoke_async(prompt)
        return str(result)
```

### 6.4 Signals to Use This Pattern

- "pipeline"
- "sequential steps"
- "task dependencies"
- "data processing workflow"
- "ETL process"
- "step 1, then step 2, then step 3"

---

## 7. Pattern Comparison

### 7.1 Control Flow

| Pattern | Who Decides? | Predictability |
|---------|-------------|----------------|
| Simple | Single agent | High |
| Agents-as-Tools | Orchestrator | High |
| Swarm | Agents collectively | Low (emergent) |
| Graph | Developer-defined conditions | High |
| Workflow | Dependencies | High |

### 7.2 When to ASK for Clarification

**ASK** if the prompt includes:
- "multiple agents" without clear pattern preference
- "collaborate" (could be Swarm or Graph)
- "process" (could be Graph or Workflow)
- Ambiguous task decomposition

**Example clarification**:
> "I see you want multiple agents to work together. Should they:
> 1. **Collaborate freely** (Swarm) - agents hand off dynamically based on their judgment
> 2. **Follow a defined flow** (Graph) - deterministic transitions with conditions
> 3. **Execute sequentially** (Workflow) - pipeline with dependencies
> 
> Which best matches your use case?"

---

## 8. Async Execution

All multi-agent patterns support async execution:

```python
import asyncio

# Swarm async
result = await swarm.invoke_async("Research topic")

# Graph async
result = await graph.invoke_async("Process request")

# Streaming (for real-time updates)
async for event in swarm.stream_async("Research topic"):
    print(event)
```

---

## 9. Anti-Patterns to Avoid

### 9.1 Over-Orchestration

❌ **Wrong**: Creating complex graphs for simple tasks
```python
# Don't do this for simple Q&A
builder = GraphBuilder()
builder.add_node(question_parser, "parse")
builder.add_node(answer_generator, "generate")
builder.add_node(response_formatter, "format")
# ... excessive nodes for simple task
```

✅ **Right**: Use simple agent
```python
agent = Agent(system_prompt="Answer questions clearly and concisely")
```

### 9.2 Swarm When Graph is Needed

❌ **Wrong**: Using Swarm for deterministic processes
```python
# Don't use Swarm for approval workflows
swarm = Swarm([submitter, reviewer, approver])  # Order matters!
```

✅ **Right**: Use Graph for deterministic flow
```python
builder = GraphBuilder()
builder.add_node(submitter, "submit")
builder.add_node(reviewer, "review")
builder.add_edge("submit", "review")
# Deterministic flow
```

### 9.3 Ignoring Handoff Limits

❌ **Wrong**: Unlimited handoffs causing infinite loops
```python
swarm = Swarm(agents=[a, b, c])  # No limits
```

✅ **Right**: Set appropriate limits
```python
swarm = Swarm(
    agents=[a, b, c],
    max_handoffs=10,
    repetitive_handoff_detection_window=5,
)
```

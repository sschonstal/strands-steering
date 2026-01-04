# Strands Agents SDK - TypeScript Implementation

> **Purpose**: TypeScript-specific patterns for Strands Agents SDK.
> 
> ⚠️ **IMPORTANT**: The TypeScript SDK is in **PUBLIC PREVIEW**. APIs may change, and some Python SDK features are not yet available.

---

## 1. Preview Status & Limitations

### 1.1 Current Limitations (as of SDK preview)

You **MUST** be aware of these limitations:

| Feature | Python SDK | TypeScript SDK |
|---------|------------|----------------|
| Core Agent | ✅ Stable | ✅ Available |
| Custom Tools | ✅ @tool decorator | ✅ tool() function with Zod |
| MCP Integration | ✅ Full support | ✅ Available |
| Streaming | ✅ Full support | ✅ Available |
| Multi-Agent (Swarm) | ✅ Available | ❌ Not yet |
| Multi-Agent (Graph) | ✅ Available | ❌ Not yet |
| Structured Output | ✅ Pydantic | ❌ Not yet |
| Session Managers | ✅ File/S3 | ❌ Not yet |
| Community Tools | ✅ strands-agents-tools | ⚠️ Limited (vended_tools) |
| Hooks System | ✅ Full | ⚠️ Limited |

### 1.2 When to Use TypeScript SDK

**USE TypeScript SDK when**:
- Project is already TypeScript/JavaScript
- Building browser-based agents
- Simple single-agent use cases
- Node.js serverless functions (Lambda)
- Team prefers TypeScript

**USE Python SDK instead when**:
- Multi-agent orchestration required (Swarm, Graph)
- Structured output with schema validation needed
- Session persistence required
- Full feature set is necessary

---

## 2. Installation & Setup

### 2.1 Installation

```bash
# Core SDK
npm install @strands-agents/sdk

# With Zod for tool schemas (REQUIRED for custom tools)
npm install @strands-agents/sdk zod
```

### 2.2 TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 2.3 Package.json Setup

```json
{
  "name": "my-strands-agent",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "npx tsx src/index.ts"
  },
  "dependencies": {
    "@strands-agents/sdk": "^0.1.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0"
  }
}
```

---

## 3. Agent Creation Patterns

### 3.1 Basic Agent

```typescript
import { Agent } from '@strands-agents/sdk'

// Minimal agent (uses Bedrock defaults)
const agent = new Agent()
const response = await agent.invoke("What is 2 + 2?")
console.log(response.lastMessage)
```

### 3.2 Agent with Custom Model

```typescript
import { Agent, BedrockModel } from '@strands-agents/sdk'

const model = new BedrockModel({
    modelId: 'us.anthropic.claude-sonnet-4-20250514-v1:0',
    region: 'us-east-1',
    temperature: 0.3,
    maxTokens: 4096,
})

const agent = new Agent({
    model,
    systemPrompt: 'You are a helpful coding assistant.',
})

const response = await agent.invoke("Explain async/await in JavaScript")
```

### 3.3 Agent with Tools

```typescript
import { Agent, tool } from '@strands-agents/sdk'
import { z } from 'zod'

// Define custom tool with Zod schema
const weatherTool = tool({
    name: 'get_weather',
    description: 'Get the current weather for a specific location.',
    inputSchema: z.object({
        location: z.string().describe('The city and state, e.g., San Francisco, CA'),
        unit: z.enum(['celsius', 'fahrenheit']).default('fahrenheit').describe('Temperature unit'),
    }),
    callback: async (input) => {
        // Input is fully typed based on Zod schema
        const { location, unit } = input
        // Implementation here
        return `The weather in ${location} is 72°${unit === 'celsius' ? 'C' : 'F'} and sunny.`
    },
})

const agent = new Agent({
    tools: [weatherTool],
})

const response = await agent.invoke("What's the weather in Seattle?")
```

### 3.4 Using Vended Tools

```typescript
import { Agent } from '@strands-agents/sdk'
import { notebook } from '@strands-agents/sdk/vended_tools/notebook'
import { fileEditor } from '@strands-agents/sdk/vended_tools/file_editor'
import { httpRequest } from '@strands-agents/sdk/vended_tools/http_request'

const agent = new Agent({
    tools: [notebook, fileEditor, httpRequest],
    systemPrompt: 'You can manage notes and make HTTP requests.',
})
```

---

## 4. MCP Integration

### 4.1 MCP Client with Stdio Transport

```typescript
import { Agent, McpClient } from '@strands-agents/sdk'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

// Create MCP client
const mcpClient = new McpClient({
    transport: new StdioClientTransport({
        command: 'uvx',
        args: ['awslabs.aws-documentation-mcp-server@latest'],
    }),
})

// Create agent with MCP tools
const agent = new Agent({
    systemPrompt: 'You are a helpful assistant using MCP tools.',
    tools: [mcpClient],  // Pass MCP client directly as tool source
})

try {
    const response = await agent.invoke("Search AWS documentation for Lambda functions")
    console.log(response.lastMessage)
} finally {
    // Clean up MCP connection
    await mcpClient.disconnect()
}
```

### 4.2 MCP with SSE Transport

```typescript
import { Agent, McpClient } from '@strands-agents/sdk'
import { SSEClientTransport } from '@modelcontextprotocol/sdk/client/sse.js'

const mcpClient = new McpClient({
    transport: new SSEClientTransport(new URL('http://localhost:8000/sse')),
})

const agent = new Agent({
    tools: [mcpClient],
})
```

---

## 5. Streaming Patterns

### 5.1 Async Generator Streaming

```typescript
import { Agent, AgentStreamEvent } from '@strands-agents/sdk'

const agent = new Agent()

function processEvent(event: AgentStreamEvent): void {
    switch (event.type) {
        case 'beforeInvocationEvent':
            console.log('🔄 Agent started')
            break
            
        case 'modelContentBlockDeltaEvent':
            if (event.delta.type === 'textDelta') {
                process.stdout.write(event.delta.text)
            }
            break
            
        case 'modelContentBlockStartEvent':
            if (event.start?.type === 'toolUseStart') {
                console.log(`\n🔧 Using tool: ${event.start.name}`)
            }
            break
            
        case 'afterInvocationEvent':
            console.log('\n✅ Agent completed')
            if (event.error) {
                console.error('Error:', event.error)
            }
            break
    }
}

// Stream responses
const responseGenerator = agent.stream("Explain quantum computing")
for await (const event of responseGenerator) {
    processEvent(event)
}
```

### 5.2 Express.js with Server-Sent Events

```typescript
import express from 'express'
import { Agent } from '@strands-agents/sdk'

const app = express()
app.use(express.json())

const agent = new Agent()

app.post('/chat/stream', async (req, res) => {
    const { message } = req.body
    
    res.setHeader('Content-Type', 'text/event-stream')
    res.setHeader('Cache-Control', 'no-cache')
    res.setHeader('Connection', 'keep-alive')
    
    try {
        for await (const event of agent.stream(message)) {
            if (event.type === 'modelContentBlockDeltaEvent' && 
                event.delta.type === 'textDelta') {
                res.write(`data: ${JSON.stringify({ text: event.delta.text })}\n\n`)
            }
        }
        res.write('data: [DONE]\n\n')
    } catch (error) {
        res.write(`data: ${JSON.stringify({ error: String(error) })}\n\n`)
    } finally {
        res.end()
    }
})

app.listen(3000)
```

---

## 6. Model Providers

### 6.1 Amazon Bedrock (Default)

```typescript
import { Agent, BedrockModel } from '@strands-agents/sdk'

const model = new BedrockModel({
    modelId: 'us.anthropic.claude-sonnet-4-20250514-v1:0',
    region: 'us-east-1',
    temperature: 0.3,
    maxTokens: 4096,
})

const agent = new Agent({ model })
```

### 6.2 OpenAI

```typescript
import { Agent } from '@strands-agents/sdk'
import { OpenAIModel } from '@strands-agents/sdk/openai'

// Automatically uses process.env.OPENAI_API_KEY
const model = new OpenAIModel({
    modelId: 'gpt-4-turbo',
    temperature: 0.7,
})

const agent = new Agent({ model })
```

### 6.3 Anthropic Direct

```typescript
import { Agent } from '@strands-agents/sdk'
import { AnthropicModel } from '@strands-agents/sdk/anthropic'

// Automatically uses process.env.ANTHROPIC_API_KEY
const model = new AnthropicModel({
    modelId: 'claude-sonnet-4-20250514',
    maxTokens: 4096,
})

const agent = new Agent({ model })
```

---

## 7. Error Handling

### 7.1 Basic Error Handling

```typescript
import { Agent } from '@strands-agents/sdk'

const agent = new Agent()

async function safeInvoke(prompt: string): Promise<string | null> {
    try {
        const response = await agent.invoke(prompt)
        return response.lastMessage?.content?.[0]?.text ?? null
    } catch (error) {
        if (error instanceof Error) {
            console.error('Agent error:', error.message)
            
            // Handle specific error types
            if (error.message.includes('ThrottlingException')) {
                console.log('Rate limited, please retry')
            } else if (error.message.includes('context window')) {
                console.log('Context too large, reduce conversation')
            }
        }
        return null
    }
}
```

### 7.2 Retry with Exponential Backoff

```typescript
async function invokeWithRetry(
    agent: Agent,
    prompt: string,
    maxRetries = 3,
    baseDelay = 2000,
): Promise<string> {
    let lastError: Error | null = null
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            const response = await agent.invoke(prompt)
            return response.lastMessage?.content?.[0]?.text ?? ''
        } catch (error) {
            lastError = error as Error
            
            if (attempt < maxRetries) {
                const delay = Math.min(baseDelay * Math.pow(2, attempt), 60000)
                console.warn(`Attempt ${attempt + 1} failed, retrying in ${delay}ms...`)
                await new Promise(resolve => setTimeout(resolve, delay))
            }
        }
    }
    
    throw lastError ?? new Error('All retries exhausted')
}
```

---

## 8. Environment Configuration

### 8.1 AWS Credentials

```typescript
// Option 1: Environment variables
// AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN

// Option 2: AWS Profile
// AWS_PROFILE=my-profile

// Option 3: Bedrock API Key (for development)
// AWS_BEARER_TOKEN_BEDROCK=your-key

// Option 4: IAM Role (Lambda, ECS, EC2)
// Automatically detected
```

### 8.2 Configuration Pattern

```typescript
// config.ts
export interface AgentConfig {
    modelId: string
    region: string
    temperature: number
    maxTokens: number
}

export function getConfig(): AgentConfig {
    return {
        modelId: process.env.STRANDS_MODEL_ID ?? 
            'us.anthropic.claude-sonnet-4-20250514-v1:0',
        region: process.env.AWS_REGION ?? 'us-east-1',
        temperature: parseFloat(process.env.STRANDS_TEMPERATURE ?? '0.3'),
        maxTokens: parseInt(process.env.STRANDS_MAX_TOKENS ?? '4096'),
    }
}
```

---

## 9. Testing

### 9.1 Basic Agent Test

```typescript
// agent.test.ts
import { describe, it, expect, beforeAll } from 'vitest'  // or jest
import { Agent } from '@strands-agents/sdk'

describe('Agent', () => {
    let agent: Agent
    
    beforeAll(() => {
        agent = new Agent({
            systemPrompt: 'You are a test assistant.',
        })
    })
    
    it('should respond to simple queries', async () => {
        const response = await agent.invoke('Say hello')
        expect(response.lastMessage).toBeDefined()
    })
    
    it('should use tools when available', async () => {
        // Test with tools
    })
})
```

---

## 10. Limitations Workarounds

### 10.1 No Multi-Agent Support

If you need multi-agent in TypeScript, consider:

```typescript
// Workaround: Implement agents-as-tools pattern manually
import { Agent, tool } from '@strands-agents/sdk'
import { z } from 'zod'

// Create specialist agent
const researchAgent = new Agent({
    systemPrompt: 'You are a research specialist.',
})

// Wrap as tool for orchestrator
const researchTool = tool({
    name: 'research',
    description: 'Delegate research tasks to the research specialist.',
    inputSchema: z.object({
        query: z.string().describe('Research query'),
    }),
    callback: async ({ query }) => {
        const result = await researchAgent.invoke(query)
        return result.lastMessage?.content?.[0]?.text ?? 'No result'
    },
})

// Orchestrator uses research tool
const orchestrator = new Agent({
    tools: [researchTool],
    systemPrompt: 'You orchestrate tasks using available tools.',
})
```

### 10.2 No Session Persistence

```typescript
// Workaround: Manual message management
import { Agent } from '@strands-agents/sdk'
import fs from 'fs/promises'

class SessionManager {
    private sessionFile: string
    
    constructor(sessionId: string) {
        this.sessionFile = `./sessions/${sessionId}.json`
    }
    
    async saveMessages(messages: any[]): Promise<void> {
        await fs.writeFile(this.sessionFile, JSON.stringify(messages, null, 2))
    }
    
    async loadMessages(): Promise<any[]> {
        try {
            const data = await fs.readFile(this.sessionFile, 'utf-8')
            return JSON.parse(data)
        } catch {
            return []
        }
    }
}

// Usage: Restore and save messages manually
// Note: This is a workaround until session managers are available
```

---

## 11. Production Checklist (TypeScript)

Before deploying:

- [ ] TypeScript compiles without errors
- [ ] Dependencies pinned in package.json
- [ ] Environment variables documented
- [ ] Error handling implemented
- [ ] Streaming works for user-facing features
- [ ] MCP connections properly closed
- [ ] Tests pass
- [ ] Aware of preview limitations
- [ ] Fallback plan if Python SDK needed later

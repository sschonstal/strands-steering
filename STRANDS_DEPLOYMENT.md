# Strands Agents SDK - Deployment

> **Purpose**: Guide deployment of Strands agents from local development to production, with patterns that enable easy migration to AgentCore.

---

## 1. Deployment Philosophy

### 1.1 Core Principles

1. **Local First**: Always build and test locally before deploying
2. **Abstract for Portability**: Design components to work across environments
3. **AgentCore Ready**: Structure code for future AgentCore migration
4. **Environment Parity**: Minimize differences between dev and prod

### 1.2 Deployment Progression

```
Local Development → Lambda/Docker → AgentCore (Target)
       ↓                 ↓                ↓
   File System       S3/DynamoDB    AgentCore Memory
   Local LLM         Bedrock API    AgentCore Runtime
   Debug Mode        Monitoring     Full Observability
```

---

## 2. Local Development

### 2.1 Project Setup

```bash
# Create project structure
mkdir -p my-agent/{agents,tools,config,tests}
cd my-agent

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# .venv\Scripts\activate   # Windows

# Install dependencies
pip install strands-agents strands-agents-tools
pip install pytest python-dotenv  # Development tools

# Create requirements file
pip freeze > requirements.txt
```

### 2.2 Environment Configuration

```python
# config/settings.py
import os
from dataclasses import dataclass, field
from dotenv import load_dotenv

load_dotenv()  # Load .env file

@dataclass
class Settings:
    """Environment-aware configuration."""
    
    # Environment
    environment: str = field(default_factory=lambda: os.getenv("ENVIRONMENT", "development"))
    debug: bool = field(default_factory=lambda: os.getenv("DEBUG", "true").lower() == "true")
    
    # AWS
    aws_region: str = field(default_factory=lambda: os.getenv("AWS_REGION", "us-east-1"))
    
    # Model
    model_id: str = field(default_factory=lambda: os.getenv(
        "STRANDS_MODEL_ID", 
        "us.anthropic.claude-sonnet-4-20250514-v1:0"
    ))
    
    # Session storage
    session_bucket: str = field(default_factory=lambda: os.getenv("SESSION_BUCKET", ""))
    session_dir: str = field(default_factory=lambda: os.getenv("SESSION_DIR", "./sessions"))
    
    @property
    def is_production(self) -> bool:
        return self.environment == "production"
    
    @property
    def is_lambda(self) -> bool:
        return bool(os.getenv("AWS_LAMBDA_FUNCTION_NAME"))

settings = Settings()
```

```bash
# .env (development)
ENVIRONMENT=development
DEBUG=true
AWS_REGION=us-east-1
STRANDS_MODEL_ID=us.anthropic.claude-sonnet-4-20250514-v1:0
SESSION_DIR=./sessions
```

### 2.3 Local Agent Entry Point

```python
# main.py
from agents.main_agent import create_agent
from config.settings import settings

def main():
    """Local development entry point."""
    agent = create_agent()
    
    print(f"Agent initialized in {settings.environment} mode")
    print("Type 'quit' to exit\n")
    
    while True:
        user_input = input("You: ").strip()
        if user_input.lower() == 'quit':
            break
        
        response = agent(user_input)
        print(f"Agent: {response}\n")

if __name__ == "__main__":
    main()
```

### 2.4 Running Locally

```bash
# Basic run
python main.py

# With debug logging
DEBUG=true python main.py

# Run tests
pytest tests/ -v

# Test specific agent
python -c "from agents.main_agent import create_agent; a = create_agent(); print(a('Hello'))"
```

---

## 3. AWS Lambda Deployment

### 3.1 Lambda Handler Pattern

```python
# lambda_handler.py
import json
import logging
from typing import Any
from agents.main_agent import create_agent
from strands.session.s3_session_manager import S3SessionManager
import os

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event: dict, context: Any) -> dict:
    """AWS Lambda handler for agent invocation."""
    
    try:
        # Parse input
        body = json.loads(event.get("body", "{}"))
        message = body.get("message")
        session_id = body.get("session_id", "default")
        
        if not message:
            return {
                "statusCode": 400,
                "body": json.dumps({"error": "Message required"})
            }
        
        # Create session manager
        session_manager = S3SessionManager(
            session_id=session_id,
            bucket=os.environ["SESSION_BUCKET"],
            prefix=os.environ.get("SESSION_PREFIX", ""),
        )
        
        # Create and invoke agent
        agent = create_agent(session_manager=session_manager)
        response = agent(message)
        
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "response": str(response),
                "session_id": session_id,
            })
        }
        
    except Exception as e:
        logger.exception("Agent invocation failed")
        return {
            "statusCode": 500,
            "body": json.dumps({"error": str(e)})
        }
```

### 3.2 Streaming Lambda Handler (Function URL)

```python
# lambda_streaming_handler.py
import json
import logging
from agents.main_agent import create_agent

logger = logging.getLogger()

def handler(event: dict, context):
    """Lambda streaming handler using Function URL with response streaming."""
    
    body = json.loads(event.get("body", "{}"))
    message = body.get("message", "")
    
    agent = create_agent()
    
    # Return streaming response
    def generate():
        yield '{"chunks":['
        first = True
        
        for event in agent.stream(message):
            if "data" in event:
                if not first:
                    yield ','
                yield json.dumps({"text": event["data"]})
                first = False
        
        yield ']}'
    
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": generate(),
    }
```

### 3.3 SAM Template

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Parameters:
  Environment:
    Type: String
    Default: development
    AllowedValues: [development, staging, production]

Globals:
  Function:
    Timeout: 300  # 5 minutes for agent operations
    MemorySize: 1024
    Runtime: python3.11
    Environment:
      Variables:
        ENVIRONMENT: !Ref Environment
        SESSION_BUCKET: !Ref SessionBucket
        SESSION_PREFIX: !Sub "${Environment}/"

Resources:
  AgentFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: lambda_handler.handler
      CodeUri: .
      Policies:
        - S3CrudPolicy:
            BucketName: !Ref SessionBucket
        - Statement:
            - Effect: Allow
              Action:
                - bedrock:InvokeModel
                - bedrock:InvokeModelWithResponseStream
              Resource: "*"
      Events:
        Api:
          Type: Api
          Properties:
            Path: /chat
            Method: POST

  SessionBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "agent-sessions-${AWS::AccountId}-${Environment}"
      VersioningConfiguration:
        Status: Enabled

Outputs:
  ApiEndpoint:
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/chat"
```

### 3.4 Lambda Deployment Commands

```bash
# Build and deploy with SAM
sam build
sam deploy --guided  # First time
sam deploy            # Subsequent deployments

# Test locally with SAM
sam local invoke AgentFunction -e events/test_event.json

# View logs
sam logs -n AgentFunction --tail
```

---

## 4. Docker Deployment

### 4.1 Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY agents/ ./agents/
COPY tools/ ./tools/
COPY config/ ./config/
COPY main.py .

# Create non-root user
RUN useradd -m -u 1000 agent
USER agent

# Environment variables
ENV PYTHONUNBUFFERED=1
ENV ENVIRONMENT=production

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "from agents.main_agent import create_agent; create_agent()" || exit 1

# Entry point
CMD ["python", "main.py"]
```

### 4.2 Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  agent:
    build: .
    environment:
      - ENVIRONMENT=development
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - SESSION_DIR=/app/sessions
    volumes:
      - ./sessions:/app/sessions
      - ./logs:/app/logs
    ports:
      - "8000:8000"

  # Optional: Local API server
  api:
    build: .
    command: uvicorn api:app --host 0.0.0.0 --port 8000
    environment:
      - ENVIRONMENT=development
      - AWS_REGION=us-east-1
    ports:
      - "8000:8000"
    depends_on:
      - agent
```

### 4.3 FastAPI Container

```python
# api.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from agents.main_agent import create_agent
from config.settings import settings

app = FastAPI(title="Strands Agent API")

class ChatRequest(BaseModel):
    message: str
    session_id: str | None = None

class ChatResponse(BaseModel):
    response: str
    session_id: str

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    try:
        agent = create_agent(session_id=request.session_id)
        response = await agent.invoke_async(request.message)
        return ChatResponse(
            response=str(response),
            session_id=request.session_id or "default",
        )
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

### 4.4 Docker Commands

```bash
# Build image
docker build -t my-agent:latest .

# Run locally
docker run -it --rm \
    -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
    -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
    -e AWS_REGION=us-east-1 \
    my-agent:latest

# Push to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REGISTRY
docker tag my-agent:latest $ECR_REGISTRY/my-agent:latest
docker push $ECR_REGISTRY/my-agent:latest
```

---

## 5. AgentCore-Ready Patterns

### 5.1 Design for AgentCore Migration

Structure code to minimize changes when migrating to AgentCore:

```python
# agents/base.py
from abc import ABC, abstractmethod
from strands import Agent
from typing import Optional

class AgentBase(ABC):
    """Base class for AgentCore-ready agents."""
    
    def __init__(self, session_id: Optional[str] = None):
        self.session_id = session_id
        self._agent: Optional[Agent] = None
    
    @abstractmethod
    def create_tools(self) -> list:
        """Override to provide agent tools."""
        pass
    
    @abstractmethod
    def get_system_prompt(self) -> str:
        """Override to provide system prompt."""
        pass
    
    def create_agent(self) -> Agent:
        """Create agent instance."""
        from config.models import create_model
        from config.sessions import create_session_manager
        
        return Agent(
            model=create_model(),
            tools=self.create_tools(),
            system_prompt=self.get_system_prompt(),
            session_manager=create_session_manager(self.session_id) if self.session_id else None,
        )
    
    @property
    def agent(self) -> Agent:
        if self._agent is None:
            self._agent = self.create_agent()
        return self._agent
    
    def invoke(self, message: str, **kwargs) -> str:
        """Invoke agent with message."""
        result = self.agent(message, **kwargs)
        return str(result)
    
    async def invoke_async(self, message: str, **kwargs) -> str:
        """Async invocation."""
        result = await self.agent.invoke_async(message, **kwargs)
        return str(result)
```

### 5.2 Environment-Agnostic Factory

```python
# config/factory.py
import os
from typing import Optional
from strands import Agent
from strands.models import BedrockModel

def create_model():
    """Create model based on environment."""
    return BedrockModel(
        model_id=os.getenv(
            "STRANDS_MODEL_ID",
            "us.anthropic.claude-sonnet-4-20250514-v1:0"
        ),
        region_name=os.getenv("AWS_REGION", "us-east-1"),
    )

def create_session_manager(session_id: Optional[str]):
    """Create session manager based on environment."""
    if not session_id:
        return None
    
    # Future: AgentCore memory
    if os.environ.get("AGENTCORE_RUNTIME"):
        # from bedrock_agentcore.memory.integrations.strands import AgentCoreMemorySessionManager
        # return AgentCoreMemorySessionManager(...)
        pass
    
    # Production: S3
    if os.environ.get("SESSION_BUCKET"):
        from strands.session.s3_session_manager import S3SessionManager
        return S3SessionManager(
            session_id=session_id,
            bucket=os.environ["SESSION_BUCKET"],
        )
    
    # Development: File system
    from strands.session.file_session_manager import FileSessionManager
    return FileSessionManager(session_id=session_id)

def create_agent(session_id: Optional[str] = None, **kwargs) -> Agent:
    """Factory function for creating agents."""
    return Agent(
        model=create_model(),
        session_manager=create_session_manager(session_id),
        **kwargs,
    )
```

### 5.3 Tool Abstraction Layer

```python
# tools/abstractions.py
from abc import ABC, abstractmethod
from typing import Any, Dict
import os

class StorageBackend(ABC):
    """Abstract storage for tool data."""
    
    @abstractmethod
    def read(self, key: str) -> Any:
        pass
    
    @abstractmethod
    def write(self, key: str, data: Any) -> None:
        pass

class LocalStorage(StorageBackend):
    def __init__(self, base_path: str = "./data"):
        self.base_path = base_path
    
    def read(self, key: str) -> Any:
        import json
        with open(f"{self.base_path}/{key}.json") as f:
            return json.load(f)
    
    def write(self, key: str, data: Any) -> None:
        import json
        with open(f"{self.base_path}/{key}.json", "w") as f:
            json.dump(data, f)

class S3Storage(StorageBackend):
    def __init__(self, bucket: str, prefix: str = ""):
        import boto3
        self.bucket = bucket
        self.prefix = prefix
        self.s3 = boto3.client("s3")
    
    def read(self, key: str) -> Any:
        import json
        response = self.s3.get_object(Bucket=self.bucket, Key=f"{self.prefix}{key}")
        return json.loads(response["Body"].read())
    
    def write(self, key: str, data: Any) -> None:
        import json
        self.s3.put_object(
            Bucket=self.bucket,
            Key=f"{self.prefix}{key}",
            Body=json.dumps(data),
        )

def get_storage() -> StorageBackend:
    """Get appropriate storage backend."""
    if bucket := os.environ.get("DATA_BUCKET"):
        return S3Storage(bucket)
    return LocalStorage()
```

---

## 6. CI/CD Pipeline

### 6.1 GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Agent

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest
      
      - name: Run tests
        run: pytest tests/ -v

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Set up SAM
        uses: aws-actions/setup-sam@v2
      
      - name: Build
        run: sam build
      
      - name: Deploy
        run: sam deploy --no-confirm-changeset --no-fail-on-empty-changeset
```

---

## 7. Monitoring & Observability

### 7.1 OpenTelemetry Setup

```python
# config/telemetry.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
import os

def setup_telemetry():
    """Configure OpenTelemetry for agent tracing."""
    
    if not os.environ.get("OTEL_EXPORTER_OTLP_ENDPOINT"):
        return  # Skip if not configured
    
    provider = TracerProvider()
    processor = BatchSpanProcessor(OTLPSpanExporter())
    provider.add_span_processor(processor)
    trace.set_tracer_provider(provider)
```

### 7.2 Structured Logging

```python
# config/logging.py
import logging
import json
import sys

class JSONFormatter(logging.Formatter):
    """JSON log formatter for production."""
    
    def format(self, record):
        log_record = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
        }
        if record.exc_info:
            log_record["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_record)

def setup_logging(level: str = "INFO"):
    """Configure logging based on environment."""
    import os
    
    handler = logging.StreamHandler(sys.stdout)
    
    if os.environ.get("ENVIRONMENT") == "production":
        handler.setFormatter(JSONFormatter())
    else:
        handler.setFormatter(logging.Formatter(
            "%(levelname)s | %(name)s | %(message)s"
        ))
    
    logging.root.addHandler(handler)
    logging.root.setLevel(getattr(logging, level.upper()))
    
    # Strands logging
    logging.getLogger("strands").setLevel(logging.INFO)
```

---

## 8. Deployment Checklist

### 8.1 Pre-Deployment

- [ ] All tests pass locally
- [ ] Environment variables documented
- [ ] Session management configured
- [ ] Error handling implemented
- [ ] Retry logic with backoff
- [ ] Logging configured
- [ ] Dependencies pinned

### 8.2 Lambda Deployment

- [ ] IAM role with Bedrock permissions
- [ ] S3 bucket for sessions
- [ ] Memory size adequate (≥1024MB)
- [ ] Timeout sufficient (≥60s, ≤900s)
- [ ] VPC config if needed
- [ ] Function URL or API Gateway

### 8.3 Docker Deployment

- [ ] Non-root user in container
- [ ] Health check configured
- [ ] Resource limits set
- [ ] Secrets management (not in image)
- [ ] Logging to stdout/stderr

### 8.4 Production

- [ ] Monitoring/alerting configured
- [ ] Telemetry enabled
- [ ] Backup strategy for sessions
- [ ] Rate limiting/throttling
- [ ] Cost monitoring

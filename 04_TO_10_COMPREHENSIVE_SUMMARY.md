# 04-10 - Comprehensive Backend Analysis Summary

> **Purpose:** Complete analysis of remaining components for Python migration  
> **Date:** 2026-01-03

## Table of Contents

1. [Business Logic & Controllers](#04-business-logic--controllers)
2. [AI Providers System](#05-ai-providers-system)
3. [Tools & Executors](#06-tools--executors)
4. [Infrastructure Components](#07-infrastructure-components)
5. [Configuration Management](#08-configuration-management)
6. [Dependencies Mapping](#09-dependencies-mapping)
7. [Migration Plan](#10-migration-plan)

---

# 04 - Business Logic & Controllers

## Controller Architecture

### Flow Controller
**Purpose:** Orchestrates the entire execution lifecycle

**Key Responsibilities:**
1. **Flow Management** - Create, start, stop, delete flows
2. **Task Orchestration** - Queue and execute tasks
3. **Worker Pool Management** - Manage concurrent executions
4. **State Management** - Track execution state across restarts
5. **Event Broadcasting** - Publish updates via subscriptions

**Core Methods:**
```go
type FlowController interface {
    LoadFlows(ctx context.Context) error
    CreateFlow(ctx context.Context, input CreateFlowInput) (FlowWorker, error)
    GetFlow(ctx context.Context, flowID int64) (FlowWorker, error)
    DeleteFlow(ctx context.Context, flowID int64) error
    ListFlows(ctx context.Context) []FlowWorker
}
```

**Workers Pattern:**
- **FlowWorker** - Manages a single flow lifecycle
- **TaskWorker** - Executes individual tasks
- **SubtaskWorker** - Executes subtask with AI agents
- **AssistantWorker** - Manages assistant conversations

### Task Controller
**Purpose:** Manages task breakdown and execution

**Workflow:**
```
1. User input → CreateTask
   ↓
2. Generate subtasks (generator agent)
   ↓
3. For each subtask:
   - Create SubtaskWorker
   - Execute with primary_agent
   - Collect results
   ↓
4. Refine subtasks if needed (refiner agent)
   ↓
5. Generate final report (reporter agent)
```

**State Machines:**
```
Task:    created → running → [waiting] → finished/failed
Subtask: created → running → [waiting] → finished/failed
```

### Subtask Controller
**Purpose:** Execute individual subtasks with AI agents

**Execution Flow:**
1. **Prepare Context** - Gather task context, previous results
2. **Create MsgChain** - Initialize agent conversation
3. **Execute Agent** - Run primary_agent with tools
4. **Handle Tools** - Execute tool calls (search, terminal, code)
5. **Handle Input** - Wait for user input if needed
6. **Mark Done** - Update status and result

**Agent Types Used:**
- `primary_agent` - Main orchestrator
- `adviser` - Provides guidance
- `coder` - Writes/analyzes code
- `installer` - Installs software
- `pentester` - Performs security testing
- `searcher` - Web search
- `memorist` - Memory/vector store operations

---

## Data Flow

### Flow Creation
```
User Input
  ↓
FlowController.CreateFlow()
  ↓
Database: INSERT INTO flows
  ↓
Provider Selection (from config)
  ↓
Create FlowWorker
  ↓
Subscription: flowCreated event
  ↓
TaskController.CreateTask(input)
  ↓
TaskWorker spawned (goroutine)
  ↓
Generator Agent: Create subtasks list
  ↓
For each subtask:
    SubtaskWorker spawned
    ↓
    Primary Agent execution
    ↓
    Tool calls handled
    ↓
    Result collected
  ↓
Reporter Agent: Generate final report
  ↓
Task marked finished
  ↓
Subscription: taskUpdated event
```

### Tool Call Flow
```
Agent decides to use tool
  ↓
LLM returns tool_call in response
  ↓
ToolExecutor.Execute(name, args)
  ↓
Specific tool handler invoked:
  - terminal.go → Execute in Docker
  - browser.go → Scrape webpage
  - search.go → Query search engine
  - memory.go → Vector store search
  ↓
Tool result returned
  ↓
Add to message chain as tool response
  ↓
LLM continues with tool result
```

---

## Event System

### Subscriptions Controller
**Technology:** WebSocket via GraphQL subscriptions

**Event Types:**
- Flow events: created, updated, deleted
- Task events: created, updated
- Assistant events: created, updated, deleted
- Log events: messageLog, terminalLog, agentLog, searchLog added/updated

**Implementation:**
```go
type SubscriptionsController interface {
    PublishFlowCreated(flow *database.Flow)
    PublishTaskCreated(flowID int64, task *database.Task)
    PublishMessageLog(flowID int64, log *database.Msglog)
    // ... etc
    
    SubscribeToFlow(flowID int64) <-chan Event
}
```

**Usage in Frontend:**
```javascript
subscription {
  messageLogAdded(flowId: "42") {
    id
    type
    message
    result
    createdAt
  }
}
```

---

# 05 - AI Providers System

## Provider Interface

**File:** `backend/pkg/providers/provider.go`

**Core Interface:**
```go
type Provider interface {
    // Metadata
    Type() ProviderType
    Model(opt ProviderOptionsType) string
    
    // LLM Calls
    Call(ctx context.Context, opt ProviderOptionsType, prompt string) (string, error)
    CallJSON(ctx context.Context, opt ProviderOptionsType, prompt string, schema any) (json.RawMessage, error)
    Stream(ctx context.Context, opt ProviderOptionsType, messages []Message, tools []Tool, handler StreamHandler) error
    
    // Tool Calling
    SupportsToolCalling() bool
    SupportsStreaming() bool
}
```

## Supported Providers

### 1. OpenAI
**File:** `backend/pkg/providers/openai/`

**Models:**
- GPT-4o, GPT-4o-mini
- o1, o1-mini, o1-preview (reasoning models)
- o3-mini (new reasoning)

**Features:**
- Streaming ✅
- Tool calling ✅
- JSON mode ✅
- Vision ✅
- Reasoning (for o1/o3) ✅

**Configuration:**
```json
{
  "api_key": "sk-...",
  "base_url": "https://api.openai.com/v1",
  "organization": "org-...",
  "agents": {
    "primary_agent": {
      "model": "gpt-4o",
      "temperature": 0.7,
      "max_tokens": 4096
    }
  }
}
```

### 2. Anthropic Claude
**File:** `backend/pkg/providers/anthropic/`

**Models:**
- Claude 3.5 Sonnet
- Claude 3 Opus, Sonnet, Haiku

**Features:**
- Streaming ✅
- Tool calling ✅
- JSON mode ✅ (via prompt engineering)
- Vision ✅
- Extended context (200K tokens)

**Prompt Caching:** Automatic for repeated context

### 3. Google Gemini
**File:** `backend/pkg/providers/gemini/`

**Models:**
- Gemini 1.5 Pro, Flash
- Gemini 2.0 Flash (experimental)

**Features:**
- Streaming ✅
- Tool calling ✅
- JSON mode ✅
- Vision ✅
- 2M token context

### 4. AWS Bedrock
**File:** `backend/pkg/providers/bedrock/`

**Supported Models:**
- Anthropic Claude (via Bedrock)
- Amazon Titan
- Meta Llama
- Cohere Command

**Authentication:** AWS credentials (IAM)

### 5. Ollama
**File:** `backend/pkg/providers/ollama/`

**Purpose:** Run local models

**Supported:**
- Llama 3/3.1/3.2
- Qwen 2.5
- DeepSeek
- Mistral
- Custom GGUF models

**Configuration:**
```json
{
  "base_url": "http://localhost:11434",
  "agents": {
    "primary_agent": {
      "model": "qwen2.5:32b",
      "temperature": 0.7
    }
  }
}
```

### 6. Custom (OpenAI-compatible)
**File:** `backend/pkg/providers/custom/`

**Purpose:** Any OpenAI-compatible API

**Examples:**
- OpenRouter
- Together AI
- DeepInfra
- vLLM deployments
- LiteLLM proxy

## Streaming Implementation

**Pattern:**
```go
handler := func(ctx context.Context, chunk StreamChunk) error {
    switch chunk.Type {
    case ChunkTypeContent:
        // Stream text content
        publishUpdate(chunk.Content)
    case ChunkTypeToolCall:
        // Execute tool
        result := executeTool(chunk.ToolCall)
        return result
    }
    return nil
}

err := provider.Stream(ctx, messages, tools, handler)
```

**Chunk Types:**
- `content` - Text response
- `tool_call` - Tool invocation
- `thinking` - Reasoning (o1/o3 models)

## Token Usage Tracking

**Stored in msgchains table:**
```go
type Msgchain struct {
    UsageIn  int64  // Input tokens
    UsageOut int64  // Output tokens
    // ...
}
```

**Cost Calculation:**
```go
inputCost := (usageIn / 1000000.0) * modelPricePerMInput
outputCost := (usageOut / 1000000.0) * modelPricePerMOutput
totalCost := inputCost + outputCost
```

---

# 06 - Tools & Executors

## Tool System Architecture

### Tool Interface
```go
type Tool interface {
    Handle(ctx context.Context, name string, args json.RawMessage) (string, error)
    IsAvailable() bool
}
```

### Tool Executor
**File:** `backend/pkg/tools/executor.go`

**Purpose:** Manages tool lifecycle and execution

**Executor Types:**
1. **PrimaryExecutor** - For primary_agent (all tools)
2. **AssistantExecutor** - For assistant mode
3. **CoderExecutor** - For coder agent (code tools)
4. **PentesterExecutor** - For pentester agent (terminal, browser)
5. **SearcherExecutor** - For searcher agent (search tools)
6. **InstallerExecutor** - For installer agent (terminal, maintenance)

## Built-in Tools

### 1. Terminal Tool
**File:** `tools/terminal.go`

**Purpose:** Execute commands in Docker container

**Functions:**
- `terminal` - Run command and return output
- `file` - Read/write files

**Example:**
```json
{
  "name": "terminal",
  "arguments": {
    "command": "nmap -sV target.com",
    "timeout": 300
  }
}
```

**Implementation:**
- Uses Docker SDK to exec in container
- Streams stdout/stderr to termlogs table
- Supports interactive commands (limited)

### 2. Browser Tool
**File:** `tools/browser.go`

**Purpose:** Web scraping and screenshots

**Functions:**
- `browser` - Navigate and scrape webpage

**Example:**
```json
{
  "name": "browser",
  "arguments": {
    "url": "https://example.com",
    "action": "screenshot"
  }
}
```

**Dependencies:**
- External scraper service (configurable)
- Stores screenshots in DATA_DIR

### 3. Search Tools

**DuckDuckGo** (`tools/duckduckgo.go`):
- Free, no API key
- HTML scraping
- Limited results

**Google** (`tools/google.go`):
- Requires API key + CX key
- Custom Search JSON API
- 100 queries/day free

**Tavily** (`tools/tavily.go`):
- AI-optimized search
- Returns summarized results
- Requires API key

**Perplexity** (`tools/perplexity.go`):
- AI-powered search
- Returns answer + sources
- Uses Perplexity API

**SearXNG** (`tools/searxng.go`):
- Self-hosted metasearch
- Privacy-focused
- No API key needed

**Traversaal** (`tools/traversaal.go`):
- Multi-source aggregation
- Requires API key

### 4. Memory Tool
**File:** `tools/memory.go`

**Purpose:** Vector store operations

**Functions:**
- `search_in_memory` - Similarity search
- `store_in_memory` - Add embedding

**Implementation:**
- Uses pgvector for storage
- Embeddings via OpenAI/custom
- Stores in `langchain_pg_embedding` table

**Example:**
```json
{
  "name": "search_in_memory",
  "arguments": {
    "query": "How to exploit SQL injection?",
    "top_k": 5
  }
}
```

### 5. Code Tool
**File:** `tools/code.go`

**Purpose:** Code storage and retrieval

**Functions:**
- `store_code` - Store code snippet
- `search_code` - Search stored code

**Use Case:** Remember exploit scripts, configurations

### 6. Graphiti Search
**File:** `tools/graphiti_search.go`

**Purpose:** Knowledge graph queries

**Integration:** External Graphiti service

**Example:**
```json
{
  "name": "graphiti_search",
  "arguments": {
    "query": "Find all vulnerabilities related to Apache"
  }
}
```

### 7. Agent Tools

**Functions that call other agents:**
- `adviser` - Ask advice agent
- `coder` - Ask coder agent
- `installer` - Ask installer agent
- `pentester` - Ask pentester agent
- `searcher` - Ask searcher agent
- `memorist` - Ask memorist agent

**Example:**
```json
{
  "name": "searcher",
  "arguments": {
    "task": "Find recent CVEs for nginx"
  }
}
```

## Tool Registry
**File:** `tools/registry.go`

**Purpose:** Central tool definitions

**Schema Definition:**
```go
var registryDefinitions = map[string]llms.FunctionDefinition{
    TerminalToolName: {
        Name: "terminal",
        Description: "Execute a command in the terminal...",
        Parameters: json.RawMessage(`{
            "type": "object",
            "properties": {
                "command": {
                    "type": "string",
                    "description": "Command to execute"
                }
            },
            "required": ["command"]
        }`),
    },
    // ... more tools
}
```

## External Functions

**Custom Tool Support:**
```json
{
  "token": "secret-token",
  "functions": [
    {
      "name": "custom_exploit",
      "url": "https://custom-service.com/exploit",
      "timeout": 120,
      "schema": {
        "type": "object",
        "properties": {
          "target": {"type": "string"}
        }
      }
    }
  ]
}
```

**Execution:**
1. LLM calls custom function
2. Executor POSTs to URL with args
3. Returns result to LLM

---

# 07 - Infrastructure Components

## Docker Integration

**File:** `backend/pkg/docker/client.go`

### Container Lifecycle

**Create Container:**
```go
cnt, err := docker.SpawnContainer(
    ctx,
    name,
    containerType,
    flowID,
    &container.Config{
        Image: "debian:latest",
        Entrypoint: []string{"tail", "-f", "/dev/null"},
    },
    &container.HostConfig{
        CapAdd: []string{"NET_RAW", "NET_ADMIN"},
    },
)
```

**Execute Command:**
```go
execConfig := types.ExecConfig{
    AttachStdout: true,
    AttachStderr: true,
    Cmd: []string{"bash", "-c", command},
}
execID, err := docker.ContainerExecCreate(ctx, containerID, execConfig)
resp, err := docker.ContainerExecAttach(ctx, execID, types.ExecStartCheck{})
```

**Container Naming:**
```
pentagi-flow-{flowID}-primary
pentagi-flow-{flowID}-secondary-{n}
```

### Capabilities

**Default:** `NET_RAW` (for ping, nmap)

**Optional:** `NET_ADMIN` (for iptables, network config)

### Volume Mounts

**Work Directory:**
```
Host: ${DOCKER_WORK_DIR}
Container: /workspace
```

## Queue System

**File:** `backend/pkg/queue/queue.go`

**Purpose:** Task scheduling (currently unused, planned for future)

**Implementation:** Channel-based queue

**Future Use:**
- Distribute tasks across workers
- Rate limiting
- Priority queues

## Observability

### OpenTelemetry

**File:** `backend/pkg/observability/otelclient.go`

**Components:**
1. **Traces** - Distributed tracing
2. **Metrics** - Performance metrics
3. **Logs** - Structured logging

**Span Hierarchy:**
```
flow-worker
  ├─ task-execution
  │   ├─ generator-agent
  │   ├─ subtask-execution
  │   │   ├─ primary-agent
  │   │   │   ├─ llm-call
  │   │   │   ├─ tool-execution
  │   │   │   └─ llm-call
  │   │   └─ result-collection
  │   └─ reporter-agent
  └─ cleanup
```

**Metrics Collected:**
- CPU usage
- Memory usage
- Goroutine count
- Request latency
- Token usage
- Error rate

### Langfuse

**File:** `backend/pkg/observability/langfuse/`

**Purpose:** LLM-specific observability

**Tracked:**
- LLM calls (model, tokens, latency)
- Cost per request
- Prompt versions
- User feedback
- Trace sessions

**Trace Structure:**
```
Trace (flow)
  ├─ Span (task)
  │   ├─ Span (agent)
  │   │   ├─ Generation (LLM call)
  │   │   │   ├─ Input: messages
  │   │   │   ├─ Output: response
  │   │   │   ├─ Tokens: in/out
  │   │   │   └─ Cost: $0.02
  │   │   └─ Span (tool)
  │   └─ Span (reporter)
  └─ Metrics
```

## Terminal System

**File:** `backend/pkg/terminal/output.go`

**Purpose:** Format terminal output

**Features:**
- Color codes (ANSI)
- Progress indicators
- Table formatting

**Usage:** Primarily for installer TUI

---

# 08 - Configuration Management

## Environment Variables

**File:** `backend/pkg/config/config.go`

### General Settings
```bash
DATABASE_URL=postgres://user:pass@host:5432/db
DEBUG=false
DATA_DIR=./data
ASK_USER=false  # Enable interactive prompts
```

### Server Settings
```bash
SERVER_PORT=8080
SERVER_HOST=0.0.0.0
SERVER_USE_SSL=false
SERVER_SSL_KEY=/path/to/key.pem
SERVER_SSL_CRT=/path/to/cert.pem
STATIC_URL=http://localhost:3000  # Frontend dev server
STATIC_DIR=./fe  # Built frontend
CORS_ORIGINS=*  # Or comma-separated list
PUBLIC_URL=https://pentagi.example.com  # For OAuth callbacks
```

### Docker Settings
```bash
DOCKER_INSIDE=false  # Running inside Docker?
DOCKER_NET_ADMIN=false  # Grant NET_ADMIN capability?
DOCKER_SOCKET=/var/run/docker.sock
DOCKER_NETWORK=pentagi-net  # Custom network
DOCKER_PUBLIC_IP=0.0.0.0
DOCKER_WORK_DIR=/tmp/pentagi-workspace
DOCKER_DEFAULT_IMAGE=debian:latest
DOCKER_DEFAULT_IMAGE_FOR_PENTEST=vxcontrol/kali-linux
```

### LLM Providers

**OpenAI:**
```bash
OPEN_AI_KEY=sk-...
OPEN_AI_SERVER_URL=https://api.openai.com/v1
```

**Anthropic:**
```bash
ANTHROPIC_API_KEY=sk-ant-...
ANTHROPIC_SERVER_URL=https://api.anthropic.com/v1
```

**Google Gemini:**
```bash
GEMINI_API_KEY=AI...
GEMINI_SERVER_URL=https://generativelanguage.googleapis.com
```

**AWS Bedrock:**
```bash
BEDROCK_REGION=us-east-1
BEDROCK_ACCESS_KEY_ID=AKIA...
BEDROCK_SECRET_ACCESS_KEY=...
BEDROCK_SESSION_TOKEN=...  # Optional
BEDROCK_SERVER_URL=...  # Optional custom endpoint
```

**Ollama:**
```bash
OLLAMA_SERVER_URL=http://localhost:11434
OLLAMA_SERVER_CONFIG_PATH=/path/to/config.json
```

**Custom Provider:**
```bash
LLM_SERVER_URL=https://custom-llm.com/v1
LLM_SERVER_KEY=...
LLM_SERVER_MODEL=custom-model
LLM_SERVER_CONFIG_PATH=/path/to/config.json
LLM_SERVER_LEGACY_REASONING=false
```

### Embeddings
```bash
EMBEDDING_URL=https://api.openai.com/v1
EMBEDDING_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_PROVIDER=openai
EMBEDDING_STRIP_NEW_LINES=true
EMBEDDING_BATCH_SIZE=512
```

### Search Engines
```bash
# DuckDuckGo (no key needed)
DUCKDUCKGO_ENABLED=true

# Google
GOOGLE_API_KEY=...
GOOGLE_CX_KEY=...  # Custom Search Engine ID
GOOGLE_LR_KEY=lang_en

# Tavily
TAVILY_API_KEY=tvly-...

# Traversaal
TRAVERSAAL_API_KEY=...

# Perplexity
PERPLEXITY_API_KEY=pplx-...
PERPLEXITY_MODEL=sonar
PERPLEXITY_CONTEXT_SIZE=low  # low|medium|high

# SearXNG
SEARXNG_URL=http://localhost:8080
SEARXNG_CATEGORIES=general
SEARXNG_LANGUAGE=en
SEARXNG_SAFESEARCH=0  # 0=off, 1=moderate, 2=strict
SEARXNG_TIME_RANGE=  # Optional: day, week, month, year
```

### Browser/Scraper
```bash
SCRAPER_PUBLIC_URL=http://scraper:3000
SCRAPER_PRIVATE_URL=http://scraper:3000
```

### OAuth
```bash
OAUTH_GOOGLE_CLIENT_ID=...
OAUTH_GOOGLE_CLIENT_SECRET=...
OAUTH_GITHUB_CLIENT_ID=...
OAUTH_GITHUB_CLIENT_SECRET=...
```

### Security
```bash
COOKIE_SIGNING_SALT=random-secret-32-chars-min
```

### Observability
```bash
# OpenTelemetry
OTEL_HOST=otel-collector:4317

# Langfuse
LANGFUSE_BASE_URL=https://cloud.langfuse.com
LANGFUSE_PROJECT_ID=...
LANGFUSE_PUBLIC_KEY=pk-...
LANGFUSE_SECRET_KEY=sk-...
```

### Graphiti (Knowledge Graph)
```bash
GRAPHITI_ENABLED=false
GRAPHITI_TIMEOUT=30
GRAPHITI_URL=http://graphiti:8000
```

### Summarizer Settings
```bash
SUMMARIZER_PRESERVE_LAST=true
SUMMARIZER_USE_QA=true
SUMMARIZER_SUM_MSG_HUMAN_IN_QA=false
SUMMARIZER_LAST_SEC_BYTES=51200
SUMMARIZER_MAX_BP_BYTES=16384
SUMMARIZER_MAX_QA_SECTIONS=10
SUMMARIZER_MAX_QA_BYTES=65536
SUMMARIZER_KEEP_QA_SECTIONS=1
```

### Assistant Settings
```bash
ASSISTANT_USE_AGENTS=false
ASSISTANT_SUMMARIZER_PRESERVE_LAST=true
ASSISTANT_SUMMARIZER_LAST_SEC_BYTES=76800
ASSISTANT_SUMMARIZER_MAX_BP_BYTES=16384
ASSISTANT_SUMMARIZER_MAX_QA_SECTIONS=7
ASSISTANT_SUMMARIZER_MAX_QA_BYTES=76800
ASSISTANT_SUMMARIZER_KEEP_QA_SECTIONS=3
```

### Network
```bash
PROXY_URL=http://proxy:3128  # Optional HTTP proxy
EXTERNAL_SSL_CA_PATH=/path/to/ca.crt  # Custom CA
EXTERNAL_SSL_INSECURE=false  # Skip SSL verification (insecure!)
```

### Installation
```bash
INSTALLATION_ID=auto-generated-uuid
LICENSE_KEY=...  # Optional PentAGI Cloud license
```

## Example .env File

```bash
# Database
DATABASE_URL=postgres://pentagiuser:pentagipass@localhost:5432/pentagidb?sslmode=disable

# Server
SERVER_PORT=8080
DEBUG=false
DATA_DIR=./data

# Docker
DOCKER_SOCKET=/var/run/docker.sock
DOCKER_DEFAULT_IMAGE=debian:latest

# LLM Provider (choose one or more)
OPEN_AI_KEY=sk-...
# ANTHROPIC_API_KEY=sk-ant-...
# GEMINI_API_KEY=AI...

# Embeddings
EMBEDDING_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small

# Search (optional)
DUCKDUCKGO_ENABLED=true
# TAVILY_API_KEY=tvly-...

# Security
COOKIE_SIGNING_SALT=$(openssl rand -hex 32)

# OAuth (optional)
# OAUTH_GOOGLE_CLIENT_ID=...
# OAUTH_GOOGLE_CLIENT_SECRET=...

# Observability (optional)
# OTEL_HOST=localhost:4317
# LANGFUSE_BASE_URL=https://cloud.langfuse.com
```

---

# 09 - Dependencies Mapping

## Core Dependencies

| Go Library | Purpose | Python Alternative |
|-----------|---------|-------------------|
| **HTTP/API** |||
| gin-gonic/gin | HTTP framework | FastAPI |
| 99designs/gqlgen | GraphQL server | Strawberry GraphQL |
| gorilla/websocket | WebSocket | python-socketio / websockets |
| swaggo/swag | Swagger docs | FastAPI (built-in) |
| **Database** |||
| lib/pq | PostgreSQL driver | psycopg3 / asyncpg |
| jinzhu/gorm | ORM | SQLAlchemy 2.0 |
| sqlc | Type-safe SQL | SQLAlchemy + Alembic |
| pressly/goose | Migrations | Alembic |
| pgvector/pgvector-go | Vector search | pgvector-python |
| **AI/LLM** |||
| vxcontrol/langchaingo | LLM abstraction | langchain |
| (custom) | OpenAI | openai-python |
| (custom) | Anthropic | anthropic-python |
| (custom) | Google AI | google-generativeai |
| aws-sdk-go-v2 | AWS Bedrock | boto3 |
| ollama/ollama | Ollama | ollama-python |
| **Docker** |||
| docker/docker | Docker SDK | docker-py |
| creack/pty | PTY for containers | python-pty |
| **Observability** |||
| opentelemetry | OTEL SDK | opentelemetry-python |
| (custom langfuse) | Langfuse | langfuse-python |
| sirupsen/logrus | Structured logging | structlog / loguru |
| **Templates** |||
| (custom) | Jinja2-like | jinja2 |
| **Authentication** |||
| golang-jwt/jwt | JWT | python-jose |
| coreos/go-oidc | OIDC | python-jose / authlib |
| golang.org/x/crypto | Bcrypt | bcrypt / passlib |
| **Testing** |||
| stretchr/testify | Assertions | pytest |
| **Configuration** |||
| caarlos0/env | Env parsing | pydantic-settings |
| joho/godotenv | .env loading | python-dotenv |
| **Utilities** |||
| google/uuid | UUID generation | uuid (stdlib) |
| hashicorp/golang-lru | LRU cache | cachetools |
| **TUI (Installer)** |||
| charmbracelet/bubbletea | TUI framework | textual / rich |
| charmbracelet/lipgloss | Styling | rich |
| charmbracelet/bubbles | Components | textual widgets |

## Detailed Migration Notes

### HTTP Framework: Gin → FastAPI

**Similarity:** Very high

**Gin:**
```go
r := gin.Default()
r.GET("/flows", func(c *gin.Context) {
    flows := getFlows()
    c.JSON(200, flows)
})
```

**FastAPI:**
```python
app = FastAPI()

@app.get("/flows")
async def get_flows():
    flows = await flow_service.list_flows()
    return flows
```

### GraphQL: gqlgen → Strawberry

**Gin + gqlgen:**
```go
// Schema in schema.graphqls
type Query {
  flows: [Flow!]!
}

// Resolver in Go
func (r *queryResolver) Flows(ctx context.Context) ([]*model.Flow, error) {
    return r.flowService.ListFlows(ctx)
}
```

**FastAPI + Strawberry:**
```python
import strawberry
from strawberry.fastapi import GraphQLRouter

@strawberry.type
class Query:
    @strawberry.field
    async def flows(self, info) -> List[Flow]:
        return await flow_service.list_flows()

schema = strawberry.Schema(query=Query)
app.include_router(GraphQLRouter(schema), prefix="/graphql")
```

### Database: GORM + sqlc → SQLAlchemy

**Go (sqlc):**
```sql
-- name: GetFlows :many
SELECT * FROM flows WHERE deleted_at IS NULL;
```

```go
flows, err := queries.GetFlows(ctx)
```

**Python (SQLAlchemy 2.0):**
```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_flows(session: AsyncSession) -> List[Flow]:
    stmt = select(Flow).where(Flow.deleted_at.is_(None))
    result = await session.execute(stmt)
    return result.scalars().all()
```

### AI Providers: Custom → LangChain

**Go (custom):**
```go
response, err := provider.Call(ctx, pconfig.OptionsTypePrimaryAgent, prompt)
```

**Python (LangChain):**
```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Factory pattern
def get_llm(provider_type: str, config: dict):
    if provider_type == "openai":
        return ChatOpenAI(**config)
    elif provider_type == "anthropic":
        return ChatAnthropic(**config)
    # etc.

llm = get_llm(flow.model_provider_type, config)
response = await llm.ainvoke(messages)
```

### Docker: docker-sdk-go → docker-py

**Go:**
```go
resp, err := docker.ContainerExecCreate(ctx, containerID, types.ExecConfig{
    Cmd: []string{"bash", "-c", command},
})
```

**Python:**
```python
import docker

client = docker.from_env()
exec_result = client.containers.get(container_id).exec_run(
    cmd=["bash", "-c", command],
    stdout=True,
    stderr=True
)
```

---

# 10 - Migration Plan

## Phase 1: Foundation (Weeks 1-2)

### Week 1: Database & Models
- [ ] Set up SQLAlchemy 2.0 + Alembic
- [ ] Create all table models
- [ ] Port all enums
- [ ] Create migration scripts
- [ ] Set up pgvector extension
- [ ] Verify data integrity

### Week 2: Core API
- [ ] Set up FastAPI project structure
- [ ] Implement authentication (JWT)
- [ ] Create REST endpoints
- [ ] Set up Strawberry GraphQL
- [ ] Implement basic CRUD operations
- [ ] Add WebSocket support

**Validation:**
- All API endpoints return correct data
- Authentication works
- WebSocket connections established

---

## Phase 2: Business Logic (Weeks 3-5)

### Week 3: Controllers
- [ ] Port FlowController
- [ ] Port TaskController
- [ ] Port SubtaskController
- [ ] Implement worker pattern (asyncio)
- [ ] Create event system (WebSocket broadcasts)

### Week 4: AI Providers
- [ ] Create provider interface
- [ ] Implement OpenAI provider
- [ ] Implement Anthropic provider
- [ ] Implement Gemini provider
- [ ] Add streaming support
- [ ] Implement tool calling

### Week 5: Tool System
- [ ] Port tool interface
- [ ] Implement terminal tool (Docker)
- [ ] Implement search tools
- [ ] Implement memory tool (pgvector)
- [ ] Implement browser tool
- [ ] Create tool executor

**Validation:**
- Simple flows execute end-to-end
- Providers switch correctly
- Tools execute properly

---

## Phase 3: Advanced Features (Weeks 6-8)

### Week 6: Agents & Prompts
- [ ] Implement agent handlers
- [ ] Port prompt templates (Jinja2)
- [ ] Implement summarization
- [ ] Add chain management
- [ ] Implement context preparation

### Week 7: Infrastructure
- [ ] Add OpenTelemetry
- [ ] Integrate Langfuse
- [ ] Implement logging
- [ ] Add metrics collection
- [ ] Performance optimization

### Week 8: Additional Providers
- [ ] Bedrock integration
- [ ] Ollama integration
- [ ] Custom provider support
- [ ] Provider testing framework

**Validation:**
- Complex flows with multiple subtasks work
- Observability data collected
- All providers functional

---

## Phase 4: Testing & Migration (Weeks 9-10)

### Week 9: Testing
- [ ] Unit tests (pytest)
- [ ] Integration tests
- [ ] E2E tests
- [ ] Load testing
- [ ] Security audit

### Week 10: Data Migration
- [ ] Export data from Go version
- [ ] Import data to Python version
- [ ] Verify data integrity
- [ ] User acceptance testing
- [ ] Performance tuning

**Validation:**
- All tests passing
- Performance acceptable
- Data migrated successfully

---

## Phase 5: Deployment (Week 11-12)

### Week 11: Deployment Prep
- [ ] Containerization (Docker)
- [ ] CI/CD pipelines
- [ ] Documentation
- [ ] Deployment guides
- [ ] Monitoring setup

### Week 12: Launch
- [ ] Staging deployment
- [ ] Production deployment
- [ ] Monitoring & alerts
- [ ] Bug fixes
- [ ] User training

---

## Complexity Estimates

| Component | Complexity | Effort | Risk |
|-----------|-----------|---------|------|
| Database Models | ⭐ Easy | 1 week | Low |
| REST API | ⭐ Easy | 1 week | Low |
| GraphQL API | ⭐⭐ Medium | 1 week | Medium |
| Authentication | ⭐⭐ Medium | 3 days | Low |
| Controllers | ⭐⭐⭐ Medium-Hard | 2 weeks | Medium |
| AI Providers | ⭐⭐⭐⭐ Hard | 3 weeks | High |
| Tool System | ⭐⭐⭐⭐ Hard | 2 weeks | High |
| Streaming | ⭐⭐⭐ Medium-Hard | 1 week | Medium |
| Docker Integration | ⭐⭐ Medium | 1 week | Low |
| Observability | ⭐⭐ Medium | 1 week | Low |
| Prompts & Templates | ⭐⭐ Medium | 3 days | Low |
| WebSockets | ⭐⭐⭐ Medium-Hard | 1 week | Medium |
| TUI Installer | ⭐⭐⭐⭐ Hard | 2 weeks | Medium |

**Total Estimated Time:** 10-12 weeks with 2-3 developers

---

## Risk Mitigation

### High-Risk Areas

1. **AI Provider Streaming**
   - Risk: Complex async handling
   - Mitigation: Thorough testing with all providers
   - Fallback: Synchronous mode

2. **Tool Execution in Docker**
   - Risk: Container management complexity
   - Mitigation: Use established docker-py patterns
   - Fallback: Limit tool capabilities initially

3. **WebSocket Subscriptions**
   - Risk: Connection stability, scaling
   - Mitigation: Use proven libraries (Strawberry subscriptions)
   - Fallback: Polling mode

4. **Data Migration**
   - Risk: Data loss, corruption
   - Mitigation: Extensive testing, backups
   - Fallback: Parallel run period

### Testing Strategy

**Unit Tests:**
- All models
- All services
- All utilities
- Coverage target: 80%+

**Integration Tests:**
- API endpoints
- Database operations
- Provider integrations
- Tool executions

**E2E Tests:**
- Complete flow execution
- All agent types
- All tool types
- Error scenarios

**Performance Tests:**
- Concurrent flows
- Large context handling
- Memory usage
- Database query performance

---

## Post-Migration

### Monitoring

**Metrics to Track:**
- API response times
- Flow completion rate
- Provider latency
- Token usage
- Error rate
- Memory usage

**Alerts:**
- API downtime
- Database connection loss
- Provider failures
- High error rate
- Memory leaks

### Optimization Opportunities

1. **Database:**
   - Connection pooling (pgbouncer)
   - Query optimization
   - Index tuning

2. **Caching:**
   - Redis for session storage
   - Provider response caching
   - Prompt template caching

3. **Scaling:**
   - Horizontal scaling (multiple workers)
   - Background task queue (Celery)
   - Load balancing

4. **Performance:**
   - Async everywhere
   - Batch database operations
   - Lazy loading

---

## Success Criteria

### Functional Requirements
✅ All existing features work in Python version
✅ API compatibility maintained
✅ Data successfully migrated
✅ All tests passing

### Non-Functional Requirements
✅ Response time < 2x Go version
✅ Memory usage < 1.5x Go version
✅ Supports same concurrent load
✅ 99.9% uptime

### User Acceptance
✅ Users can perform all previous tasks
✅ No workflow disruptions
✅ Improved maintainability
✅ Better developer experience

---

## Conclusion

This migration from Go to Python is **feasible** but requires:
- **Time:** 10-12 weeks
- **Team:** 2-3 experienced Python/FastAPI developers
- **Focus:** Careful testing, especially AI providers and tool system
- **Risk Management:** Parallel run period, comprehensive testing

**Recommended Approach:**
1. Start with core (database, API)
2. Add providers incrementally
3. Test extensively at each phase
4. Maintain feature parity
5. Optimize after migration

**Key Success Factors:**
- Strong testing strategy
- Incremental migration
- Performance monitoring
- User feedback loop

---

**End of Comprehensive Analysis**

**Documents Created:**
1. ✅ 01_PROJECT_STRUCTURE.md
2. ✅ 02_DATA_MODELS.md  
3. ✅ 03_API_AND_GRAPHQL_REFERENCE.md
4. ✅ 04-10_COMPREHENSIVE_ANALYSIS_SUMMARY.md (This document)

**Total Analysis Coverage:** ~90% of backend codebase
**Ready for Migration:** Yes, with plan provided

# 01 - Project Structure Analysis - PentAGI Backend

> **Project:** PentAGI Backend (Go)  
> **Purpose:** Comprehensive analysis for Python migration  
> **Date:** 2026-01-03

## 📚 Table of Contents

1. [Overview](#overview)
2. [Complete File Tree](#complete-file-tree)
3. [Directory Structure Description](#directory-structure-description)
4. [Entry Points](#entry-points)
5. [Bootstrap Sequence](#bootstrap-sequence)

---

## Overview

PentAGI is an AI-powered autonomous penetration testing platform built with Go. The backend consists of **388 Go files** organized into a modular structure with clear separation of concerns.

**Technology Stack:**
- **Language:** Go 1.24.0
- **HTTP Framework:** Gin (gin-gonic/gin)
- **GraphQL:** gqlgen (99designs/gqlgen)
- **Database:** PostgreSQL with pgvector extension
- **ORM:** GORM + sqlc for type-safe queries
- **Container Management:** Docker SDK
- **Observability:** OpenTelemetry + Langfuse
- **AI Providers:** OpenAI, Anthropic, Google Gemini, AWS Bedrock, Ollama, Custom

---

## Complete File Tree

```
backend/
├── go.mod                          # Go module definition with 71 dependencies
├── go.sum                          # Dependency checksums
│
├── cmd/                            # Entry point executables
│   ├── pentagi/                    # Main application entry point
│   │   ├── main.go                # Server bootstrap and initialization
│   │   └── tools.go               # Tool registration utilities
│   │
│   ├── ctester/                    # Controller testing utility
│   │   ├── main.go
│   │   ├── models.go
│   │   ├── report.go
│   │   └── utils.go
│   │
│   ├── etester/                    # Embeddings testing utility
│   │   ├── main.go
│   │   ├── flush.go               # Vector store cleanup
│   │   ├── info.go                # Embeddings info
│   │   ├── reindex.go             # Re-indexing operations
│   │   ├── search.go              # Search testing
│   │   ├── test.go                # Test runner
│   │   └── tester.go              # Core tester logic
│   │
│   ├── ftester/                    # Flow testing utility
│   │   ├── main.go
│   │   ├── mocks/                 # Mock implementations
│   │   │   ├── tools.go
│   │   │   └── logs.go
│   │   └── worker/                # Worker implementation
│   │       ├── interactive.go
│   │       ├── tester.go
│   │       ├── executor.go
│   │       └── args.go
│   │
│   └── installer/                  # Interactive installer (TUI)
│       ├── main.go
│       ├── main_test.go
│       ├── checker/               # System requirements checker
│       ├── files/                 # File management
│       ├── hardening/             # Security hardening
│       ├── loader/                # Configuration loader
│       ├── navigator/             # Navigation logic
│       ├── processor/             # Config processor
│       ├── state/                 # State management
│       └── wizard/                # TUI wizard (Charm Bubbletea)
│           ├── controller/
│           ├── locale/            # i18n support
│           ├── logger/
│           ├── models/            # Bubbletea models
│           │   ├── helpers/
│           │   ├── docker_form.go
│           │   ├── llm_provider_form.go
│           │   ├── tools.go
│           │   └── types.go
│           ├── registry/
│           ├── styles/
│           ├── terminal/
│           │   └── vt/            # Virtual terminal
│           └── window/
│
├── docs/                           # Project documentation
│   ├── chain_ast.md
│   ├── chain_summary.md
│   ├── charm.md
│   ├── config.md
│   ├── controller.md
│   ├── database.md
│   ├── docker.md
│   ├── flow_execution.md
│   ├── gemini.md
│   ├── installer.md
│   ├── langfuse.md
│   ├── observability.md
│   ├── ollama.md
│   ├── prompt_engineering_openai.md
│   ├── prompt_engineering_pentagi.md
│   └── installer/                 # Installer-specific docs
│       ├── charm-architecture-patterns.md
│       ├── charm-best-practices.md
│       ├── charm-core-libraries.md
│       ├── charm-debugging-guide.md
│       ├── charm-form-patterns.md
│       ├── charm-navigation-patterns.md
│       ├── checker-test-scenarios.md
│       ├── checker.md
│       ├── installer-architecture-design.md
│       ├── installer-base-screen.md
│       ├── installer-overview.md
│       ├── installer-troubleshooting.md
│       ├── processor-implementation.md
│       ├── processor-logic-implementation.md
│       ├── processor-wizard-integation.md
│       ├── processor.md
│       ├── reference-config-pattern.md
│       └── terminal-wizard-integation.md
│
├── fern/                           # API documentation generation
│   ├── fern.config.json
│   ├── generators.yml
│   └── langfuse/
│       └── openapi.yml
│
├── gqlgen/                         # GraphQL code generation config
│   └── gqlgen.yml
│
├── migrations/                     # Database migrations
│   ├── migrations.go              # Embedded migrations
│   └── sql/                       # SQL migration files
│       ├── 20241026_115120_initial_state.sql
│       ├── 20241130_183411_new_type_logs.sql
│       ├── 20241215_132209_new_user_role.sql
│       ├── 20241222_171335_msglog_result_format.sql
│       ├── 20250102_152614_flow_trace_id.sql
│       ├── 20250103_1215631_new_msgchain_type_fixer.sql
│       ├── 20250322_172248_new_searchengine_types.sql
│       ├── 20250331_200137_assistant_mode.sql
│       ├── 20250412_181121_subtask_context copy.sql
│       ├── 20250414_213004_thinking_msg_part.sql
│       ├── 20250419_100249_new_logs_indices.sql
│       ├── 20250420_120356_settings_permission.sql
│       ├── 20250701_094823_base_settings.sql
│       ├── 20250821_123456_add_searxng_search_type.sql
│       ├── 20250901_165149_remove_input_idx.sql
│       ├── 20251028_113516_remove_result_idx.sql
│       └── 20251102_194813_remove_description_idx.sql
│
├── pkg/                            # Core application packages
│   ├── cast/                       # Chain AST (Abstract Syntax Tree)
│   │   ├── chain_ast.go           # Message chain to AST conversion
│   │   ├── chain_ast_test.go
│   │   └── chain_data_test.go
│   │
│   ├── config/                     # Configuration management
│   │   └── config.go              # Environment-based config (90+ vars)
│   │
│   ├── controller/                 # Business logic controllers
│   │   ├── alog.go                # Agent log controller
│   │   ├── alogs.go               # Agent logs collection
│   │   ├── aslog.go               # Assistant log controller
│   │   ├── aslogs.go              # Assistant logs collection
│   │   ├── assistant.go           # Assistant lifecycle management
│   │   ├── context.go             # Context management
│   │   ├── flow.go                # Flow lifecycle management
│   │   ├── flows.go               # Flows collection
│   │   ├── msglog.go              # Message log controller
│   │   ├── msglogs.go             # Message logs collection
│   │   ├── screenshot.go          # Screenshot management
│   │   ├── screenshots.go         # Screenshots collection
│   │   ├── slog.go                # Search log controller
│   │   ├── slogs.go               # Search logs collection
│   │   ├── subtask.go             # Subtask execution
│   │   ├── subtasks.go            # Subtasks collection
│   │   ├── task.go                # Task execution orchestration
│   │   ├── tasks.go               # Tasks collection
│   │   ├── termlog.go             # Terminal log controller
│   │   ├── termlogs.go            # Terminal logs collection
│   │   ├── vslog.go               # Vector store log controller
│   │   └── vslogs.go              # Vector store logs collection
│   │
│   ├── csum/                       # Chain summarization
│   │   ├── chain_summary.go       # Message chain summarizer
│   │   ├── chain_summary_e2e_test.go
│   │   └── chain_summary_split_test.go
│   │
│   ├── database/                   # Data access layer
│   │   ├── database.go            # Database connection
│   │   ├── db.go                  # Core DB operations
│   │   ├── models.go              # Type definitions (generated by sqlc)
│   │   ├── querier.go             # Query interface
│   │   ├── agentlogs.sql.go       # Agent logs queries
│   │   ├── assistantlogs.sql.go   # Assistant logs queries
│   │   ├── assistants.sql.go      # Assistants queries
│   │   ├── containers.sql.go      # Containers queries
│   │   ├── flows.sql.go           # Flows queries
│   │   ├── msgchains.sql.go       # Message chains queries
│   │   ├── msglogs.sql.go         # Message logs queries
│   │   ├── prompts.sql.go         # Prompts queries
│   │   ├── providers.sql.go       # Providers queries
│   │   ├── roles.sql.go           # Roles queries
│   │   ├── screenshots.sql.go     # Screenshots queries
│   │   ├── searchlogs.sql.go      # Search logs queries
│   │   ├── subtasks.sql.go        # Subtasks queries
│   │   ├── tasks.sql.go           # Tasks queries
│   │   ├── termlogs.sql.go        # Terminal logs queries
│   │   ├── toolcalls.sql.go       # Tool calls queries
│   │   ├── users.sql.go           # Users queries
│   │   ├── vecstorelogs.sql.go    # Vector store logs queries
│   │   └── converter/             # Model converters
│   │
│   ├── docker/                     # Docker client wrapper
│   │   └── client.go              # Container lifecycle management
│   │
│   ├── graph/                      # GraphQL layer
│   │   ├── context.go             # GraphQL context
│   │   ├── generated.go           # Generated GraphQL code
│   │   ├── resolver.go            # Root resolver
│   │   ├── schema.graphqls        # GraphQL schema definition
│   │   ├── schema.resolvers.go    # Resolver implementations
│   │   ├── model/                 # GraphQL models
│   │   └── subscriptions/         # Real-time subscriptions
│   │
│   ├── graphiti/                   # Graphiti knowledge graph client
│   │   └── client.go
│   │
│   ├── observability/              # Telemetry and monitoring
│   │   ├── collector.go           # Metrics collector
│   │   ├── lfclient.go            # Langfuse client
│   │   ├── obs.go                 # Observability interface
│   │   ├── otelclient.go          # OpenTelemetry client
│   │   ├── langfuse/              # Langfuse integration
│   │   │   └── api/               # Langfuse API client
│   │   └── profiling/             # Performance profiling
│   │
│   ├── providers/                  # AI provider implementations
│   │   ├── provider.go            # Provider interface
│   │   ├── providers.go           # Provider controller
│   │   ├── assistant.go           # Assistant mode logic
│   │   ├── handlers.go            # Agent handlers
│   │   ├── helpers.go             # Helper functions
│   │   ├── helpers_test.go
│   │   ├── performer.go           # Single agent performer
│   │   ├── performers.go          # Multi-agent performers
│   │   ├── subtask_patch.go       # Subtask patching logic
│   │   ├── subtask_patch_test.go
│   │   ├── anthropic/             # Anthropic Claude integration
│   │   ├── bedrock/               # AWS Bedrock integration
│   │   ├── custom/                # Custom OpenAI-compatible APIs
│   │   ├── embeddings/            # Embedding providers
│   │   ├── gemini/                # Google Gemini integration
│   │   ├── ollama/                # Ollama local LLM integration
│   │   ├── openai/                # OpenAI integration
│   │   ├── pconfig/               # Provider configuration
│   │   ├── provider/              # Provider base types
│   │   └── tester/                # Provider testing framework
│   │
│   ├── queue/                      # Task queue system
│   │   ├── queue.go
│   │   └── queue_test.go
│   │
│   ├── schema/                     # JSON schema utilities
│   │   └── schema.go
│   │
│   ├── server/                     # HTTP server and routing
│   │   ├── middleware.go          # HTTP middlewares
│   │   ├── router.go              # Route definitions
│   │   ├── auth/                  # Authentication
│   │   ├── context/               # Request context
│   │   ├── docs/                  # Swagger docs
│   │   ├── logger/                # HTTP logger
│   │   ├── models/                # HTTP models
│   │   ├── oauth/                 # OAuth2 integration
│   │   ├── rdb/                   # Redis integration
│   │   ├── response/              # Response builders
│   │   └── services/              # HTTP services
│   │
│   ├── system/                     # System utilities
│   │   ├── host_id.go             # Host identification
│   │   ├── utils.go               # Common utilities
│   │   ├── utils_darwin.go        # macOS-specific
│   │   ├── utils_linux.go         # Linux-specific
│   │   ├── utils_test.go
│   │   └── utils_windows.go       # Windows-specific
│   │
│   ├── templates/                  # Prompt templates
│   │   ├── templates.go           # Template engine
│   │   ├── templates_test.go
│   │   ├── graphiti/              # Graphiti templates
│   │   ├── prompts/               # AI agent prompts
│   │   └── validator/             # Template validation
│   │
│   ├── terminal/                   # Terminal output utilities
│   │   └── output.go
│   │
│   ├── tools/                      # AI agent tools
│   │   ├── tools.go               # Tool interfaces and executor
│   │   ├── args.go                # Tool argument parsing
│   │   ├── context.go             # Tool context
│   │   ├── executor.go            # Tool executor implementation
│   │   ├── registry.go            # Tool registry
│   │   ├── browser.go             # Web browser tool
│   │   ├── browser_test.go
│   │   ├── code.go                # Code search/store tool
│   │   ├── memory.go              # Memory/vector store tool
│   │   ├── terminal.go            # Terminal execution tool
│   │   ├── guide.go               # Guide storage tool
│   │   ├── graphiti_search.go     # Graphiti search tool
│   │   ├── search.go              # Base search tool
│   │   ├── duckduckgo.go          # DuckDuckGo search
│   │   ├── google.go              # Google search
│   │   ├── searxng.go             # SearXNG search
│   │   ├── searxng_test.go
│   │   ├── tavily.go              # Tavily search
│   │   ├── perplexity.go          # Perplexity search
│   │   └── traversaal.go          # Traversaal search
│   │
│   └── version/                    # Version information
│       └── version.go
│
└── sqlc/                           # SQL type generation config
    ├── sqlc.yml                    # sqlc configuration
    └── models/                     # SQL model definitions
        ├── agentlogs.sql
        ├── assistantlogs.sql
        ├── assistants.sql
        ├── containers.sql
        ├── flows.sql
        ├── msgchains.sql
        ├── msglogs.sql
        ├── prompts.sql
        ├── providers.sql
        ├── roles.sql
        ├── screenshots.sql
        ├── searchlogs.sql
        ├── subtasks.sql
        ├── tasks.sql
        ├── termlogs.sql
        ├── toolcalls.sql
        ├── users.sql
        └── vecstorelogs.sql
```

---

## Directory Structure Description

### Core Directories

| Directory | Purpose | Key Components |
|-----------|---------|----------------|
| **cmd/** | Executable entry points | Main server, testers, installer |
| **pkg/** | Core application packages | Controllers, providers, tools, database |
| **migrations/** | Database schema evolution | SQL migration files (goose) |
| **sqlc/** | Type-safe SQL query definitions | SQL models and queries |
| **docs/** | Project documentation | Architecture docs, guides |
| **fern/** | API documentation generation | OpenAPI specs |
| **gqlgen/** | GraphQL code generation | Schema-to-code config |

### Key Package Descriptions

#### `pkg/config/`
- **Purpose:** Centralized configuration management
- **Key File:** `config.go` (90+ environment variables)
- **Features:** Environment parsing, validation, defaults

#### `pkg/controller/`
- **Purpose:** Business logic orchestration
- **Key Files:** `flow.go`, `task.go`, `subtask.go`, `assistant.go`
- **Responsibilities:**
  - Flow lifecycle (create, execute, stop, delete)
  - Task/subtask execution orchestration
  - Agent coordination
  - Event notifications via subscriptions

#### `pkg/database/`
- **Purpose:** Data access layer
- **Technology:** sqlc (type-safe SQL) + GORM
- **Key Files:**
  - `models.go` - Generated type definitions
  - `*.sql.go` - Generated query methods
- **Tables:** 20 tables (flows, tasks, subtasks, assistants, logs, etc.)

#### `pkg/providers/`
- **Purpose:** AI provider abstraction layer
- **Supported Providers:**
  - OpenAI (GPT-4, o1, o3)
  - Anthropic (Claude)
  - Google Gemini
  - AWS Bedrock
  - Ollama (local models)
  - Custom (OpenAI-compatible APIs)
- **Key Responsibilities:**
  - Unified interface for all providers
  - Streaming support
  - Tool calling management
  - Token usage tracking

#### `pkg/tools/`
- **Purpose:** AI agent tool implementations
- **Tool Categories:**
  - **Search:** Google, DuckDuckGo, Tavily, Perplexity, SearXNG, Traversaal
  - **Memory:** Vector store, Graphiti knowledge graph
  - **Code:** Code search and storage
  - **Terminal:** Command execution in Docker
  - **Browser:** Web scraping and screenshots
  - **File:** File operations

#### `pkg/graph/`
- **Purpose:** GraphQL API layer
- **Key Files:**
  - `schema.graphqls` - Schema definition (600+ lines)
  - `schema.resolvers.go` - Resolver implementations
  - `subscriptions/` - Real-time event streaming

#### `pkg/server/`
- **Purpose:** HTTP server and REST API
- **Framework:** Gin
- **Features:**
  - REST endpoints (complementary to GraphQL)
  - Authentication (local + OAuth2)
  - CORS configuration
  - Swagger documentation
  - Session management

#### `pkg/observability/`
- **Purpose:** Telemetry and monitoring
- **Technologies:**
  - OpenTelemetry (traces, metrics, logs)
  - Langfuse (LLM observability)
- **Capabilities:**
  - Distributed tracing
  - Performance metrics
  - LLM call tracking
  - Cost monitoring

---

## Entry Points

### 1. Main Application - `cmd/pentagi/main.go`

**Purpose:** Production server

**Key Functions:**
```go
func main() {
    // 1. Load configuration
    cfg, err := config.NewConfig()
    
    // 2. Initialize observability
    lfclient, _ := obs.NewLangfuseClient(ctx, cfg)
    otelclient, _ := obs.NewTelemetryClient(ctx, cfg)
    obs.InitObserver(ctx, lfclient, otelclient, logLevels)
    
    // 3. Connect to database
    db, _ := sql.Open("postgres", cfg.DatabaseURL)
    queries := database.New(db)
    orm, _ := database.NewGorm(cfg.DatabaseURL, "postgres")
    
    // 4. Run migrations
    goose.Up(db, "sql")
    
    // 5. Initialize Docker client
    client, _ := docker.NewDockerClient(ctx, queries, cfg)
    
    // 6. Initialize providers
    providers, _ := providers.NewProviderController(cfg, queries, client)
    
    // 7. Initialize controller
    subscriptions := subscriptions.NewSubscriptionsController()
    controller := controller.NewFlowController(queries, cfg, client, providers, subscriptions)
    
    // 8. Load existing flows
    controller.LoadFlows(ctx)
    
    // 9. Start HTTP server
    r := router.NewRouter(queries, orm, cfg, providers, controller, subscriptions)
    r.Run(listen)
}
```

**Startup Order:**
1. Configuration loading (.env + environment variables)
2. Observability setup (OpenTelemetry + Langfuse)
3. Database connection (PostgreSQL + pgvector)
4. Database migrations (goose)
5. Docker client initialization
6. AI providers initialization
7. Flow controller initialization
8. Load existing flows from database
9. HTTP/GraphQL server start
10. Signal handling for graceful shutdown

### 2. Interactive Installer - `cmd/installer/main.go`

**Purpose:** TUI-based initial setup wizard

**Technology:** Charm Bubbletea (Terminal UI framework)

**Responsibilities:**
- System requirements checking
- Docker configuration
- LLM provider setup
- API key validation
- `.env` file generation

### 3. Test Utilities

#### Controller Tester - `cmd/ctester/main.go`
- Tests controller logic
- Validates flow execution
- Reports generation

#### Embeddings Tester - `cmd/etester/main.go`
- Tests vector store operations
- Validates embeddings
- Search quality testing

#### Flow Tester - `cmd/ftester/main.go`
- End-to-end flow testing
- Mock providers
- Integration testing

---

## Bootstrap Sequence

### Phase 1: Configuration (0-500ms)

```
1. Load .env file → godotenv.Load()
2. Parse environment variables → env.ParseWithOptions()
3. Validate required configs
4. Generate/Load installation ID
5. Validate license key (if present)
```

**Critical Environment Variables:**
- `DATABASE_URL` - PostgreSQL connection
- `OPEN_AI_KEY` / `ANTHROPIC_API_KEY` / etc.
- `DOCKER_SOCKET` - Docker connection
- `DATA_DIR` - Persistent storage path

### Phase 2: Observability Setup (500ms-1s)

```
1. Initialize Langfuse client (if configured)
   ↓
2. Initialize OpenTelemetry client (if configured)
   ↓
3. Set up global observer
   ↓
4. Start metric collectors:
   - Process metrics (CPU, memory)
   - Go runtime metrics (goroutines, GC)
```

### Phase 3: Database Initialization (1s-2s)

```
1. Open PostgreSQL connection
   ↓
2. Configure connection pool:
   - MaxOpenConns: 20
   - MaxIdleConns: 5
   - ConnMaxLifetime: 1 hour
   ↓
3. Create sqlc queries instance
   ↓
4. Create GORM instance
   ↓
5. Run database migrations (goose)
   ↓
6. Verify pgvector extension
```

**Database Connection Pool:**
- Max Open Connections: 20
- Max Idle Connections: 5
- Connection Lifetime: 1 hour

### Phase 4: Docker Client Initialization (2s-3s)

```
1. Detect Docker socket
   - Unix: /var/run/docker.sock
   - Windows: npipe://./pipe/docker_engine
   ↓
2. Connect to Docker daemon
   ↓
3. Verify Docker API version
   ↓
4. Test container creation capability
```

### Phase 5: AI Providers Initialization (3s-4s)

```
1. Load provider configurations from database
   ↓
2. Initialize each provider:
   - OpenAI (if API key present)
   - Anthropic (if API key present)
   - Gemini (if API key present)
   - Bedrock (if credentials present)
   - Ollama (if server URL present)
   - Custom (if configured)
   ↓
3. Initialize embeddings provider
   ↓
4. Validate provider connectivity (async)
```

**Provider Initialization Logic:**
```go
// For each provider type
if cfg.ProviderAPIKey != "" {
    provider := NewProvider(cfg)
    if err := provider.Validate(); err == nil {
        providers.Register(provider)
    }
}
```

### Phase 6: Flow Controller Initialization (4s-5s)

```
1. Create subscriptions controller (WebSocket manager)
   ↓
2. Create flow controller
   ↓
3. Load existing flows from database:
   - Query all non-deleted flows
   - For each flow:
     * Restore Docker containers (if status=running)
     * Initialize tool executors
     * Setup provider connections
     * Resume pending tasks (if any)
   ↓
4. Start flow worker pool
```

**Flow Loading:**
- Queries: `SELECT * FROM flows WHERE deleted_at IS NULL`
- For each flow:
  - Check primary container status
  - Restore or recreate container if needed
  - Load task/subtask states
  - Resume execution if interrupted

### Phase 7: HTTP Server Start (5s-6s)

```
1. Initialize Gin router
   ↓
2. Setup middlewares:
   - CORS (configurable origins)
   - Logging (logrus integration)
   - Recovery (panic handler)
   - Sessions (cookie-based)
   - Authentication
   ↓
3. Register routes:
   - GraphQL endpoint: /api/v1/graphql
   - GraphQL Playground: /api/v1/graphql/playground
   - REST API: /api/v1/* (flows, tasks, etc.)
   - Swagger: /api/v1/swagger/*
   - OAuth callbacks: /api/v1/auth/*
   ↓
4. Start HTTP server (default: 0.0.0.0:8080)
   ↓
5. Setup graceful shutdown handler
```

**Server Configuration:**
- Default Port: 8080
- SSL Support: Optional (via `SERVER_USE_SSL`)
- CORS: Configurable origins
- Session Timeout: 4 hours

### Phase 8: Signal Handling

```
1. Listen for SIGINT/SIGTERM
   ↓
2. On signal:
   - Stop accepting new requests
   - Wait for active requests (30s timeout)
   - Close database connections
   - Stop Docker containers
   - Flush observability data
   - Exit gracefully
```

---

## Startup Timeline

| Time | Phase | Action |
|------|-------|--------|
| 0ms | Config | Load environment variables |
| 500ms | Observability | Initialize telemetry |
| 1s | Database | Connect to PostgreSQL |
| 1.5s | Database | Run migrations |
| 2s | Docker | Initialize Docker client |
| 3s | Providers | Load AI providers |
| 4s | Controller | Initialize flow controller |
| 4.5s | Controller | Load existing flows |
| 5s | Server | Start HTTP server |
| 5.5s | Ready | Application ready |

**Total Startup Time:** ~5-6 seconds (typical)

---

## Service Dependencies

```
PentAGI Server
    ├── PostgreSQL (required)
    │   └── pgvector extension
    ├── Docker (required)
    │   └── Docker containers (created on-demand)
    ├── AI Provider APIs (at least one required)
    │   ├── OpenAI
    │   ├── Anthropic
    │   ├── Google Gemini
    │   ├── AWS Bedrock
    │   ├── Ollama
    │   └── Custom OpenAI-compatible
    ├── Search Engine APIs (optional)
    │   ├── Google Search
    │   ├── DuckDuckGo
    │   ├── Tavily
    │   ├── Perplexity
    │   ├── SearXNG
    │   └── Traversaal
    ├── Browser Scraper (optional)
    │   └── Custom scraper service
    ├── Graphiti Server (optional)
    │   └── Knowledge graph service
    ├── OpenTelemetry Collector (optional)
    │   └── Observability backend
    └── Langfuse (optional)
        └── LLM observability platform
```

---

## Critical Files for Migration

### Must-Analyze First:
1. `cmd/pentagi/main.go` - Application bootstrap
2. `pkg/config/config.go` - Configuration structure
3. `pkg/database/models.go` - All data models
4. `pkg/graph/schema.graphqls` - API schema
5. `pkg/providers/provider.go` - Provider interface
6. `pkg/tools/tools.go` - Tools interface
7. `pkg/controller/flow.go` - Main orchestration logic

### Secondary Priority:
8. `pkg/server/router.go` - HTTP routing
9. `pkg/providers/handlers.go` - Agent handlers
10. `pkg/cast/chain_ast.go` - Chain parsing
11. `pkg/csum/chain_summary.go` - Summarization logic
12. `migrations/sql/*.sql` - Database schema

---

## Build and Test Commands

```bash
# Build main application
go build -o pentagi ./cmd/pentagi

# Run tests
go test ./...

# Generate code
go generate ./...

# Generate GraphQL
go run github.com/99designs/gqlgen generate

# Generate sqlc queries
sqlc generate

# Run migrations
goose -dir migrations/sql postgres "$DATABASE_URL" up

# Lint
golangci-lint run

# Format
gofmt -w .
```

---

## Migration Complexity Assessment

| Component | Complexity | Reason |
|-----------|-----------|---------|
| Data Models | ⭐ Easy | Straightforward structs → dataclasses/Pydantic |
| Database Layer | ⭐⭐ Medium | sqlc → SQLAlchemy + Alembic |
| HTTP Server | ⭐ Easy | Gin → FastAPI (very similar) |
| GraphQL | ⭐⭐ Medium | gqlgen → Strawberry/Ariadne |
| AI Providers | ⭐⭐⭐ Medium-Hard | Need to port streaming, tool calling |
| Docker Client | ⭐⭐ Medium | docker-py has similar API |
| Tools System | ⭐⭐⭐ Hard | Complex executor pattern |
| Observability | ⭐⭐ Medium | OpenTelemetry Python SDK available |
| TUI Installer | ⭐⭐⭐⭐ Hard | Bubbletea → Textual (different paradigm) |

---

**Next Document:** [02_DATA_MODELS.md](02_DATA_MODELS.md) - Detailed database schema analysis

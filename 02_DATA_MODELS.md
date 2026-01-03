# 02 - Data Models Analysis - PentAGI Backend

> **Project:** PentAGI Backend (Go)  
> **Purpose:** Database schema and models for Python migration  
> **Date:** 2026-01-03

## 📚 Table of Contents

1. [Overview](#overview)
2. [Database Technology](#database-technology)
3. [All Tables](#all-tables)
4. [Core Models](#core-models)
5. [Logging Models](#logging-models)
6. [Authentication Models](#authentication-models)
7. [Configuration Models](#configuration-models)
8. [Entity Relationship Diagram](#entity-relationship-diagram)
9. [Table Relationships](#table-relationships)
10. [Enums and Types](#enums-and-types)
11. [Indexes and Performance](#indexes-and-performance)
12. [Migration Strategy](#migration-strategy)

---

## Overview

The PentAGI backend uses **PostgreSQL** with **pgvector** extension for vector storage. The database schema consists of **20 tables** organized into logical groups:
- **Core Execution**: flows, tasks, subtasks, containers
- **AI Interaction**: msgchains, msglogs, assistants, assistantlogs
- **Logging**: agentlogs, searchlogs, termlogs, vecstorelogs, screenshots
- **Tool Tracking**: toolcalls
- **Configuration**: prompts, providers
- **Authentication**: users, roles, privileges

**Code Generation Tools:**
- **sqlc** - Type-safe SQL queries → Go structs
- **goose** - Database migrations

---

## Database Technology

### PostgreSQL Configuration
```go
// From main.go
db.SetMaxOpenConns(20)
db.SetMaxIdleConns(5)
db.SetConnMaxLifetime(time.Hour)
```

### Extensions Required
```sql
-- Vector similarity search
CREATE EXTENSION vector;

-- For pgvector similarity search
-- Embeddings stored as vector(1536) for OpenAI ada-002
-- Embeddings stored as vector(3072) for larger models
```

### Connection String Format
```
postgres://user:pass@host:port/dbname?sslmode=disable
```

---

## All Tables

| Table Name | Purpose | Row Count (Typical) | Relationships |
|-----------|---------|---------------------|---------------|
| **flows** | Execution flows | 10-100 | → users, containers, tasks |
| **tasks** | User tasks | 50-500 | → flows, subtasks, msgchains |
| **subtasks** | Task breakdown | 200-2000 | → tasks, msgchains |
| **containers** | Docker containers | 10-100 | → flows, termlogs |
| **msgchains** | AI conversation chains | 500-5000 | → flows, tasks, subtasks |
| **msglogs** | User-visible messages | 1000-10000 | → flows, tasks, subtasks |
| **assistants** | Assistant instances | 20-200 | → flows, assistantlogs |
| **assistantlogs** | Assistant messages | 500-5000 | → flows, assistants |
| **agentlogs** | Internal agent calls | 1000-10000 | → flows, tasks, subtasks |
| **searchlogs** | Search operations | 500-5000 | → flows, tasks, subtasks |
| **termlogs** | Terminal I/O | 5000-50000 | → containers |
| **vecstorelogs** | Vector store ops | 100-1000 | → flows, tasks, subtasks |
| **toolcalls** | Tool invocations | 2000-20000 | → flows, tasks, subtasks |
| **screenshots** | Browser screenshots | 100-1000 | → flows |
| **prompts** | Custom prompts | 10-100 | → users |
| **providers** | LLM providers | 5-20 | → users |
| **users** | System users | 1-100 | → flows, prompts, providers |
| **roles** | User roles | 2-10 | → users, privileges |
| **privileges** | Permissions | 50-200 | → roles |

---

## Core Models

### 1. Flow (Execution Flow)

**Purpose:** Represents a complete AI-powered security assessment flow

**Go Struct:**
```go
type Flow struct {
    ID                int64           `json:"id"`
    Status            FlowStatus      `json:"status"`           // created|running|waiting|finished|failed
    Title             string          `json:"title"`
    Model             string          `json:"model"`            // e.g., "gpt-4o"
    ModelProviderName string          `json:"model_provider_name"` // e.g., "OpenAI Main"
    ModelProviderType ProviderType    `json:"model_provider_type"` // openai|anthropic|gemini|bedrock|ollama|custom
    Language          string          `json:"language"`         // e.g., "en"
    Functions         json.RawMessage `json:"functions"`        // Tool configuration
    UserID            int64           `json:"user_id"`
    CreatedAt         sql.NullTime    `json:"created_at"`
    UpdatedAt         sql.NullTime    `json:"updated_at"`
    DeletedAt         sql.NullTime    `json:"deleted_at"`       // Soft delete
    TraceID           sql.NullString  `json:"trace_id"`         // Observability trace ID
}
```

**SQL Schema:**
```sql
CREATE TYPE FLOW_STATUS AS ENUM ('created','running','waiting','finished','failed');

CREATE TABLE flows (
  id                  BIGINT        PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  status              FLOW_STATUS   NOT NULL DEFAULT 'created',
  title               TEXT          NOT NULL DEFAULT 'untitled',
  model               TEXT          NOT NULL,
  model_provider_name TEXT          NOT NULL,
  model_provider_type PROVIDER_TYPE NOT NULL,
  language            TEXT          NOT NULL,
  functions           JSON          NOT NULL DEFAULT '{}',
  user_id             BIGINT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  trace_id            TEXT          NULL,
  created_at          TIMESTAMPTZ   DEFAULT CURRENT_TIMESTAMP,
  updated_at          TIMESTAMPTZ   DEFAULT CURRENT_TIMESTAMP,
  deleted_at          TIMESTAMPTZ   NULL
);
```

**Indexes:**
```sql
CREATE INDEX flows_status_idx ON flows(status);
CREATE INDEX flows_title_idx ON flows(title);
CREATE INDEX flows_language_idx ON flows(language);
CREATE INDEX flows_model_provider_name_idx ON flows(model_provider_name);
CREATE INDEX flows_user_id_idx ON flows(user_id);
```

**Relationships:**
- **Belongs to:** users (user_id)
- **Has many:** tasks, containers, msgchains, msglogs, screenshots

**Python Model (SQLAlchemy):**
```python
from sqlalchemy import Column, BigInteger, String, JSON, DateTime, Enum
from sqlalchemy.orm import relationship
from datetime import datetime
from enum import Enum as PyEnum

class FlowStatus(str, PyEnum):
    CREATED = "created"
    RUNNING = "running"
    WAITING = "waiting"
    FINISHED = "finished"
    FAILED = "failed"

class Flow(Base):
    __tablename__ = 'flows'
    
    id = Column(BigInteger, primary_key=True)
    status = Column(Enum(FlowStatus), nullable=False, default=FlowStatus.CREATED)
    title = Column(String, nullable=False, default='untitled')
    model = Column(String, nullable=False)
    model_provider_name = Column(String, nullable=False)
    model_provider_type = Column(Enum(ProviderType), nullable=False)
    language = Column(String, nullable=False)
    functions = Column(JSON, nullable=False, default={})
    user_id = Column(BigInteger, ForeignKey('users.id', ondelete='CASCADE'), nullable=False)
    trace_id = Column(String, nullable=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    deleted_at = Column(DateTime, nullable=True)
    
    # Relationships
    user = relationship("User", back_populates="flows")
    tasks = relationship("Task", back_populates="flow", cascade="all, delete-orphan")
    containers = relationship("Container", back_populates="flow", cascade="all, delete-orphan")
```

---

### 2. Task

**Purpose:** A user-defined task to be executed within a flow

**Go Struct:**
```go
type Task struct {
    ID        int64        `json:"id"`
    Status    TaskStatus   `json:"status"`      // created|running|waiting|finished|failed
    Title     string       `json:"title"`
    Input     string       `json:"input"`        // User's task description
    Result    string       `json:"result"`       // Final task result
    FlowID    int64        `json:"flow_id"`
    CreatedAt sql.NullTime `json:"created_at"`
    UpdatedAt sql.NullTime `json:"updated_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE TASK_STATUS AS ENUM ('created','running','waiting','finished','failed');

CREATE TABLE tasks (
  id           BIGINT         PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  status       TASK_STATUS    NOT NULL DEFAULT 'created',
  title        TEXT           NOT NULL DEFAULT 'untitled',
  input        TEXT           NOT NULL,
  result       TEXT           NOT NULL DEFAULT '',
  flow_id      BIGINT         NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  created_at   TIMESTAMPTZ    DEFAULT CURRENT_TIMESTAMP,
  updated_at   TIMESTAMPTZ    DEFAULT CURRENT_TIMESTAMP
);
```

**Indexes:**
```sql
CREATE INDEX tasks_status_idx ON tasks(status);
CREATE INDEX tasks_title_idx ON tasks(title);
CREATE INDEX tasks_flow_id_idx ON tasks(flow_id);
```

**Relationships:**
- **Belongs to:** flows (flow_id)
- **Has many:** subtasks, msgchains, msglogs, agentlogs

**State Machine:**
```
created → running → waiting (if user input needed)
                 → finished (success)
                 → failed (error)
```

---

### 3. Subtask

**Purpose:** Breakdown of a task into atomic execution units

**Go Struct:**
```go
type Subtask struct {
    ID          int64         `json:"id"`
    Status      SubtaskStatus `json:"status"`      // created|running|waiting|finished|failed
    Title       string        `json:"title"`
    Description string        `json:"description"`  // AI-generated description
    Result      string        `json:"result"`
    Context     string        `json:"context"`      // Execution context
    TaskID      int64         `json:"task_id"`
    CreatedAt   sql.NullTime  `json:"created_at"`
    UpdatedAt   sql.NullTime  `json:"updated_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE SUBTASK_STATUS AS ENUM ('created','running','waiting','finished','failed');

CREATE TABLE subtasks (
  id            BIGINT           PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  status        SUBTASK_STATUS   NOT NULL DEFAULT 'created',
  title         TEXT             NOT NULL,
  description   TEXT             NOT NULL,
  result        TEXT             NOT NULL DEFAULT '',
  context       TEXT             NOT NULL DEFAULT '',  -- Added in migration
  task_id       BIGINT           NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  created_at    TIMESTAMPTZ      DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMPTZ      DEFAULT CURRENT_TIMESTAMP
);
```

**Indexes:**
```sql
CREATE INDEX subtasks_status_idx ON subtasks(status);
CREATE INDEX subtasks_title_idx ON subtasks(title);
CREATE INDEX subtasks_task_id_idx ON subtasks(task_id);
```

**Relationships:**
- **Belongs to:** tasks (task_id)
- **Has many:** msgchains, msglogs, agentlogs

**Context Field:**
Contains rendered execution context including:
- Task description
- Completed subtasks
- Available tools
- Docker image info
- Current time

---

### 4. Container

**Purpose:** Docker container management for isolated execution

**Go Struct:**
```go
type Container struct {
    ID        int64           `json:"id"`
    Type      ContainerType   `json:"type"`        // primary|secondary
    Name      string          `json:"name"`
    Image     string          `json:"image"`       // e.g., "debian:latest"
    Status    ContainerStatus `json:"status"`      // starting|running|stopped|deleted|failed
    LocalID   sql.NullString  `json:"local_id"`    // Docker container ID
    LocalDir  sql.NullString  `json:"local_dir"`   // Mounted directory
    FlowID    int64           `json:"flow_id"`
    CreatedAt sql.NullTime    `json:"created_at"`
    UpdatedAt sql.NullTime    `json:"updated_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE CONTAINER_TYPE AS ENUM ('primary','secondary');
CREATE TYPE CONTAINER_STATUS AS ENUM ('starting','running','stopped','deleted','failed');

CREATE TABLE containers (
  id           BIGINT             PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  type         CONTAINER_TYPE     NOT NULL DEFAULT 'primary',
  name         TEXT               NOT NULL DEFAULT MD5(RANDOM()::text),
  image        TEXT               NOT NULL,
  status       CONTAINER_STATUS   NOT NULL DEFAULT 'starting',
  local_id     TEXT               UNIQUE,                    -- Docker ID
  local_dir    TEXT,                                         -- Host mount path
  flow_id      BIGINT             NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  created_at   TIMESTAMPTZ        DEFAULT CURRENT_TIMESTAMP,
  updated_at   TIMESTAMPTZ        DEFAULT CURRENT_TIMESTAMP
);
```

**Container Types:**
- **primary:** Main execution container (one per flow)
- **secondary:** Additional containers spawned during execution

**Lifecycle:**
```
starting → running → stopped → deleted
                  → failed
```

---

### 5. MsgChain (Message Chain)

**Purpose:** Stores AI conversation history for specific agent types

**Go Struct:**
```go
type Msgchain struct {
    ID            int64           `json:"id"`
    Type          MsgchainType    `json:"type"`           // Agent type
    Model         string          `json:"model"`
    ModelProvider string          `json:"model_provider"`
    UsageIn       int64           `json:"usage_in"`       // Input tokens
    UsageOut      int64           `json:"usage_out"`      // Output tokens
    Chain         json.RawMessage `json:"chain"`          // []llms.MessageContent
    FlowID        int64           `json:"flow_id"`
    TaskID        sql.NullInt64   `json:"task_id"`
    SubtaskID     sql.NullInt64   `json:"subtask_id"`
    CreatedAt     sql.NullTime    `json:"created_at"`
    UpdatedAt     sql.NullTime    `json:"updated_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE MSGCHAIN_TYPE AS ENUM (
  'primary_agent',    -- Main execution agent
  'reporter',         -- Task result reporting
  'generator',        -- Subtask generation
  'refiner',          -- Subtask refinement
  'reflector',        -- Reflection agent
  'enricher',         -- Context enrichment
  'adviser',          -- Advisory agent
  'coder',            -- Code writing agent
  'memorist',         -- Memory management
  'searcher',         -- Web search agent
  'installer',        -- Software installation
  'pentester',        -- Penetration testing
  'summarizer',       -- Content summarization
  'tool_call_fixer',  -- Fix malformed tool calls
  'assistant'         -- Assistant mode
);

CREATE TABLE msgchains (
  id               BIGINT          PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  type             MSGCHAIN_TYPE   NOT NULL DEFAULT 'primary_agent',
  model            TEXT            NOT NULL,
  model_provider   TEXT            NOT NULL,
  usage_in         BIGINT          NOT NULL DEFAULT 0,
  usage_out        BIGINT          NOT NULL DEFAULT 0,
  chain            JSON            NOT NULL,              -- LLM message history
  flow_id          BIGINT          NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  task_id          BIGINT          NULL REFERENCES tasks(id) ON DELETE CASCADE,
  subtask_id       BIGINT          NULL REFERENCES subtasks(id) ON DELETE CASCADE,
  created_at       TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP,
  updated_at       TIMESTAMPTZ     DEFAULT CURRENT_TIMESTAMP
);
```

**Chain Field Structure:**
```json
[
  {
    "role": "system",
    "parts": [
      {"text": "You are a penetration testing AI..."}
    ]
  },
  {
    "role": "user",
    "parts": [
      {"text": "Scan the target website..."}
    ]
  },
  {
    "role": "assistant",
    "parts": [
      {"text": "I'll start by..."},
      {
        "toolCall": {
          "id": "call_abc123",
          "name": "terminal",
          "args": "{\"command\": \"nmap -sV target.com\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "parts": [
      {
        "toolResponse": {
          "toolCallID": "call_abc123",
          "content": "PORT    STATE SERVICE..."
        }
      }
    ]
  }
]
```

**Relationships:**
- **Belongs to:** flows, tasks (optional), subtasks (optional)
- **Tracks:** Token usage per agent type

---

### 6. MsgLog (Message Log)

**Purpose:** User-facing messages and notifications

**Go Struct:**
```go
type Msglog struct {
    ID           int64              `json:"id"`
    Type         MsglogType         `json:"type"`
    Message      string             `json:"message"`        // User-visible prompt
    Thinking     sql.NullString     `json:"thinking"`       // AI reasoning (optional)
    Result       string             `json:"result"`         // Result content
    ResultFormat MsglogResultFormat `json:"result_format"`  // plain|markdown|terminal
    FlowID       int64              `json:"flow_id"`
    TaskID       sql.NullInt64      `json:"task_id"`
    SubtaskID    sql.NullInt64      `json:"subtask_id"`
    CreatedAt    sql.NullTime       `json:"created_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE MSGLOG_TYPE AS ENUM (
  'answer',    -- General answer/response
  'report',    -- Task/subtask report
  'thoughts',  -- AI internal thoughts
  'browser',   -- Browser action log
  'terminal',  -- Terminal action log
  'file',      -- File operation log
  'search',    -- Search action log
  'advice',    -- Advice from adviser agent
  'ask',       -- Question to user
  'input',     -- User input received
  'done'       -- Completion notification
);

CREATE TYPE MSGLOG_RESULT_FORMAT AS ENUM ('plain', 'markdown', 'terminal');

CREATE TABLE msglogs (
  id            BIGINT              PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  type          MSGLOG_TYPE         NOT NULL,
  message       TEXT                NOT NULL,
  thinking      TEXT                NULL,                  -- AI reasoning
  result        TEXT                NOT NULL DEFAULT '',
  result_format MSGLOG_RESULT_FORMAT NOT NULL DEFAULT 'markdown',
  flow_id       BIGINT              NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
  task_id       BIGINT              NULL REFERENCES tasks(id) ON DELETE CASCADE,
  subtask_id    BIGINT              NULL REFERENCES subtasks(id) ON DELETE CASCADE,
  created_at    TIMESTAMPTZ         DEFAULT CURRENT_TIMESTAMP
);
```

**UI Display Logic:**
```
type=thoughts → Display as collapsible "AI Reasoning" section
type=terminal → Display with terminal styling (monospace, dark bg)
type=browser  → Display with link to screenshot
type=ask      → Display as interactive input prompt
type=report   → Display as formatted result card
```

---

## Logging Models

### 7. AgentLog

**Purpose:** Internal agent-to-agent communication log

```go
type Agentlog struct {
    ID        int64         `json:"id"`
    Initiator MsgchainType  `json:"initiator"`  // Which agent started
    Executor  MsgchainType  `json:"executor"`   // Which agent executed
    Task      string        `json:"task"`       // What was requested
    Result    string        `json:"result"`     // What was returned
    FlowID    int64         `json:"flow_id"`
    TaskID    sql.NullInt64 `json:"task_id"`
    SubtaskID sql.NullInt64 `json:"subtask_id"`
    CreatedAt sql.NullTime  `json:"created_at"`
}
```

**Example:**
```json
{
  "initiator": "primary_agent",
  "executor": "searcher",
  "task": "Search for latest CVEs for Apache Tomcat",
  "result": "Found 5 recent CVEs: CVE-2024-...",
  "flow_id": 42,
  "task_id": 7,
  "subtask_id": 15
}
```

---

### 8. SearchLog

**Purpose:** Web search operations tracking

```go
type Searchlog struct {
    ID        int64            `json:"id"`
    Initiator MsgchainType     `json:"initiator"`
    Executor  MsgchainType     `json:"executor"`
    Engine    SearchengineType `json:"engine"`    // google|tavily|duckduckgo|etc.
    Query     string           `json:"query"`
    Result    string           `json:"result"`    // Search results
    FlowID    int64            `json:"flow_id"`
    TaskID    sql.NullInt64    `json:"task_id"`
    SubtaskID sql.NullInt64    `json:"subtask_id"`
    CreatedAt sql.NullTime     `json:"created_at"`
}
```

**Search Engine Types:**
```sql
CREATE TYPE SEARCHENGINE_TYPE AS ENUM (
  'google',
  'tavily',
  'traversaal',
  'browser',
  'duckduckgo',
  'perplexity',
  'searxng'
);
```

---

### 9. TermLog (Terminal Log)

**Purpose:** Docker container terminal I/O

```go
type Termlog struct {
    ID          int64        `json:"id"`
    Type        TermlogType  `json:"type"`       // stdin|stdout|stderr
    Text        string       `json:"text"`
    ContainerID int64        `json:"container_id"`
    CreatedAt   sql.NullTime `json:"created_at"`
}
```

**SQL Schema:**
```sql
CREATE TYPE TERMLOG_TYPE AS ENUM ('stdin', 'stdout', 'stderr');

CREATE TABLE termlogs (
  id             BIGINT         PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  type           TERMLOG_TYPE   NOT NULL,
  text           TEXT           NOT NULL,
  container_id   BIGINT         NOT NULL REFERENCES containers(id) ON DELETE CASCADE,
  created_at     TIMESTAMPTZ    DEFAULT CURRENT_TIMESTAMP
);
```

**High Volume:** This table can grow very large (50k+ rows per flow)

---

### 10. VectorStoreLog

**Purpose:** Vector database operations tracking

```go
type Vecstorelog struct {
    ID        int64              `json:"id"`
    Initiator MsgchainType       `json:"initiator"`
    Executor  MsgchainType       `json:"executor"`
    Filter    json.RawMessage    `json:"filter"`     // Search filter
    Query     string             `json:"query"`
    Action    VecstoreActionType `json:"action"`     // retrieve|store
    Result    string             `json:"result"`
    FlowID    int64              `json:"flow_id"`
    TaskID    sql.NullInt64      `json:"task_id"`
    SubtaskID sql.NullInt64      `json:"subtask_id"`
    CreatedAt sql.NullTime       `json:"created_at"`
}
```

**Actions:**
- **store:** Add embeddings to vector store
- **retrieve:** Similarity search

---

### 11. ToolCall

**Purpose:** Individual tool invocation tracking

```go
type Toolcall struct {
    ID        int64           `json:"id"`
    CallID    string          `json:"call_id"`      // LLM-provided ID
    Status    ToolcallStatus  `json:"status"`       // received|running|finished|failed
    Name      string          `json:"name"`         // Tool name
    Args      json.RawMessage `json:"args"`         // Tool arguments
    Result    string          `json:"result"`       // Tool result
    FlowID    int64           `json:"flow_id"`
    TaskID    sql.NullInt64   `json:"task_id"`
    SubtaskID sql.NullInt64   `json:"subtask_id"`
    CreatedAt sql.NullTime    `json:"created_at"`
    UpdatedAt sql.NullTime    `json:"updated_at"`
}
```

**Example Tools:**
- `terminal` - Execute shell commands
- `browser` - Web scraping
- `google_search` - Google search
- `memorist` - Query vector store
- `done` - Mark subtask complete

---

### 12. Screenshot

**Purpose:** Browser screenshot storage references

```go
type Screenshot struct {
    ID        int64        `json:"id"`
    Name      string       `json:"name"`      // Filename
    Url       string       `json:"url"`       // Access URL
    FlowID    int64        `json:"flow_id"`
    CreatedAt sql.NullTime `json:"created_at"`
}
```

**Storage:** Files stored in `${DATA_DIR}/screenshots/`

---

## Authentication Models

### 13. User

```go
type User struct {
    ID                     int64          `json:"id"`
    Hash                   string         `json:"hash"`           // Public identifier
    Type                   UserType       `json:"type"`           // local|oauth
    Mail                   string         `json:"mail"`
    Name                   string         `json:"name"`
    Password               sql.NullString `json:"password"`       // bcrypt hash
    Status                 UserStatus     `json:"status"`         // created|active|blocked
    RoleID                 int64          `json:"role_id"`
    PasswordChangeRequired bool           `json:"password_change_required"`
    Provider               sql.NullString `json:"provider"`       // google|github
    CreatedAt              sql.NullTime   `json:"created_at"`
}
```

**Default User:**
```
email: admin@pentagi.com
password: admin
(Must change on first login)
```

---

### 14. Role

```go
type Role struct {
    ID   int64  `json:"id"`
    Name string `json:"name"`
}
```

**Default Roles:**
- **Admin:** Full system access
- **User:** Standard user access

---

### 15. Privilege

```go
type Privilege struct {
    ID     int64  `json:"id"`
    RoleID int64  `json:"role_id"`
    Name   string `json:"name"`
}
```

**Privilege Format:** `resource.action` (e.g., `flows.create`, `users.delete`)

---

## Configuration Models

### 16. Prompt

**Purpose:** User-customized AI prompts

```go
type Prompt struct {
    ID        int64        `json:"id"`
    Type      PromptType   `json:"type"`      // primary_agent, coder, etc.
    UserID    int64        `json:"user_id"`
    Prompt    string       `json:"prompt"`    // Jinja2 template
    CreatedAt sql.NullTime `json:"created_at"`
    UpdatedAt sql.NullTime `json:"updated_at"`
}
```

**Prompt Types:** See MSGCHAIN_TYPE enum

---

### 17. Provider

**Purpose:** User-defined LLM provider configurations

```go
type Provider struct {
    ID        int64           `json:"id"`
    UserID    int64           `json:"user_id"`
    Type      ProviderType    `json:"type"`       // openai|anthropic|etc.
    Name      string          `json:"name"`       // User-friendly name
    Config    json.RawMessage `json:"config"`     // Provider-specific config
    CreatedAt sql.NullTime    `json:"created_at"`
    UpdatedAt sql.NullTime    `json:"updated_at"`
    DeletedAt sql.NullTime    `json:"deleted_at"`
}
```

**Config Structure (OpenAI example):**
```json
{
  "api_key": "sk-...",
  "base_url": "https://api.openai.com/v1",
  "agents": {
    "primary_agent": {
      "model": "gpt-4o",
      "temperature": 0.7,
      "max_tokens": 4096
    },
    "coder": {
      "model": "gpt-4o",
      "temperature": 0.3,
      "max_tokens": 8192
    }
  }
}
```

---

### 18. Assistant

**Purpose:** Standalone chat assistant instances

```go
type Assistant struct {
    ID                int64           `json:"id"`
    Status            AssistantStatus `json:"status"`
    Title             string          `json:"title"`
    Model             string          `json:"model"`
    ModelProviderName string          `json:"model_provider_name"`
    ModelProviderType ProviderType    `json:"model_provider_type"`
    Language          string          `json:"language"`
    Functions         json.RawMessage `json:"functions"`
    TraceID           sql.NullString  `json:"trace_id"`
    FlowID            int64           `json:"flow_id"`
    UseAgents         bool            `json:"use_agents"`      // Enable sub-agents
    MsgchainID        sql.NullInt64   `json:"msgchain_id"`     // Current chain
    CreatedAt         sql.NullTime    `json:"created_at"`
    UpdatedAt         sql.NullTime    `json:"updated_at"`
    DeletedAt         sql.NullTime    `json:"deleted_at"`
}
```

**vs Flow:** Assistants are lightweight, conversational interfaces without complex task breakdown

---

### 19. AssistantLog

```go
type Assistantlog struct {
    ID           int64              `json:"id"`
    Type         MsglogType         `json:"type"`
    Message      string             `json:"message"`
    Thinking     sql.NullString     `json:"thinking"`
    Result       string             `json:"result"`
    ResultFormat MsglogResultFormat `json:"result_format"`
    FlowID       int64              `json:"flow_id"`
    AssistantID  int64              `json:"assistant_id"`
    CreatedAt    sql.NullTime       `json:"created_at"`
}
```

---

## Entity Relationship Diagram

```
┌──────────┐         ┌──────────┐
│  users   │1───────*│  flows   │
└──────────┘         └──────────┘
     │1                   │1
     │                    ├───────*────────┐
     │                    │                │
     │                   1│               1│
     │              ┌──────────┐    ┌────────────┐
     │              │  tasks   │    │ containers │
     │              └──────────┘    └────────────┘
     │                   │1               │1
     │                   │                │
     │                   │*               │*
     │              ┌──────────┐    ┌──────────┐
     │              │ subtasks │    │ termlogs │
     │              └──────────┘    └──────────┘
     │                   │
     │                   │
     │                   │* (task_id, subtask_id)
     │              ┌────────────┐
     │              │ msgchains  │
     │              └────────────┘
     │                   │1
     │                   │
     │                   │* (FK to flows, tasks, subtasks)
     │              ┌──────────┐
     │              │ msglogs  │
     │              └──────────┘
     │
     │*
┌──────────┐
│ prompts  │
└──────────┘

┌──────────┐
│providers │
└──────────┘
     │*
     │
     │1
┌──────────┐
│  users   │
└──────────┘
     │1
     │
     │*
┌──────────┐
│  roles   │
└──────────┘
     │1
     │
     │*
┌────────────┐
│ privileges │
└────────────┘
```

---

## Table Relationships

### Cascade Delete Hierarchy

```
users (ON DELETE CASCADE)
  ├─> flows
  │    ├─> tasks
  │    │    ├─> subtasks
  │    │    │    ├─> msgchains
  │    │    │    ├─> msglogs
  │    │    │    ├─> agentlogs
  │    │    │    ├─> searchlogs
  │    │    │    ├─> vecstorelogs
  │    │    │    └─> toolcalls
  │    │    ├─> msgchains
  │    │    ├─> msglogs
  │    │    ├─> agentlogs
  │    │    ├─> searchlogs
  │    │    ├─> vecstorelogs
  │    │    └─> toolcalls
  │    ├─> containers
  │    │    └─> termlogs
  │    ├─> screenshots
  │    ├─> msgchains
  │    ├─> msglogs
  │    ├─> agentlogs
  │    ├─> searchlogs
  │    ├─> vecstorelogs
  │    ├─> toolcalls
  │    └─> assistants
  │         └─> assistantlogs
  ├─> prompts
  └─> providers
```

**Soft Delete:** Only `flows`, `assistants`, `providers` use soft delete (deleted_at)

---

## Enums and Types

### Status Enums

```go
// Flow/Task/Subtask/Assistant states
type Status string
const (
    StatusCreated  Status = "created"   // Initial state
    StatusRunning  Status = "running"   // Currently executing
    StatusWaiting  Status = "waiting"   // Waiting for user input
    StatusFinished Status = "finished"  // Completed successfully
    StatusFailed   Status = "failed"    // Terminated with error
)
```

### Provider Type

```go
type ProviderType string
const (
    ProviderTypeOpenai    ProviderType = "openai"
    ProviderTypeAnthropic ProviderType = "anthropic"
    ProviderTypeGemini    ProviderType = "gemini"
    ProviderTypeBedrock   ProviderType = "bedrock"
    ProviderTypeOllama    ProviderType = "ollama"
    ProviderTypeCustom    ProviderType = "custom"
)
```

### Prompt Type (30+ types)

```go
type PromptType string
const (
    // Agents
    PromptTypePrimaryAgent      PromptType = "primary_agent"
    PromptTypeAssistant         PromptType = "assistant"
    PromptTypePentester         PromptType = "pentester"
    PromptTypeCoder             PromptType = "coder"
    PromptTypeInstaller         PromptType = "installer"
    // ... (see models.go for complete list)
)
```

---

## Indexes and Performance

### Critical Indexes

**High-traffic queries:**
```sql
-- Flow lookups
CREATE INDEX flows_user_id_idx ON flows(user_id);
CREATE INDEX flows_status_idx ON flows(status);

-- Task/subtask navigation
CREATE INDEX tasks_flow_id_idx ON tasks(flow_id);
CREATE INDEX subtasks_task_id_idx ON subtasks(task_id);

-- Log filtering
CREATE INDEX msglogs_flow_id_idx ON msglogs(flow_id);
CREATE INDEX msglogs_type_idx ON msglogs(type);

-- Container operations
CREATE INDEX termlogs_container_id_idx ON termlogs(container_id);
```

### Auto-updated Timestamps

```sql
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_flows_modified
  BEFORE UPDATE ON flows
  FOR EACH ROW EXECUTE PROCEDURE update_modified_column();
```

**Applied to:** flows, tasks, subtasks, containers, toolcalls, msgchains

---

## Migration Strategy

### Phase 1: Core Tables (Week 1)
```
1. users, roles, privileges
2. providers, prompts
3. flows
4. tasks
5. subtasks
```

### Phase 2: Execution Tables (Week 2)
```
6. containers
7. msgchains
8. toolcalls
9. msglogs
10. assistants, assistantlogs
```

### Phase 3: Logging Tables (Week 3)
```
11. agentlogs
12. searchlogs
13. termlogs
14. vecstorelogs
15. screenshots
```

### Python ORM Recommendation

**SQLAlchemy 2.0** with:
- **Alembic** for migrations
- **Pydantic** for validation
- **asyncpg** for async PostgreSQL

**Example Migration (Alembic):**
```python
"""create flows table

Revision ID: 001
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import ENUM

def upgrade():
    flow_status = ENUM('created', 'running', 'waiting', 'finished', 'failed', 
                       name='flow_status', create_type=True)
    flow_status.create(op.get_bind(), checkfirst=True)
    
    op.create_table('flows',
        sa.Column('id', sa.BigInteger(), autoincrement=True, nullable=False),
        sa.Column('status', flow_status, nullable=False, server_default='created'),
        sa.Column('title', sa.String(), nullable=False, server_default='untitled'),
        sa.Column('model', sa.String(), nullable=False),
        # ... rest of columns
        sa.PrimaryKeyConstraint('id')
    )
    
def downgrade():
    op.drop_table('flows')
    ENUM(name='flow_status').drop(op.get_bind(), checkfirst=True)
```

---

## Data Volume Estimates

**Small Installation (1-10 users):**
- flows: 100 rows (~100 KB)
- tasks: 500 rows (~500 KB)
- subtasks: 2,000 rows (~2 MB)
- msgchains: 5,000 rows (~50 MB with chain JSON)
- msglogs: 10,000 rows (~10 MB)
- termlogs: 50,000 rows (~50 MB)
- **Total:** ~120 MB

**Medium Installation (10-50 users):**
- **Total:** ~1-2 GB

**Large Installation (50+ users):**
- **Total:** ~10-20 GB

**Growth Rate:** ~1-2 GB per 1000 flows

---

## Migration Complexity

| Table | Complexity | Notes |
|-------|-----------|-------|
| users, roles | ⭐ Easy | Standard auth tables |
| flows, tasks | ⭐⭐ Medium | Enums, soft delete |
| subtasks | ⭐⭐ Medium | Context field large |
| msgchains | ⭐⭐⭐⭐ Hard | Large JSON, complex structure |
| msglogs | ⭐⭐ Medium | Various types |
| containers | ⭐⭐ Medium | Docker integration |
| termlogs | ⭐ Easy | High volume |
| toolcalls | ⭐⭐ Medium | JSON args |
| providers | ⭐⭐⭐ Medium-Hard | Complex config JSON |
| prompts | ⭐⭐ Medium | Template storage |

---

**Next Document:** [03_API_REFERENCE.md](03_API_REFERENCE.md) - GraphQL and REST API analysis

# 03 - API & GraphQL Reference - PentAGI Backend

> **Purpose:** Complete API documentation for Python migration  
> **Date:** 2026-01-03

## Overview

PentAGI exposes both **GraphQL** (primary) and **REST** (supplementary) APIs:
- **GraphQL:** Main API at `/api/v1/graphql` (queries, mutations, subscriptions)
- **REST:** Administrative endpoints at `/api/v1/*`
- **Authentication:** Session-based + OAuth2 (Google, GitHub)

---

## GraphQL API

### Schema Location
- **File:** `backend/pkg/graph/schema.graphqls` (662 lines)
- **Generator:** gqlgen (99designs/gqlgen)
- **Playground:** Available at `/api/v1/graphql/playground`

### Core Types

#### Execution Types
```graphql
type Flow {
  id: ID!
  title: String!
  status: StatusType!  # created|running|waiting|finished|failed
  terminals: [Terminal!]
  provider: Provider!
  createdAt: Time!
  updatedAt: Time!
}

type Task {
  id: ID!
  title: String!
  status: StatusType!
  input: String!
  result: String!
  flowId: ID!
  subtasks: [Subtask!]
  createdAt: Time!
  updatedAt: Time!
}

type Subtask {
  id: ID!
  status: StatusType!
  title: String!
  description: String!
  result: String!
  taskId: ID!
  createdAt: Time!
  updatedAt: Time!
}
```

#### Assistant Types
```graphql
type Assistant {
  id: ID!
  title: String!
  status: StatusType!
  provider: Provider!
  flowId: ID!
  useAgents: Boolean!  # Enable sub-agents
  createdAt: Time!
  updatedAt: Time!
}

type FlowAssistant {
  flow: Flow!
  assistant: Assistant!
}
```

#### Logging Types
```graphql
type MessageLog {
  id: ID!
  type: MessageLogType!  # answer|report|thoughts|browser|terminal|...
  message: String!
  thinking: String  # AI reasoning (optional)
  result: String!
  resultFormat: ResultFormat!  # plain|markdown|terminal
  flowId: ID!
  taskId: ID
  subtaskId: ID
  createdAt: Time!
}

type AgentLog {
  id: ID!
  initiator: AgentType!  # Which agent called
  executor: AgentType!   # Which agent executed
  task: String!
  result: String!
  flowId: ID!
  taskId: ID
  subtaskId: ID
  createdAt: Time!
}

type SearchLog {
  id: ID!
  initiator: AgentType!
  executor: AgentType!
  engine: String!  # google|tavily|duckduckgo|...
  query: String!
  result: String!
  flowId: ID!
  taskId: ID
  subtaskId: ID
  createdAt: Time!
}
```

### Queries

```graphql
type Query {
  # Provider management
  providers: [Provider!]!
  
  # Flow and assistant management
  assistants(flowId: ID!): [Assistant!]
  flows: [Flow!]
  flow(flowId: ID!): Flow!
  
  # Task and execution logs
  tasks(flowId: ID!): [Task!]
  screenshots(flowId: ID!): [Screenshot!]
  terminalLogs(flowId: ID!): [TerminalLog!]
  messageLogs(flowId: ID!): [MessageLog!]
  agentLogs(flowId: ID!): [AgentLog!]
  searchLogs(flowId: ID!): [SearchLog!]
  vectorStoreLogs(flowId: ID!): [VectorStoreLog!]
  assistantLogs(flowId: ID!, assistantId: ID!): [AssistantLog!]
  
  # System settings
  settings: Settings!
  settingsProviders: ProvidersConfig!
  settingsPrompts: PromptsConfig!
}
```

### Mutations

```graphql
type Mutation {
  # Flow management
  createFlow(modelProvider: String!, input: String!): Flow!
  putUserInput(flowId: ID!, input: String!): ResultType!
  stopFlow(flowId: ID!): ResultType!
  finishFlow(flowId: ID!): ResultType!
  deleteFlow(flowId: ID!): ResultType!
  
  # Assistant management
  createAssistant(
    flowId: ID!, 
    modelProvider: String!, 
    input: String!, 
    useAgents: Boolean!
  ): FlowAssistant!
  callAssistant(
    flowId: ID!, 
    assistantId: ID!, 
    input: String!, 
    useAgents: Boolean!
  ): ResultType!
  stopAssistant(flowId: ID!, assistantId: ID!): Assistant!
  deleteAssistant(flowId: ID!, assistantId: ID!): ResultType!
  
  # Provider testing and management
  testAgent(
    type: ProviderType!, 
    agentType: AgentConfigType!, 
    agent: AgentConfigInput!
  ): AgentTestResult!
  testProvider(
    type: ProviderType!, 
    agents: AgentsConfigInput!
  ): ProviderTestResult!
  createProvider(
    name: String!, 
    type: ProviderType!, 
    agents: AgentsConfigInput!
  ): ProviderConfig!
  updateProvider(
    providerId: ID!, 
    name: String!, 
    agents: AgentsConfigInput!
  ): ProviderConfig!
  deleteProvider(providerId: ID!): ResultType!
  
  # Prompt management
  validatePrompt(type: PromptType!, template: String!): PromptValidationResult!
  createPrompt(type: PromptType!, template: String!): UserPrompt!
  updatePrompt(promptId: ID!, template: String!): UserPrompt!
  deletePrompt(promptId: ID!): ResultType!
}
```

### Subscriptions (WebSocket)

```graphql
type Subscription {
  # Flow events
  flowCreated: Flow!
  flowDeleted: Flow!
  flowUpdated: Flow!
  taskCreated(flowId: ID!): Task!
  taskUpdated(flowId: ID!): Task!
  
  # Assistant events
  assistantCreated(flowId: ID!): Assistant!
  assistantUpdated(flowId: ID!): Assistant!
  assistantDeleted(flowId: ID!): Assistant!
  
  # Log events (real-time streaming)
  screenshotAdded(flowId: ID!): Screenshot!
  terminalLogAdded(flowId: ID!): TerminalLog!
  messageLogAdded(flowId: ID!): MessageLog!
  messageLogUpdated(flowId: ID!): MessageLog!
  agentLogAdded(flowId: ID!): AgentLog!
  searchLogAdded(flowId: ID!): SearchLog!
  vectorStoreLogAdded(flowId: ID!): VectorStoreLog!
  assistantLogAdded(flowId: ID!): AssistantLog!
  assistantLogUpdated(flowId: ID!): AssistantLog!
  
  # Provider events
  providerCreated: ProviderConfig!
  providerUpdated: ProviderConfig!
  providerDeleted: ProviderConfig!
}
```

### Provider Configuration Types

```graphql
type ProviderConfig {
  id: ID!
  name: String!  # User-friendly name
  type: ProviderType!  # openai|anthropic|gemini|bedrock|ollama|custom
  agents: AgentsConfig!  # Per-agent model configs
  createdAt: Time!
  updatedAt: Time!
}

type AgentsConfig {
  simple: AgentConfig!          # Basic completion
  simpleJson: AgentConfig!      # JSON mode completion
  primaryAgent: AgentConfig!    # Main execution agent
  assistant: AgentConfig!       # Chat assistant
  generator: AgentConfig!       # Subtask generation
  refiner: AgentConfig!         # Subtask refinement
  adviser: AgentConfig!         # Advisory agent
  reflector: AgentConfig!       # Reflection agent
  searcher: AgentConfig!        # Web search agent
  enricher: AgentConfig!        # Context enrichment
  coder: AgentConfig!           # Code writing
  installer: AgentConfig!       # Software installation
  pentester: AgentConfig!       # Pentesting
}

type AgentConfig {
  model: String!  # e.g., "gpt-4o", "claude-3-5-sonnet"
  maxTokens: Int
  temperature: Float
  topK: Int
  topP: Float
  minLength: Int
  maxLength: Int
  repetitionPenalty: Float
  frequencyPenalty: Float
  presencePenalty: Float
  reasoning: ReasoningConfig  # For o1/o3 models
  price: ModelPrice
}
```

---

## REST API

### Base Path
`/api/v1/`

### Authentication Endpoints

```http
POST   /api/v1/auth/login              # Local login
GET    /api/v1/auth/logout             # Logout
GET    /api/v1/auth/authorize          # OAuth start
GET    /api/v1/auth/login-callback     # OAuth callback (GET)
POST   /api/v1/auth/login-callback     # OAuth callback (POST)
POST   /api/v1/auth/logout-callback    # OAuth logout
```

### Flow Endpoints

```http
GET    /api/v1/flows                   # List all flows
POST   /api/v1/flows                   # Create flow
GET    /api/v1/flows/:flowID           # Get flow details
PUT    /api/v1/flows/:flowID           # Update flow
DELETE /api/v1/flows/:flowID           # Delete flow
GET    /api/v1/flows/:flowID/graph     # Get flow graph (Mermaid)
```

### Task Endpoints

```http
GET    /api/v1/flows/:flowID/tasks               # List tasks
GET    /api/v1/flows/:flowID/tasks/:taskID       # Get task
GET    /api/v1/flows/:flowID/tasks/:taskID/graph # Task graph
```

### Subtask Endpoints

```http
GET    /api/v1/flows/:flowID/subtasks                        # List all subtasks
GET    /api/v1/flows/:flowID/tasks/:taskID/subtasks          # List task subtasks
GET    /api/v1/flows/:flowID/tasks/:taskID/subtasks/:subtaskID  # Get subtask
```

### Container Endpoints

```http
GET    /api/v1/containers                         # List all containers
GET    /api/v1/flows/:flowID/containers           # Flow containers
GET    /api/v1/flows/:flowID/containers/:containerID  # Get container
```

### Assistant Endpoints

```http
GET    /api/v1/flows/:flowID/assistants              # List assistants
POST   /api/v1/flows/:flowID/assistants              # Create assistant
GET    /api/v1/flows/:flowID/assistants/:assistantID # Get assistant
PUT    /api/v1/flows/:flowID/assistants/:assistantID # Update assistant
DELETE /api/v1/flows/:flowID/assistants/:assistantID # Delete assistant
```

### Log Endpoints

```http
GET    /api/v1/flows/:flowID/msglogs       # Message logs
GET    /api/v1/flows/:flowID/agentlogs     # Agent logs
GET    /api/v1/flows/:flowID/searchlogs    # Search logs
GET    /api/v1/flows/:flowID/termlogs      # Terminal logs
GET    /api/v1/flows/:flowID/vecstorelogs  # Vector store logs
GET    /api/v1/flows/:flowID/screenshots   # Screenshots
GET    /api/v1/flows/:flowID/screenshots/:screenshotID/file  # Download screenshot
```

### Provider Endpoints

```http
GET    /api/v1/providers                   # List providers
```

### Prompt Endpoints

```http
GET    /api/v1/prompts                     # List prompts
GET    /api/v1/prompts/:promptType         # Get prompt
PUT    /api/v1/prompts/:promptType         # Update prompt
POST   /api/v1/prompts/:promptType/default # Reset to default
```

### User Management Endpoints

```http
GET    /api/v1/users                       # List users (Admin)
POST   /api/v1/users                       # Create user (Admin)
GET    /api/v1/users/:hash                 # Get user (Admin)
PUT    /api/v1/users/:hash                 # Update user (Admin)
DELETE /api/v1/users/:hash                 # Delete user (Admin)
GET    /api/v1/user                        # Get current user
PUT    /api/v1/user/password               # Change own password
```

### Role Endpoints

```http
GET    /api/v1/roles                       # List roles
GET    /api/v1/roles/:roleID               # Get role
```

### Developer Endpoints

```http
GET    /api/v1/graphql/playground          # GraphQL playground UI
GET    /api/v1/swagger/*                   # Swagger documentation
GET    /api/v1/info                        # Server info
```

---

## Authentication System

### Session Management

**Technology:** Gin sessions with cookie store

**Configuration:**
```go
// From router.go
cookieStore := cookie.NewStore(
    auth.MakeCookieStoreKey(cfg.CookieSigningSalt)...
)
router.Use(sessions.Sessions("auth", cookieStore))
```

**Session Timeout:** 4 hours

**Cookie Name:** `auth`

### Local Authentication

**Password Hashing:** bcrypt (cost 10)

**Login Flow:**
1. POST `/api/v1/auth/login` with `{email, password}`
2. Server validates credentials
3. Server creates session
4. Returns user info + session cookie

**Default Admin:**
```
Email: admin@pentagi.com
Password: admin  # Must change on first login
```

### OAuth2 Authentication

**Supported Providers:**
- Google OAuth
- GitHub OAuth

**Flow:**
1. Frontend redirects to `/api/v1/auth/authorize?provider=google`
2. Server redirects to provider's OAuth page
3. User authorizes
4. Provider redirects to `/api/v1/auth/login-callback`
5. Server exchanges code for token
6. Server creates/updates user
7. Server creates session
8. Frontend redirected to application

**Environment Variables:**
```bash
# Google
OAUTH_GOOGLE_CLIENT_ID=...
OAUTH_GOOGLE_CLIENT_SECRET=...

# GitHub
OAUTH_GITHUB_CLIENT_ID=...
OAUTH_GITHUB_CLIENT_SECRET=...

# Callback URL base
PUBLIC_URL=https://pentagi.example.com
```

### Authorization (RBAC)

**Roles:**
- Admin (role_id=1): Full access
- User (role_id=2): Standard access

**Privilege Format:** `resource.action`

**Examples:**
```
flows.create
flows.delete
flows.edit
flows.view
users.create  # Admin only
users.delete  # Admin only
providers.view
prompts.edit
```

**Middleware:**
```go
// Authentication required
privateGroup.Use(authMiddleware.AuthRequired)

// Check specific privilege
if !user.HasPrivilege("flows.create") {
    return errors.New("unauthorized")
}
```

---

## WebSocket Subscriptions

### Connection

```javascript
// GraphQL WebSocket connection
const wsLink = new GraphQLWsLink(createClient({
  url: 'ws://localhost:8080/api/v1/graphql',
  connectionParams: {
    // Session cookie automatically sent
  }
}));
```

### Subscription Example

```graphql
subscription WatchFlowLogs($flowId: ID!) {
  messageLogAdded(flowId: $flowId) {
    id
    type
    message
    result
    resultFormat
    createdAt
  }
  
  terminalLogAdded(flowId: $flowId) {
    id
    type
    text
    terminal
    createdAt
  }
}
```

**Use Case:** Real-time updates in frontend as agents execute

---

## API Response Formats

### Success Response
```json
{
  "data": {
    "flow": {
      "id": "42",
      "title": "Web Application Pentest",
      "status": "running"
    }
  }
}
```

### Error Response
```json
{
  "errors": [
    {
      "message": "Flow not found",
      "path": ["flow"],
      "extensions": {
        "code": "NOT_FOUND"
      }
    }
  ]
}
```

### REST Error Response
```json
{
  "error": "unauthorized",
  "message": "You don't have permission to perform this action"
}
```

---

## Python Migration Recommendations

### GraphQL

**Library:** Strawberry GraphQL

```python
import strawberry
from typing import List, Optional

@strawberry.type
class Flow:
    id: strawberry.ID
    title: str
    status: StatusType
    terminals: List["Terminal"]
    provider: "Provider"
    created_at: datetime
    updated_at: datetime

@strawberry.type
class Query:
    @strawberry.field
    async def flows(self, info) -> List[Flow]:
        # Implementation
        pass
    
    @strawberry.field
    async def flow(self, info, flow_id: strawberry.ID) -> Optional[Flow]:
        # Implementation
        pass

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def create_flow(
        self, 
        info, 
        model_provider: str, 
        input: str
    ) -> Flow:
        # Implementation
        pass

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def message_log_added(
        self, 
        info, 
        flow_id: strawberry.ID
    ) -> AsyncGenerator[MessageLog, None]:
        # WebSocket subscription
        async for log in subscribe_to_logs(flow_id):
            yield log
```

### REST API

**Library:** FastAPI

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer

app = FastAPI(
    title="PentAGI API",
    version="2.0.0",
    docs_url="/api/v1/swagger"
)

security = HTTPBearer()

@app.get("/api/v1/flows", response_model=List[Flow])
async def list_flows(
    user: User = Depends(get_current_user)
):
    return await flow_service.list_flows(user.id)

@app.post("/api/v1/flows", response_model=Flow)
async def create_flow(
    data: CreateFlowRequest,
    user: User = Depends(get_current_user)
):
    return await flow_service.create_flow(user.id, data)

@app.delete("/api/v1/flows/{flow_id}")
async def delete_flow(
    flow_id: int,
    user: User = Depends(get_current_user)
):
    await flow_service.delete_flow(flow_id, user.id)
    return {"status": "success"}
```

### Authentication

**Library:** FastAPI with python-jose

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials"
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: int = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    user = await user_service.get_user(user_id)
    if user is None:
        raise credentials_exception
    return user
```

---

## Swagger Documentation

**Current:** Available at `/api/v1/swagger/` using swaggo/gin-swagger

**Python:** FastAPI auto-generates OpenAPI docs

**Access:**
- Swagger UI: `/api/v1/swagger/`
- ReDoc: `/api/v1/redoc/`
- OpenAPI JSON: `/api/v1/openapi.json`

---

## API Migration Complexity

| Component | Complexity | Reason |
|-----------|-----------|---------|
| GraphQL Schema | ⭐⭐ Medium | Direct port to Strawberry |
| REST Endpoints | ⭐ Easy | FastAPI similar to Gin |
| Authentication | ⭐⭐⭐ Medium-Hard | Session vs JWT decision |
| OAuth2 | ⭐⭐ Medium | python-social-auth available |
| WebSocket Subs | ⭐⭐⭐ Medium-Hard | AsyncIO subscriptions |
| RBAC | ⭐⭐ Medium | Straightforward port |

---

**Next:** Business Logic & Controllers Analysis

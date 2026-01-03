# Comprehensive Analysis of LLM Architecture in PentAGI Project

## 📋 Overview

This documentation provides a comprehensive analysis of all components related to the setup and architecture of Large Language Models (LLM) in the PentAGI project. This information was extracted from the project's source code in the `backend/pkg/providers/` directory.

---

## 1️⃣ LLM Service Providers

### Supported Providers List

PentAGI supports **6 main providers** for Large Language Models:

#### 1. OpenAI (`backend/pkg/providers/openai/`)
- **Type**: `ProviderOpenAI`
- **Main File**: `openai.go`
- **Config File**: `config.yml`
- **Models File**: `models.yml`

#### 2. Anthropic Claude (`backend/pkg/providers/anthropic/`)
- **Type**: `ProviderAnthropic`
- **Main File**: `anthropic.go`
- **Config File**: `config.yml`
- **Models File**: `models.yml`

#### 3. Google Gemini (`backend/pkg/providers/gemini/`)
- **Type**: `ProviderGemini`
- **Main File**: `gemini.go`
- **Config File**: `config.yml`
- **Models File**: `models.yml`

#### 4. AWS Bedrock (`backend/pkg/providers/bedrock/`)
- **Type**: `ProviderBedrock`
- **Main File**: `bedrock.go`
- **Config File**: `config.yml`
- **Models File**: `models.yml`

#### 5. Ollama (Local Hosting) (`backend/pkg/providers/ollama/`)
- **Type**: `ProviderOllama`
- **Main File**: `ollama.go`
- **Config File**: `config.yml`

#### 6. Custom APIs (OpenAI-Compatible) (`backend/pkg/providers/custom/`)
- **Type**: `ProviderCustom`
- **Main File**: `custom.go`
- **Description**: Supports any OpenAI-compatible API

---

## 2️⃣ Provider Configuration Details

### Provider Interface

Base interface definition in `backend/pkg/providers/provider/provider.go`:

```go
type Provider interface {
    Type() ProviderType
    GetRawConfig() []byte
    GetProviderConfig() *pconfig.ProviderConfig
    GetModels() pconfig.ModelsConfig
    Model(opt pconfig.ProviderOptionsType) string
    Call(ctx context.Context, optType pconfig.ProviderOptionsType, text string, options ...llms.CallOption) (string, error)
    Generate(ctx context.Context, optType pconfig.ProviderOptionsType, messages []llms.MessageContent, options ...llms.CallOption) (*llms.ContentResponse, error)
    GenerateFromSinglePrompt(ctx context.Context, optType pconfig.ProviderOptionsType, text string, options ...llms.CallOption) (*llms.ContentResponse, error)
    GetEmbedder() embeddings.Embedder
}
```

### Common Connection Options

All providers support the following options from `backend/pkg/providers/pconfig/config.go`:

#### AgentConfig Structure
```go
type AgentConfig struct {
    Model             string          // Model name
    MaxTokens         int             // Maximum tokens
    Temperature       float64         // Randomness (0-2)
    TopK              int             // Top-K sampling
    TopP              float64         // Top-P (nucleus) sampling
    N                 int             // Number of completions
    MinLength         int             // Minimum length
    MaxLength         int             // Maximum length
    RepetitionPenalty float64         // Repetition penalty
    FrequencyPenalty  float64         // Frequency penalty
    PresencePenalty   float64         // Presence penalty
    JSON              bool            // Enable JSON Mode
    ResponseMIMEType  string          // Response MIME type
    Reasoning         ReasoningConfig // Advanced reasoning config
    Price             *PriceInfo      // Pricing information
}
```

#### ReasoningConfig
```go
type ReasoningConfig struct {
    Effort    llms.ReasoningEffort // low, medium, high
    MaxTokens int                  // Maximum tokens
}
```

### Key Feature Support

| Feature | Support | Notes |
|---------|---------|-------|
| **Streaming** | ✅ All Providers | Live response streaming |
| **Tool Calling** | ✅ All Providers | Function calling support |
| **JSON Mode** | ✅ All Providers | Explicit JSON mode |
| **Reasoning** | OpenAI, Anthropic, Gemini | Advanced thinking capabilities |
| **Vision** | OpenAI, Anthropic, Gemini | Image support |
| **Embeddings** | ✅ All Providers | Generate embeddings |

### Error Handling and Retry

All providers use the same error handling mechanism:
- Automatic retry on failure
- Error logging with full context
- Observability support via Langfuse & OpenTelemetry

---

## 3️⃣ Model Configuration Per Provider

### 3.1 OpenAI

#### Available Models (from `models.yml`)

**GPT-5.2 Series** (Latest agentic models):
- `gpt-5.2`: Enhanced flagship model
  - Thinking: ✅
  - Release Date: 2025-12-11
  - Price: $1.75 input / $14.0 output (per million tokens)

**GPT-5 Series** (Advanced agentic models):
- `gpt-5`: Base model
- `gpt-5-mini`: Efficient version
- `gpt-5-nano`: Fastest model

**GPT-4.1 Series** (Enhanced models):
- `gpt-4.1`: Enhanced flagship
- `gpt-4.1-mini`: Balanced performance
- `gpt-4.1-nano`: Lightweight and fast

**GPT-4o Series** (Multimodal models):
- `gpt-4o`: Flagship with Vision
- `gpt-4o-mini`: Compact version

**O-Series** (Reasoning models):
- `o1`: Most powerful reasoning ($15/$60)
- `o3`: Advanced reasoning
- `o4-mini`: Optimized reasoning
- `o3-mini`: Compact reasoning

#### Default Configuration (from `config.yml`)

```yaml
simple:
  model: gpt-4.1-mini
  temperature: 0.5
  top_p: 0.5
  n: 1
  max_tokens: 3000

primary_agent:
  model: gpt-5
  n: 1
  max_tokens: 4000
  reasoning:
    effort: low

generator:
  model: gpt-5.2
  n: 1
  max_tokens: 8192
  reasoning:
    effort: medium

coder:
  model: gpt-5.2
  n: 1
  max_tokens: 6000
  reasoning:
    effort: medium

installer:
  model: o4-mini
  n: 1
  max_tokens: 6000
  reasoning:
    effort: low

pentester:
  model: o4-mini
  n: 1
  max_tokens: 4000
  reasoning:
    effort: low
```

### 3.2 Anthropic Claude

#### Available Models

**Claude 4 Series**:
- `claude-opus-4-5`: Most powerful ($5/$25)
- `claude-sonnet-4-5`: Advanced with deep reasoning
- `claude-sonnet-4-0`: Previous version
- `claude-haiku-4-5`: Fast and efficient

**Claude 3.7 Series**:
- `claude-3-7-sonnet-latest`: Extended thinking

**Claude 3.5 Series**:
- `claude-3-5-sonnet-latest`: High intelligence
- `claude-3-5-haiku-latest`: Ultra-fast

#### Default Configuration

```yaml
primary_agent:
  model: claude-sonnet-4-5
  temperature: 1.0
  n: 1
  max_tokens: 4000
  reasoning:
    max_tokens: 0

generator:
  model: claude-opus-4-5
  temperature: 1.0
  n: 1
  max_tokens: 16384

coder:
  model: claude-sonnet-4-5
  temperature: 1.0
  n: 1
  max_tokens: 6000
```

### 3.3 Google Gemini

#### Available Models

**Gemini 2.5 Series**:
- `gemini-2.5-pro`: Advanced flagship
- `gemini-2.5-flash`: Fast and optimized

**Gemini 2.0 Series**:
- `gemini-2.0-flash`: Balanced
- `gemini-2.0-flash-lite`: Lightweight

#### Default Configuration

```yaml
primary_agent:
  model: gemini-2.5-flash
  temperature: 0.8
  top_p: 0.95
  n: 1
  max_tokens: 6000
  reasoning:
    effort: medium

generator:
  model: gemini-2.5-pro
  temperature: 0.8
  top_p: 0.95
  n: 1
  max_tokens: 12000
  reasoning:
    effort: high
```

### 3.4 AWS Bedrock

Uses Claude models through AWS Bedrock:

```yaml
primary_agent:
  model: us.anthropic.claude-sonnet-4-20250514-v1:0
  temperature: 1.0
  n: 1
  max_tokens: 4000

coder:
  model: us.anthropic.claude-sonnet-4-20250514-v1:0
  temperature: 0.2
  top_p: 0.2
  n: 1
  max_tokens: 6000
```

### 3.5 Ollama (Local Hosting)

```yaml
simple:
  model: llama3.1:8b
  temperature: 0.2
  top_p: 0.3
  n: 1
  max_tokens: 4000

primary_agent:
  model: llama3.1:8b
  temperature: 0.2
  top_p: 0.3
  n: 1
  max_tokens: 4000
```

### 3.6 Custom API

Supports any OpenAI-compatible API. Configuration via environment variables:
- `LLM_SERVER_URL`: API endpoint
- `LLM_SERVER_KEY`: API key
- `LLM_SERVER_MODEL`: Default model
- `LLM_SERVER_CONFIG`: Config file path (optional)

---

## 4️⃣ Multi-Agent System

### Agent Types

From `backend/pkg/providers/pconfig/config.go`:

```go
const (
    OptionsTypePrimaryAgent ProviderOptionsType = "primary_agent"  // Main coordinator
    OptionsTypeAssistant    ProviderOptionsType = "assistant"     // Interactive assistant
    OptionsTypeSimple       ProviderOptionsType = "simple"        // Simple agent
    OptionsTypeSimpleJSON   ProviderOptionsType = "simple_json"   // Simple JSON agent
    OptionsTypeAdviser      ProviderOptionsType = "adviser"       // Advice provider
    OptionsTypeGenerator    ProviderOptionsType = "generator"     // Task generator
    OptionsTypeRefiner      ProviderOptionsType = "refiner"       // Plan refiner
    OptionsTypeSearcher     ProviderOptionsType = "searcher"      // Search agent
    OptionsTypeEnricher     ProviderOptionsType = "enricher"      // Information enricher
    OptionsTypeCoder        ProviderOptionsType = "coder"         // Coding agent
    OptionsTypeInstaller    ProviderOptionsType = "installer"     // Installation agent
    OptionsTypePentester    ProviderOptionsType = "pentester"     // Penetration testing
    OptionsTypeReflector    ProviderOptionsType = "reflector"     // Reflection agent
)
```

### Complete Agent List with Details

#### 1. **primary_agent** - Main Orchestration Agent
- **File**: `backend/pkg/templates/prompts/primary_agent.tmpl`
- **Role**: Primary task coordinator and work distributor
- **Available Tools**:
  - `{{.FinalyToolName}}`: Complete task
  - `{{.SearchToolName}}`: Search for information
  - `{{.PentesterToolName}}`: Penetration testing
  - `{{.CoderToolName}}`: Programming
  - `{{.AdviceToolName}}`: Request advice
  - `{{.MemoristToolName}}`: Memory management
  - `{{.MaintenanceToolName}}`: Maintenance
  - `{{.SummarizationToolName}}`: Summarization

- **Core Capabilities**:
  - Complex task analysis and breakdown
  - Delegation decision-making
  - Task context maintenance
  - Environment state verification

#### 2. **assistant** - Interactive Assistant Mode
- **File**: `backend/pkg/templates/prompts/assistant.tmpl`
- **Role**: Direct interactive assistant for users
- **Models Used**:
  - OpenAI: `gpt-5` (medium reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

#### 3. **pentester** - Penetration Testing Agent
- **File**: `backend/pkg/templates/prompts/pentester.tmpl`
- **Role**: Security testing and vulnerability exploitation
- **Models Used**:
  - OpenAI: `o4-mini` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

- **Available Tools**:
  - `{{.SearchGuideToolName}}`: Search testing guides
  - `{{.StoreGuideToolName}}`: Store methodologies
  - Network scanners
  - Exploitation frameworks
  - Privilege escalation tools

- **Capabilities**:
  - Vulnerability discovery
  - Attack execution
  - Security control bypass
  - Reconnaissance

#### 4. **coder** - Development Agent
- **File**: `backend/pkg/templates/prompts/coder.tmpl`
- **Role**: Elite developer for code and custom exploits
- **Models Used**:
  - OpenAI: `gpt-5.2` (medium reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

- **Available Tools**:
  - `{{.SearchCodeToolName}}`: Search previous code
  - `{{.StoreCodeToolName}}`: Store valuable code
  - Programming tools (all languages)
  - Build systems

- **Capabilities**:
  - High-quality code writing
  - Exploit modification
  - Tool development
  - Automation

#### 5. **installer** - Installation Agent
- **File**: `backend/pkg/templates/prompts/installer.tmpl`
- **Role**: Tool and package installation specialist
- **Models Used**:
  - OpenAI: `o4-mini` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-flash`

#### 6. **searcher** - Search Agent
- **File**: `backend/pkg/templates/prompts/searcher.tmpl`
- **Role**: Information gathering and technical research
- **Models Used**:
  - OpenAI: `gpt-4.1-mini`
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

- **Available Tools**:
  - Search engines
  - OSINT frameworks
  - Threat intelligence databases
  - Browser

#### 7. **memorist** - Memory Management Agent
- **File**: `backend/pkg/templates/prompts/memorist.tmpl`
- **Role**: Long-term memory and institutional knowledge management

#### 8. **adviser** - Advisory Agent
- **File**: `backend/pkg/templates/prompts/adviser.tmpl`
- **Role**: Strategic advice and guidance
- **Models Used**:
  - OpenAI: `gpt-5.2` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

#### 9. **generator** - Task Generation Agent
- **File**: `backend/pkg/templates/prompts/generator.tmpl`
- **Role**: Subtask plan generation
- **Models Used**:
  - OpenAI: `gpt-5.2` (medium reasoning)
  - Anthropic: `claude-opus-4-5`
  - Gemini: `gemini-2.5-pro` (high reasoning)

- **Available Tools**:
  - `{{.SubtaskListToolName}}`: Subtask management
  - `{{.SearchToolName}}`: Search
  - `{{.TerminalToolName}}`: Terminal
  - `{{.FileToolName}}`: Files
  - `{{.BrowserToolName}}`: Browser

#### 10. **refiner** - Plan Refinement Agent
- **File**: `backend/pkg/templates/prompts/refiner.tmpl`
- **Role**: Subtask plan refinement and modification
- **Models Used**:
  - OpenAI: `gpt-5` (high reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro` (medium reasoning)

- **Available Tools**:
  - `{{.SubtaskPatchToolName}}`: Task modification
  - All Generator tools

#### 11. **reporter** - Report Generation Agent
- **File**: `backend/pkg/templates/prompts/reporter.tmpl` & `task_reporter.tmpl`
- **Role**: Final task report generation
- **Available Tools**:
  - `{{.ReportResultToolName}}`: Report creation
  - `{{.SummarizationToolName}}`: Summarization

#### 12. **reflector** - Reflection and Correction Agent
- **File**: `backend/pkg/templates/prompts/reflector.tmpl`
- **Role**: Result review and error detection
- **Models Used**:
  - OpenAI: `o4-mini` (medium reasoning)
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

#### 13. **enricher** - Information Enrichment Agent
- **File**: `backend/pkg/templates/prompts/enricher.tmpl`
- **Role**: Information enrichment and expansion
- **Models Used**:
  - OpenAI: `gpt-4.1-mini`
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

#### 14. **toolcall_fixer** - Tool Call Fixer Agent
- **File**: `backend/pkg/templates/prompts/toolcall_fixer.tmpl`
- **Role**: Fix broken tool calls

#### 15. **summarizer** - Summarization Agent
- **File**: `backend/pkg/templates/prompts/summarizer.tmpl`
- **Role**: Long content summarization

---

## 5️⃣ Prompt Templates

### Complete Template File List

Location: `backend/pkg/templates/prompts/`

#### Main Agent Templates:
1. `primary_agent.tmpl` - Main orchestration agent
2. `assistant.tmpl` - Interactive assistant
3. `pentester.tmpl` - Penetration testing agent
4. `coder.tmpl` - Programming agent
5. `installer.tmpl` - Installation agent
6. `searcher.tmpl` - Search agent
7. `memorist.tmpl` - Memory management agent
8. `adviser.tmpl` - Advisory agent
9. `generator.tmpl` - Task generation agent
10. `refiner.tmpl` - Plan refinement agent
11. `reporter.tmpl` - Report generation agent
12. `reflector.tmpl` - Reflection agent
13. `enricher.tmpl` - Information enrichment agent

#### Question Templates for Agents:
14. `question_pentester.tmpl` - Penetration testing questions
15. `question_coder.tmpl` - Programming questions
16. `question_installer.tmpl` - Installation questions
17. `question_searcher.tmpl` - Search questions
18. `question_memorist.tmpl` - Memory questions
19. `question_adviser.tmpl` - Advisory questions
20. `question_enricher.tmpl` - Enrichment questions
21. `question_reflector.tmpl` - Reflection questions

#### Task Templates:
22. `subtasks_generator.tmpl` - Subtask generation
23. `subtasks_refiner.tmpl` - Subtask refinement
24. `task_reporter.tmpl` - Task reporting
25. `task_descriptor.tmpl` - Task description

#### Context Templates:
26. `full_execution_context.tmpl` - Full execution context
27. `short_execution_context.tmpl` - Short execution context
28. `execution_logs.tmpl` - Execution logs

#### Utility Templates:
29. `toolcall_fixer.tmpl` - Tool call fixing
30. `input_toolcall_fixer.tmpl` - Input tool call fixing
31. `summarizer.tmpl` - Summarization
32. `image_chooser.tmpl` - Image selection
33. `language_chooser.tmpl` - Language selection
34. `flow_descriptor.tmpl` - Flow description

---

## 6️⃣ System Prompt Structure

### System Prompt Components

Each agent has a system prompt consisting of:

1. **Agent Identity**
   - Role and responsibilities
   - Core capabilities

2. **Tool Usage Rules**
   ```
   - ALL actions MUST use structured tool calls
   - VERIFY tool call success/failure
   - AVOID redundant actions
   - PRIORITIZE minimally invasive tools
   ```

3. **Memory Protocol**
   ```
   - ALWAYS retrieve from memory FIRST
   - Leverage previous solutions
   - Store valuable discoveries
   ```

4. **Team Collaboration**
   - List of specialist agents
   - When to delegate to each agent
   - Available tools per agent

5. **Graphiti Integration (if enabled)**
   - Historical context search
   - Institutional knowledge usage

### Example: Primary Agent System Prompt

```
# TEAM ORCHESTRATION MANAGER

You are the primary task orchestrator for a specialized engineering 
and penetration testing company.

## CORE CAPABILITIES / KNOWLEDGE BASE
- Skilled at analyzing complex tasks
- Expert at delegation decision-making
- Proficient at maintaining task context
- Capable of verifying environment state

## TOOL EXECUTION RULES
- ALL actions MUST use structured tool calls
- VERIFY tool call success/failure
- AVOID redundant actions

## MEMORY SYSTEM INTEGRATION
- ALWAYS attempt to retrieve relevant information from memory FIRST
- Leverage previously stored solutions

## TEAM COLLABORATION & DELEGATION
<specialist name="searcher">
  <skills>Information gathering, research</skills>
  <use_cases>Find information, create guides</use_cases>
  <tool_name>{{.SearchToolName}}</tool_name>
</specialist>

<specialist name="pentester">
  <skills>Security testing, exploitation</skills>
  <use_cases>Discover vulnerabilities, bypass controls</use_cases>
  <tool_name>{{.PentesterToolName}}</tool_name>
</specialist>
...
```

---

## 7️⃣ Delegation Mechanism

### How Delegation Works

1. **Task Analysis**: Primary agent analyzes the task
2. **Agent Selection**: Based on task type
3. **Context Preparation**: Prepare `ExecutionContext` for agent
4. **Agent Invocation**: Via Tool Calling
5. **Result Processing**: Receive and process agent result
6. **Continue or Complete**: Based on task state

### Execution Context

From `backend/pkg/providers/provider.go`:

```go
type FlowProvider interface {
    PrepareAgentChain(ctx context.Context, taskID, subtaskID int64) (int64, error)
    PerformAgentChain(ctx context.Context, taskID, subtaskID, msgChainID int64) (PerformResult, error)
    PutInputToAgentChain(ctx context.Context, msgChainID int64, input string) error
    EnsureChainConsistency(ctx context.Context, msgChainID int64) error
}
```

### Perform Results

```go
const (
    PerformResultError   PerformResult = iota // Error
    PerformResultWaiting                      // Waiting for input
    PerformResultDone                         // Completed
)
```

---

## 8️⃣ Streaming Support

### StreamMessageChunk Types

```go
const (
    StreamMessageChunkTypeThinking StreamMessageChunkType = "thinking"
    StreamMessageChunkTypeContent  StreamMessageChunkType = "content"
    StreamMessageChunkTypeResult   StreamMessageChunkType = "result"
    StreamMessageChunkTypeFlush    StreamMessageChunkType = "flush"
    StreamMessageChunkTypeUpdate   StreamMessageChunkType = "update"
)
```

### StreamMessageHandler

```go
type StreamMessageHandler func(ctx context.Context, chunk *StreamMessageChunk) error
```

Allows progressive response processing during generation.

---

## 9️⃣ Embeddings Support

All providers support embeddings generation via:

```go
type Embedder interface {
    EmbedDocuments(ctx context.Context, texts []string) ([][]float64, error)
    EmbedQuery(ctx context.Context, text string) ([]float64, error)
}
```

Location: `backend/pkg/providers/embeddings/embedder.go`

---

## 🔟 Observability Integration

### Langfuse Integration

All agents are integrated with Langfuse for monitoring:

```go
observation.Span(
    langfuse.WithStartSpanName("primary agent"),
    langfuse.WithStartSpanInput(chain),
    langfuse.WithStartSpanMetadata(metadata),
)
```

### OpenTelemetry Support

Full OpenTelemetry Tracing support:

```go
ctx, span := obs.Observer.NewSpan(ctx, obs.SpanKindInternal, "agent.execute")
defer span.End()
```

---

## 1️⃣1️⃣ Message Limits

```go
const (
    msgGeneratorSizeLimit = 150 * 1024 // 150 KB
    msgRefinerSizeLimit   = 100 * 1024 // 100 KB
    msgReporterSizeLimit  = 100 * 1024 // 100 KB
    msgSummarizerLimit    = 16 * 1024  // 16 KB
)
```

When these limits are exceeded, content is automatically summarized.

---

## 1️⃣2️⃣ Graphiti Support (Knowledge Graph)

### What is Graphiti?

Graphiti is a temporal knowledge graph system that stores:
- All previous agent responses
- Tool execution records
- Entity relationships
- Historical context

### Graphiti Search Types

1. **recent_context** - Recent context
2. **successful_tools** - Successful tools
3. **episode_context** - Complete episode context
4. **diverse_results** - Diverse results

---

## 1️⃣3️⃣ System Tools

### Tools Available to Agents:

1. **FinalyToolName** - Complete task
2. **AskUserToolName** - Ask user
3. **SearchToolName** - Search
4. **PentesterToolName** - Penetration testing
5. **CoderToolName** - Programming
6. **InstallerToolName** - Installation
7. **AdviceToolName** - Advice
8. **MemoristToolName** - Memory
9. **MaintenanceToolName** - Maintenance
10. **SummarizationToolName** - Summarization
11. **SubtaskListToolName** - Subtask management
12. **SubtaskPatchToolName** - Task modification
13. **ReportResultToolName** - Report creation
14. **TerminalToolName** - Terminal
15. **FileToolName** - Files
16. **BrowserToolName** - Browser
17. **SearchCodeToolName** - Search code
18. **StoreCodeToolName** - Store code
19. **SearchGuideToolName** - Search guides
20. **StoreGuideToolName** - Store guides

---

## 1️⃣4️⃣ Model Selection Strategy

### General Principles:

1. **Main and Complex Tasks**:
   - OpenAI: `gpt-5`, `gpt-5.2`, `o4-mini`
   - Anthropic: `claude-sonnet-4-5`, `claude-opus-4-5`
   - Gemini: `gemini-2.5-pro`

2. **Fast and Simple Tasks**:
   - OpenAI: `gpt-4.1-mini`
   - Anthropic: `claude-haiku-4-5`
   - Gemini: `gemini-2.0-flash`

3. **Deep Reasoning Tasks**:
   - OpenAI: `o1`, `o3`, `o4-mini`
   - Anthropic: `claude-opus-4-5`
   - Gemini: `gemini-2.5-pro` (high reasoning)

4. **Programming Tasks**:
   - OpenAI: `gpt-5.2`
   - Anthropic: `claude-sonnet-4-5`
   - Gemini: `gemini-2.5-pro`

---

## 1️⃣5️⃣ Pricing Information

### Cost Comparison (per million tokens)

#### Most Expensive Models:
- OpenAI `o1`: $15 / $60
- Anthropic `claude-opus-4-5`: $5 / $25

#### Mid-Range Models:
- OpenAI `gpt-5`: $1.25 / $10
- OpenAI `gpt-5.2`: $1.75 / $14
- Anthropic `claude-sonnet-4-5`: $3 / $15
- Gemini `gemini-2.5-pro`: $1.25 / $10

#### Economy Models:
- OpenAI `gpt-4.1-mini`: $0.4 / $1.6
- Anthropic `claude-haiku-4-5`: $1 / $5
- Gemini `gemini-2.0-flash`: $0.1 / $0.4

---

## 📚 Summary

### Key Statistics:

- **Number of Providers**: 6 (OpenAI, Anthropic, Gemini, Bedrock, Ollama, Custom)
- **Number of Agents**: 15 specialized agents
- **Number of Prompt Templates**: 34 templates
- **Number of Tools**: 20+ available tools
- **Streaming Support**: ✅ Yes
- **Tool Calling Support**: ✅ Yes
- **JSON Mode Support**: ✅ Yes
- **Vision Support**: ✅ Yes (OpenAI, Anthropic, Gemini)
- **Reasoning Support**: ✅ Yes (OpenAI o-series, Anthropic Claude 4, Gemini 2.5)
- **Embeddings Support**: ✅ Yes
- **Observability Support**: ✅ Yes (Langfuse, OpenTelemetry)

---

## 🔗 Reference Files

### Main Files:
- `backend/pkg/providers/provider.go` - Base interface
- `backend/pkg/providers/pconfig/config.go` - Configuration system
- `backend/pkg/providers/providers.go` - Provider manager
- `backend/pkg/providers/performer.go` - Agent execution logic
- `backend/pkg/templates/templates.go` - Template system

### Provider Files:
- `backend/pkg/providers/openai/openai.go`
- `backend/pkg/providers/anthropic/anthropic.go`
- `backend/pkg/providers/gemini/gemini.go`
- `backend/pkg/providers/bedrock/bedrock.go`
- `backend/pkg/providers/ollama/ollama.go`
- `backend/pkg/providers/custom/custom.go`

---

## 📝 Final Notes

This analysis covers the essential aspects of the LLM architecture in the PentAGI project. The system is designed in a highly modular and flexible manner, with multi-provider support and a sophisticated agent system using the latest AI technologies.

**Analysis Date**: January 2026
**Project Version**: PentAGI
**Source**: https://github.com/raglox/pentagi

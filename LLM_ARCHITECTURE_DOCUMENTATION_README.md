# PentAGI LLM Architecture Analysis Documentation

## 📘 Overview

This repository now includes comprehensive documentation analyzing the entire Large Language Model (LLM) architecture of the PentAGI project. The analysis was conducted in response to a request for extracting and documenting all LLM-related components.

## 📄 Documentation Files

### 1. **LLM_ARCHITECTURE_ANALYSIS_AR.md** (Arabic Version)
Complete analysis in Arabic covering:
- ✅ All 6 LLM providers (OpenAI, Anthropic, Gemini, Bedrock, Ollama, Custom)
- ✅ Configuration details for each provider
- ✅ All 15 specialized agents
- ✅ 34 prompt templates
- ✅ System prompts for each agent
- ✅ 20+ available tools
- ✅ Model selection strategies
- ✅ Pricing information

### 2. **LLM_ARCHITECTURE_ANALYSIS_EN.md** (English Version)
Complete analysis in English with the same comprehensive coverage.

## 📊 What's Covered

### 1️⃣ LLM Service Providers
Detailed analysis of all 6 providers:
1. **OpenAI** - 15+ models including GPT-5, GPT-4.1, o-series
2. **Anthropic Claude** - Claude 4, 3.7, and 3.5 series
3. **Google Gemini** - Gemini 2.5 and 2.0 series
4. **AWS Bedrock** - Claude models via AWS
5. **Ollama** - Local hosting with open-source models
6. **Custom APIs** - OpenAI-compatible custom endpoints

### 2️⃣ Provider Configuration
For each provider:
- ✅ Default configuration files (`config.yml`)
- ✅ Available models list (`models.yml`)
- ✅ API connection options
- ✅ Streaming support
- ✅ Tool calling capabilities
- ✅ JSON mode support
- ✅ Reasoning/thinking capabilities
- ✅ Error handling and retry mechanisms

### 3️⃣ Multi-Agent System
Complete documentation of all 15 agents:
1. **primary_agent** - Main orchestration coordinator
2. **assistant** - Interactive assistant mode
3. **pentester** - Penetration testing specialist
4. **coder** - Development and programming
5. **installer** - Tool and package installation
6. **searcher** - Information gathering and research
7. **memorist** - Memory management
8. **adviser** - Strategic advice provider
9. **generator** - Task generation
10. **refiner** - Plan refinement
11. **reporter** - Report generation
12. **reflector** - Reflection and correction
13. **enricher** - Information enrichment
14. **toolcall_fixer** - Tool call fixing
15. **summarizer** - Content summarization

### 4️⃣ Prompt Templates
All 34 prompt templates documented:
- Agent templates (13 main agents)
- Question templates (8 question formats)
- Task templates (4 task types)
- Context templates (3 context types)
- Utility templates (6 utilities)

### 5️⃣ System Architecture
- **AgentConfig Structure** - Complete configuration schema
- **ReasoningConfig** - Advanced reasoning configuration
- **ProviderOptionsType** - All agent type constants
- **FlowProvider Interface** - Agent execution interface
- **Delegation Mechanism** - How agents coordinate
- **Streaming Support** - Live response streaming
- **Embeddings Support** - Vector embeddings generation
- **Observability Integration** - Langfuse & OpenTelemetry

## 🎯 Key Features Documented

### Provider Features Matrix

| Feature | OpenAI | Anthropic | Gemini | Bedrock | Ollama | Custom |
|---------|--------|-----------|--------|---------|--------|--------|
| **Streaming** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Tool Calling** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **JSON Mode** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Reasoning** | ✅ | ✅ | ✅ | ❌ | ❌ | Depends |
| **Vision** | ✅ | ✅ | ✅ | ✅ | ❌ | Depends |
| **Embeddings** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Agent Capabilities Matrix

| Agent | Primary Model (OpenAI) | Reasoning | Main Purpose |
|-------|------------------------|-----------|--------------|
| **primary_agent** | gpt-5 | Low | Task orchestration |
| **generator** | gpt-5.2 | Medium | Subtask generation |
| **refiner** | gpt-5 | High | Plan refinement |
| **coder** | gpt-5.2 | Medium | Code development |
| **pentester** | o4-mini | Low | Security testing |
| **installer** | o4-mini | Low | Tool installation |
| **searcher** | gpt-4.1-mini | None | Information gathering |
| **enricher** | gpt-4.1-mini | None | Info enrichment |
| **reflector** | o4-mini | Medium | Reflection |
| **adviser** | gpt-5.2 | Low | Strategic advice |

## 💰 Pricing Information

### Most Expensive Models (per million tokens)
- **OpenAI o1**: $15 input / $60 output
- **Anthropic Claude Opus 4-5**: $5 input / $25 output

### Mid-Range Models
- **OpenAI GPT-5**: $1.25 / $10
- **OpenAI GPT-5.2**: $1.75 / $14
- **Anthropic Claude Sonnet 4-5**: $3 / $15
- **Gemini 2.5 Pro**: $1.25 / $10

### Economy Models
- **OpenAI GPT-4.1-mini**: $0.4 / $1.6
- **Anthropic Claude Haiku 4-5**: $1 / $5
- **Gemini 2.0 Flash**: $0.1 / $0.4

## 🔧 Technical Details

### Configuration Structure
```go
type AgentConfig struct {
    Model             string          // Model name
    MaxTokens         int             // Maximum tokens
    Temperature       float64         // Randomness (0-2)
    TopK              int             // Top-K sampling
    TopP              float64         // Top-P sampling
    N                 int             // Number of completions
    JSON              bool            // JSON mode
    Reasoning         ReasoningConfig // Reasoning config
    Price             *PriceInfo      // Pricing info
}
```

### Provider Interface
```go
type Provider interface {
    Type() ProviderType
    GetProviderConfig() *pconfig.ProviderConfig
    GetModels() pconfig.ModelsConfig
    Call(ctx, optType, text, options) (string, error)
    Generate(ctx, optType, messages, options) (*llms.ContentResponse, error)
    GetEmbedder() embeddings.Embedder
}
```

## 📚 System Tools

20+ tools available to agents:
- **Task Management**: FinalyToolName, AskUserToolName, SubtaskListToolName
- **Development**: CoderToolName, InstallerToolName, TerminalToolName, FileToolName
- **Security**: PentesterToolName
- **Information**: SearchToolName, BrowserToolName, SearchCodeToolName
- **Memory**: MemoristToolName, StoreCodeToolName, StoreGuideToolName
- **Utilities**: SummarizationToolName, MaintenanceToolName, AdviceToolName
- **Reporting**: ReportResultToolName

## 🎓 Key Insights

### 1. Modular Architecture
- Highly flexible provider system supporting multiple LLM backends
- Easy to add new providers or switch between them
- Unified interface for all providers

### 2. Sophisticated Agent System
- 15 specialized agents each with specific expertise
- Intelligent delegation mechanism
- Memory and knowledge graph integration (Graphiti)

### 3. Advanced Features
- Full streaming support for real-time responses
- Tool calling for autonomous task execution
- Reasoning capabilities for complex problem-solving
- Observability integration for monitoring and debugging

### 4. Cost Optimization
- Multiple model tiers for different task complexities
- Smart agent selection based on task requirements
- Support for local hosting (Ollama) to reduce costs

## 📖 Usage

### Accessing the Documentation

1. **Arabic Documentation**: `LLM_ARCHITECTURE_ANALYSIS_AR.md`
   - Comprehensive analysis in Arabic
   - Perfect for Arabic-speaking developers and analysts

2. **English Documentation**: `LLM_ARCHITECTURE_ANALYSIS_EN.md`
   - Complete English version
   - Accessible to international audience

### What You'll Find

Each documentation file includes:
- **Complete provider configurations** with all available models
- **Agent system architecture** with detailed role descriptions
- **Prompt template analysis** showing how each agent is instructed
- **Tool descriptions** explaining available capabilities
- **Delegation mechanisms** showing how agents coordinate
- **Pricing information** for cost planning
- **Model selection strategies** for optimal performance

## 🔍 Source Files Referenced

All information was extracted from:
- `backend/pkg/providers/` - Provider implementations
- `backend/pkg/providers/pconfig/config.go` - Configuration system
- `backend/pkg/providers/openai/` - OpenAI provider
- `backend/pkg/providers/anthropic/` - Anthropic provider
- `backend/pkg/providers/gemini/` - Gemini provider
- `backend/pkg/providers/bedrock/` - Bedrock provider
- `backend/pkg/providers/ollama/` - Ollama provider
- `backend/pkg/providers/custom/` - Custom API provider
- `backend/pkg/templates/prompts/` - All 34 prompt templates

## ✨ Highlights

### Statistics
- **6 LLM Providers** fully documented
- **15 Specialized Agents** with complete descriptions
- **34 Prompt Templates** analyzed
- **20+ System Tools** catalogued
- **50+ Models** listed with specifications
- **Full Configuration Examples** for all providers
- **Pricing Data** for cost estimation

### Coverage
- ✅ Provider setup and configuration
- ✅ Model capabilities and specifications
- ✅ Agent roles and responsibilities
- ✅ Tool calling and function execution
- ✅ Streaming and real-time responses
- ✅ Reasoning and thinking capabilities
- ✅ Memory and knowledge management
- ✅ Observability and monitoring
- ✅ Error handling and retry logic
- ✅ Cost optimization strategies

## 🌟 Benefits

This documentation provides:
1. **Complete Understanding** of PentAGI's LLM architecture
2. **Configuration Reference** for setting up providers
3. **Agent Selection Guide** for task delegation
4. **Cost Planning** with detailed pricing information
5. **Developer Guide** for extending the system
6. **Troubleshooting Reference** with error handling details

## 📝 Document Quality

Both documents are:
- ✅ Comprehensive and detailed
- ✅ Well-organized with clear sections
- ✅ Rich with code examples
- ✅ Includes tables and matrices for easy reference
- ✅ Contains pricing and cost information
- ✅ References source files for verification
- ✅ Professional and technical in nature

## 🤝 Contributing

This analysis provides a foundation for:
- Understanding the system architecture
- Configuring and customizing providers
- Developing new agents or tools
- Optimizing model selection
- Cost management and planning

## 📅 Metadata

- **Analysis Date**: January 2026
- **Project**: PentAGI
- **Repository**: https://github.com/raglox/pentagi
- **Languages**: Arabic (AR) and English (EN)
- **Format**: Markdown (.md)

---

## 🎯 Quick Navigation

- 📄 [Arabic Documentation (العربية)](./LLM_ARCHITECTURE_ANALYSIS_AR.md)
- 📄 [English Documentation](./LLM_ARCHITECTURE_ANALYSIS_EN.md)
- 🏠 [Main Repository README](./README.md)

---

**Note**: This analysis was created by extracting and documenting information from the PentAGI source code to provide a comprehensive reference for developers, analysts, and users working with the system's LLM architecture.

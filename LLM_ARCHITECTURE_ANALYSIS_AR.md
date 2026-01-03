# تحليل شامل لبنية نماذج اللغة الكبيرة في مشروع PentAGI

## 📋 نظرة عامة

هذا التوثيق يقدم تحليلاً شاملاً لجميع مكونات إعداد وهندسة نماذج اللغة الكبيرة (LLM) في مشروع PentAGI. تم استخراج هذه المعلومات من الكود المصدري للمشروع في الدليل `backend/pkg/providers/`.

---

## 1️⃣ مزودو خدمات LLM (Providers)

### قائمة المزودين المدعومين

يدعم PentAGI **6 مزودين** رئيسيين لنماذج اللغة الكبيرة:

#### 1. OpenAI (`backend/pkg/providers/openai/`)
- **النوع**: `ProviderOpenAI`
- **الملف الرئيسي**: `openai.go`
- **ملف التكوين**: `config.yml`
- **ملف النماذج**: `models.yml`

#### 2. Anthropic Claude (`backend/pkg/providers/anthropic/`)
- **النوع**: `ProviderAnthropic`
- **الملف الرئيسي**: `anthropic.go`
- **ملف التكوين**: `config.yml`
- **ملف النماذج**: `models.yml`

#### 3. Google Gemini (`backend/pkg/providers/gemini/`)
- **النوع**: `ProviderGemini`
- **الملف الرئيسي**: `gemini.go`
- **ملف التكوين**: `config.yml`
- **ملف النماذج**: `models.yml`

#### 4. AWS Bedrock (`backend/pkg/providers/bedrock/`)
- **النوع**: `ProviderBedrock`
- **الملف الرئيسي**: `bedrock.go`
- **ملف التكوين**: `config.yml`
- **ملف النماذج**: `models.yml`

#### 5. Ollama (استضافة محلية) (`backend/pkg/providers/ollama/`)
- **النوع**: `ProviderOllama`
- **الملف الرئيسي**: `ollama.go`
- **ملف التكوين**: `config.yml`

#### 6. Custom APIs (متوافقة مع OpenAI) (`backend/pkg/providers/custom/`)
- **النوع**: `ProviderCustom`
- **الملف الرئيسي**: `custom.go`
- **الوصف**: يدعم أي API متوافقة مع بروتوكول OpenAI

---

## 2️⃣ تفاصيل تكوين كل مزود (Provider Configuration)

### واجهة المزود (Provider Interface)

تعريف الواجهة الأساسية في `backend/pkg/providers/provider/provider.go`:

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

### خيارات الاتصال العامة

تدعم جميع المزودات الخيارات التالية من `backend/pkg/providers/pconfig/config.go`:

#### AgentConfig Structure
```go
type AgentConfig struct {
    Model             string          // اسم النموذج
    MaxTokens         int             // الحد الأقصى للتوكنات
    Temperature       float64         // درجة العشوائية (0-2)
    TopK              int             // Top-K sampling
    TopP              float64         // Top-P (nucleus) sampling
    N                 int             // عدد الإكمالات المطلوبة
    MinLength         int             // الحد الأدنى للطول
    MaxLength         int             // الحد الأقصى للطول
    RepetitionPenalty float64         // عقوبة التكرار
    FrequencyPenalty  float64         // عقوبة التكرار حسب التردد
    PresencePenalty   float64         // عقوبة التكرار حسب الوجود
    JSON              bool            // تفعيل JSON Mode
    ResponseMIMEType  string          // نوع MIME للاستجابة
    Reasoning         ReasoningConfig // تكوين التفكير المتقدم
    Price             *PriceInfo      // معلومات التسعير
}
```

#### ReasoningConfig
```go
type ReasoningConfig struct {
    Effort    llms.ReasoningEffort // low, medium, high
    MaxTokens int                  // الحد الأقصى للتوكنات
}
```

### دعم الميزات الرئيسية

| الميزة | الدعم | الملاحظات |
|--------|-------|-----------|
| **Streaming** | ✅ جميع المزودين | دعم البث المباشر للردود |
| **Tool Calling** | ✅ جميع المزودين | استدعاء الأدوات (Function Calling) |
| **JSON Mode** | ✅ جميع المزودين | وضع JSON الصريح |
| **Reasoning** | OpenAI, Anthropic, Gemini | التفكير المتقدم (Thinking) |
| **Vision** | OpenAI, Anthropic, Gemini | دعم الصور |
| **Embeddings** | ✅ جميع المزودين | توليد التضمينات (Embeddings) |

### معالجة الأخطاء وإعادة المحاولة

جميع المزودين يستخدمون نفس آلية معالجة الأخطاء:
- إعادة المحاولة التلقائية عند الفشل
- تسجيل الأخطاء مع السياق الكامل
- دعم Observability عبر Langfuse و OpenTelemetry

---

## 3️⃣ تكوين النماذج لكل مزود

### 3.1 OpenAI

#### النماذج المتاحة (من `models.yml`)

**سلسلة GPT-5.2** (أحدث نماذج Agentic):
- `gpt-5.2`: النموذج الرئيسي المحسّن
  - Thinking: ✅
  - تاريخ الإصدار: 2025-12-11
  - السعر: $1.75 input / $14.0 output (لكل مليون توكن)

**سلسلة GPT-5** (نماذج Agentic متقدمة):
- `gpt-5`: النموذج الأساسي
- `gpt-5-mini`: نسخة فعالة
- `gpt-5-nano`: أسرع نموذج

**سلسلة GPT-4.1** (نماذج محسّنة):
- `gpt-4.1`: نموذج رئيسي محسّن
- `gpt-4.1-mini`: متوازن الأداء
- `gpt-4.1-nano`: خفيف وسريع

**سلسلة GPT-4o** (نماذج متعددة الوسائط):
- `gpt-4o`: نموذج رئيسي مع Vision
- `gpt-4o-mini`: نسخة مدمجة

**سلسلة O-series** (نماذج التفكير):
- `o1`: أقوى نموذج تفكير ($15/$60)
- `o3`: نموذج تفكير متقدم
- `o4-mini`: نموذج تفكير محسّن
- `o3-mini`: نموذج تفكير مدمج

#### التكوين الافتراضي (من `config.yml`)

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

#### النماذج المتاحة

**سلسلة Claude 4**:
- `claude-opus-4-5`: أقوى نموذج ($5/$25)
- `claude-sonnet-4-5`: نموذج متقدم بتفكير عميق
- `claude-sonnet-4-0`: نسخة سابقة
- `claude-haiku-4-5`: سريع وفعال

**سلسلة Claude 3.7**:
- `claude-3-7-sonnet-latest`: مع تفكير ممتد

**سلسلة Claude 3.5**:
- `claude-3-5-sonnet-latest`: ذكاء عالي
- `claude-3-5-haiku-latest`: سريع جداً

#### التكوين الافتراضي

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

#### النماذج المتاحة

**سلسلة Gemini 2.5**:
- `gemini-2.5-pro`: نموذج رئيسي متقدم
- `gemini-2.5-flash`: سريع ومحسّن

**سلسلة Gemini 2.0**:
- `gemini-2.0-flash`: متوازن
- `gemini-2.0-flash-lite`: خفيف وسريع

#### التكوين الافتراضي

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

يستخدم نماذج Claude من خلال AWS Bedrock:

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

### 3.5 Ollama (استضافة محلية)

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

يدعم أي API متوافقة مع OpenAI. التكوين يتم عبر متغيرات البيئة:
- `LLM_SERVER_URL`: عنوان API
- `LLM_SERVER_KEY`: مفتاح API
- `LLM_SERVER_MODEL`: النموذج الافتراضي
- `LLM_SERVER_CONFIG`: مسار ملف التكوين (اختياري)

---

## 4️⃣ نظام الوكلاء المتعددة (Multi-Agent System)

### أنواع الوكلاء (Agent Types)

من `backend/pkg/providers/pconfig/config.go`:

```go
const (
    OptionsTypePrimaryAgent ProviderOptionsType = "primary_agent"  // المنسق الرئيسي
    OptionsTypeAssistant    ProviderOptionsType = "assistant"     // المساعد التفاعلي
    OptionsTypeSimple       ProviderOptionsType = "simple"        // وكيل بسيط
    OptionsTypeSimpleJSON   ProviderOptionsType = "simple_json"   // وكيل JSON بسيط
    OptionsTypeAdviser      ProviderOptionsType = "adviser"       // تقديم النصائح
    OptionsTypeGenerator    ProviderOptionsType = "generator"     // توليد المهام
    OptionsTypeRefiner      ProviderOptionsType = "refiner"       // تحسين الخطط
    OptionsTypeSearcher     ProviderOptionsType = "searcher"      // البحث
    OptionsTypeEnricher     ProviderOptionsType = "enricher"      // إثراء المعلومات
    OptionsTypeCoder        ProviderOptionsType = "coder"         // البرمجة
    OptionsTypeInstaller    ProviderOptionsType = "installer"     // التثبيت
    OptionsTypePentester    ProviderOptionsType = "pentester"     // اختبار الاختراق
    OptionsTypeReflector    ProviderOptionsType = "reflector"     // المراجعة
)
```

### قائمة الوكلاء الكاملة مع التفاصيل

#### 1. **primary_agent** - الوكيل المنسق الرئيسي
- **الملف**: `backend/pkg/templates/prompts/primary_agent.tmpl`
- **الدور**: منسق المهام الرئيسي وموزع العمل على الوكلاء المتخصصين
- **الأدوات المتاحة**:
  - `{{.FinalyToolName}}`: إنهاء المهمة
  - `{{.SearchToolName}}`: البحث عن المعلومات
  - `{{.PentesterToolName}}`: اختبار الاختراق
  - `{{.CoderToolName}}`: البرمجة
  - `{{.AdviceToolName}}`: طلب المشورة
  - `{{.MemoristToolName}}`: إدارة الذاكرة
  - `{{.MaintenanceToolName}}`: الصيانة
  - `{{.SummarizationToolName}}`: التلخيص

- **القدرات الأساسية**:
  - تحليل المهام المعقدة وتقسيمها
  - اتخاذ قرارات التفويض
  - الحفاظ على سياق المهام
  - التحقق من حالة البيئة

#### 2. **assistant** - وضع المساعد التفاعلي
- **الملف**: `backend/pkg/templates/prompts/assistant.tmpl`
- **الدور**: مساعد تفاعلي مباشر للمستخدم
- **النماذج المستخدمة**:
  - OpenAI: `gpt-5` (medium reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

#### 3. **pentester** - وكيل اختبار الاختراق
- **الملف**: `backend/pkg/templates/prompts/pentester.tmpl`
- **الدور**: متخصص في اختبار الاختراق واستغلال الثغرات
- **النماذج المستخدمة**:
  - OpenAI: `o4-mini` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

- **الأدوات المتاحة**:
  - `{{.SearchGuideToolName}}`: البحث في أدلة الاختبار
  - `{{.StoreGuideToolName}}`: حفظ المنهجيات
  - أدوات فحص الشبكات
  - أدوات الاستغلال (Exploitation)
  - أدوات رفع الصلاحيات

- **القدرات**:
  - اكتشاف الثغرات
  - تنفيذ الهجمات
  - تجاوز الضوابط الأمنية
  - الاستطلاع (Reconnaissance)

#### 4. **coder** - وكيل البرمجة
- **الملف**: `backend/pkg/templates/prompts/coder.tmpl`
- **الدور**: مطور متخصص في كتابة الأكواد والاستغلالات المخصصة
- **النماذج المستخدمة**:
  - OpenAI: `gpt-5.2` (medium reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

- **الأدوات المتاحة**:
  - `{{.SearchCodeToolName}}`: البحث عن أكواد سابقة
  - `{{.StoreCodeToolName}}`: حفظ الأكواد القيمة
  - أدوات البرمجة بجميع اللغات
  - أدوات بناء الأنظمة

- **القدرات**:
  - كتابة أكواد عالية الجودة
  - تعديل الاستغلالات (Exploits)
  - تطوير الأدوات
  - الأتمتة (Automation)

#### 5. **installer** - وكيل التثبيت
- **الملف**: `backend/pkg/templates/prompts/installer.tmpl`
- **الدور**: متخصص في تثبيت الأدوات والحزم
- **النماذج المستخدمة**:
  - OpenAI: `o4-mini` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-flash`

#### 6. **searcher** - وكيل البحث
- **الملف**: `backend/pkg/templates/prompts/searcher.tmpl`
- **الدور**: جمع المعلومات والبحث التقني
- **النماذج المستخدمة**:
  - OpenAI: `gpt-4.1-mini`
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

- **الأدوات المتاحة**:
  - محركات البحث
  - أطر OSINT
  - قواعد بيانات الاستخبارات
  - المتصفح

#### 7. **memorist** - وكيل إدارة الذاكرة
- **الملف**: `backend/pkg/templates/prompts/memorist.tmpl`
- **الدور**: إدارة الذاكرة طويلة المدى والمعرفة المؤسسية

#### 8. **adviser** - وكيل تقديم النصائح
- **الملف**: `backend/pkg/templates/prompts/adviser.tmpl`
- **الدور**: تقديم المشورة والإرشادات الاستراتيجية
- **النماذج المستخدمة**:
  - OpenAI: `gpt-5.2` (low reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro`

#### 9. **generator** - وكيل توليد المهام
- **الملف**: `backend/pkg/templates/prompts/generator.tmpl`
- **الدور**: توليد خطط المهام الفرعية
- **النماذج المستخدمة**:
  - OpenAI: `gpt-5.2` (medium reasoning)
  - Anthropic: `claude-opus-4-5`
  - Gemini: `gemini-2.5-pro` (high reasoning)

- **الأدوات المتاحة**:
  - `{{.SubtaskListToolName}}`: إدارة المهام الفرعية
  - `{{.SearchToolName}}`: البحث
  - `{{.TerminalToolName}}`: الطرفية
  - `{{.FileToolName}}`: الملفات
  - `{{.BrowserToolName}}`: المتصفح

#### 10. **refiner** - وكيل تحسين الخطط
- **الملف**: `backend/pkg/templates/prompts/refiner.tmpl`
- **الدور**: تحسين وتعديل خطط المهام الفرعية
- **النماذج المستخدمة**:
  - OpenAI: `gpt-5` (high reasoning)
  - Anthropic: `claude-sonnet-4-5`
  - Gemini: `gemini-2.5-pro` (medium reasoning)

- **الأدوات المتاحة**:
  - `{{.SubtaskPatchToolName}}`: تعديل المهام
  - جميع أدوات Generator

#### 11. **reporter** - وكيل إنشاء التقارير
- **الملف**: `backend/pkg/templates/prompts/reporter.tmpl` و `task_reporter.tmpl`
- **الدور**: إنشاء التقارير النهائية للمهام
- **الأدوات المتاحة**:
  - `{{.ReportResultToolName}}`: إنشاء التقارير
  - `{{.SummarizationToolName}}`: التلخيص

#### 12. **reflector** - وكيل المراجعة والتصحيح
- **الملف**: `backend/pkg/templates/prompts/reflector.tmpl`
- **الدور**: مراجعة النتائج واكتشاف الأخطاء
- **النماذج المستخدمة**:
  - OpenAI: `o4-mini` (medium reasoning)
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

#### 13. **enricher** - وكيل إثراء المعلومات
- **الملف**: `backend/pkg/templates/prompts/enricher.tmpl`
- **الدور**: إثراء وتوسيع المعلومات
- **النماذج المستخدمة**:
  - OpenAI: `gpt-4.1-mini`
  - Anthropic: `claude-haiku-4-5`
  - Gemini: `gemini-2.0-flash`

#### 14. **toolcall_fixer** - وكيل إصلاح استدعاءات الأدوات
- **الملف**: `backend/pkg/templates/prompts/toolcall_fixer.tmpl`
- **الدور**: إصلاح استدعاءات الأدوات المعطوبة

#### 15. **summarizer** - وكيل التلخيص
- **الملف**: `backend/pkg/templates/prompts/summarizer.tmpl`
- **الدور**: تلخيص المحتوى الطويل

---

## 5️⃣ قوالب البرومبتات (Prompt Templates)

### قائمة كاملة بملفات القوالب

الموقع: `backend/pkg/templates/prompts/`

#### قوالب الوكلاء الرئيسية:
1. `primary_agent.tmpl` - الوكيل المنسق الرئيسي
2. `assistant.tmpl` - المساعد التفاعلي
3. `pentester.tmpl` - وكيل اختبار الاختراق
4. `coder.tmpl` - وكيل البرمجة
5. `installer.tmpl` - وكيل التثبيت
6. `searcher.tmpl` - وكيل البحث
7. `memorist.tmpl` - وكيل إدارة الذاكرة
8. `adviser.tmpl` - وكيل تقديم النصائح
9. `generator.tmpl` - وكيل توليد المهام
10. `refiner.tmpl` - وكيل تحسين الخطط
11. `reporter.tmpl` - وكيل إنشاء التقارير
12. `reflector.tmpl` - وكيل المراجعة
13. `enricher.tmpl` - وكيل إثراء المعلومات

#### قوالب الأسئلة للوكلاء:
14. `question_pentester.tmpl` - أسئلة اختبار الاختراق
15. `question_coder.tmpl` - أسئلة البرمجة
16. `question_installer.tmpl` - أسئلة التثبيت
17. `question_searcher.tmpl` - أسئلة البحث
18. `question_memorist.tmpl` - أسئلة الذاكرة
19. `question_adviser.tmpl` - أسئلة المشورة
20. `question_enricher.tmpl` - أسئلة الإثراء
21. `question_reflector.tmpl` - أسئلة المراجعة

#### قوالب المهام:
22. `subtasks_generator.tmpl` - توليد المهام الفرعية
23. `subtasks_refiner.tmpl` - تحسين المهام الفرعية
24. `task_reporter.tmpl` - تقرير المهمة
25. `task_descriptor.tmpl` - وصف المهمة

#### قوالب السياق:
26. `full_execution_context.tmpl` - سياق التنفيذ الكامل
27. `short_execution_context.tmpl` - سياق التنفيذ المختصر
28. `execution_logs.tmpl` - سجلات التنفيذ

#### قوالب الأدوات المساعدة:
29. `toolcall_fixer.tmpl` - إصلاح استدعاءات الأدوات
30. `input_toolcall_fixer.tmpl` - إصلاح مدخلات الأدوات
31. `summarizer.tmpl` - التلخيص
32. `image_chooser.tmpl` - اختيار الصورة
33. `language_chooser.tmpl` - اختيار اللغة
34. `flow_descriptor.tmpl` - وصف التدفق

---

## 6️⃣ بنية System Prompts

### مكونات System Prompt

كل وكيل له system prompt يتكون من:

1. **هوية الوكيل (Agent Identity)**
   - الدور والمسؤوليات
   - القدرات الأساسية

2. **قواعد استخدام الأدوات (Tool Usage Rules)**
   ```
   - ALL actions MUST use structured tool calls
   - VERIFY tool call success/failure
   - AVOID redundant actions
   - PRIORITIZE minimally invasive tools
   ```

3. **بروتوكول الذاكرة (Memory Protocol)**
   ```
   - ALWAYS retrieve from memory FIRST
   - Leverage previous solutions
   - Store valuable discoveries
   ```

4. **التعاون الجماعي (Team Collaboration)**
   - قائمة الوكلاء المتخصصين
   - متى يتم التفويض لكل وكيل
   - الأدوات المتاحة لكل وكيل

5. **دمج Graphiti (إن كان مفعلاً)**
   - البحث في السياق التاريخي
   - استخدام المعرفة المؤسسية

### مثال: System Prompt للوكيل الرئيسي

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

## 7️⃣ آلية التفويض (Delegation Mechanism)

### كيفية عمل التفويض

1. **تحليل المهمة**: الوكيل الرئيسي يحلل المهمة
2. **اختيار الوكيل المناسب**: بناءً على نوع المهمة
3. **إعداد السياق**: تجهيز `ExecutionContext` للوكيل
4. **استدعاء الوكيل**: عبر Tool Calling
5. **معالجة النتيجة**: استلام ومعالجة نتيجة الوكيل
6. **الاستمرار أو الإنهاء**: حسب حالة المهمة

### سياق التنفيذ (Execution Context)

من `backend/pkg/providers/provider.go`:

```go
type FlowProvider interface {
    PrepareAgentChain(ctx context.Context, taskID, subtaskID int64) (int64, error)
    PerformAgentChain(ctx context.Context, taskID, subtaskID, msgChainID int64) (PerformResult, error)
    PutInputToAgentChain(ctx context.Context, msgChainID int64, input string) error
    EnsureChainConsistency(ctx context.Context, msgChainID int64) error
}
```

### نتائج التنفيذ (Perform Results)

```go
const (
    PerformResultError   PerformResult = iota // خطأ
    PerformResultWaiting                      // في انتظار مدخلات
    PerformResultDone                         // تم الإنجاز
)
```

---

## 8️⃣ دعم Streaming

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

يسمح بمعالجة الردود بشكل تدريجي أثناء توليدها.

---

## 9️⃣ دعم Embeddings

جميع المزودين يدعمون توليد التضمينات (Embeddings) عبر:

```go
type Embedder interface {
    EmbedDocuments(ctx context.Context, texts []string) ([][]float64, error)
    EmbedQuery(ctx context.Context, text string) ([]float64, error)
}
```

الموقع: `backend/pkg/providers/embeddings/embedder.go`

---

## 🔟 التكامل مع Observability

### Langfuse Integration

جميع الوكلاء مدمجة مع Langfuse للمراقبة:

```go
observation.Span(
    langfuse.WithStartSpanName("primary agent"),
    langfuse.WithStartSpanInput(chain),
    langfuse.WithStartSpanMetadata(metadata),
)
```

### OpenTelemetry Support

دعم كامل لـ OpenTelemetry Tracing:

```go
ctx, span := obs.Observer.NewSpan(ctx, obs.SpanKindInternal, "agent.execute")
defer span.End()
```

---

## 1️⃣1️⃣ حدود الرسائل (Message Limits)

```go
const (
    msgGeneratorSizeLimit = 150 * 1024 // 150 KB
    msgRefinerSizeLimit   = 100 * 1024 // 100 KB
    msgReporterSizeLimit  = 100 * 1024 // 100 KB
    msgSummarizerLimit    = 16 * 1024  // 16 KB
)
```

عند تجاوز هذه الحدود، يتم تلخيص المحتوى تلقائياً.

---

## 1️⃣2️⃣ دعم Graphiti (Knowledge Graph)

### ما هو Graphiti؟

Graphiti هو نظام رسم بياني معرفي زمني يخزن:
- جميع ردود الوكلاء السابقة
- سجلات تنفيذ الأدوات
- العلاقات بين الكيانات
- السياق التاريخي

### أنواع البحث في Graphiti

1. **recent_context** - السياق الأخير
2. **successful_tools** - الأدوات الناجحة
3. **episode_context** - سياق الحلقة الكاملة
4. **diverse_results** - نتائج متنوعة

---

## 1️⃣3️⃣ أدوات النظام (System Tools)

### الأدوات المتاحة للوكلاء:

1. **FinalyToolName** - إنهاء المهمة
2. **AskUserToolName** - سؤال المستخدم
3. **SearchToolName** - البحث
4. **PentesterToolName** - اختبار الاختراق
5. **CoderToolName** - البرمجة
6. **InstallerToolName** - التثبيت
7. **AdviceToolName** - المشورة
8. **MemoristToolName** - الذاكرة
9. **MaintenanceToolName** - الصيانة
10. **SummarizationToolName** - التلخيص
11. **SubtaskListToolName** - إدارة المهام الفرعية
12. **SubtaskPatchToolName** - تعديل المهام
13. **ReportResultToolName** - إنشاء التقارير
14. **TerminalToolName** - الطرفية
15. **FileToolName** - الملفات
16. **BrowserToolName** - المتصفح
17. **SearchCodeToolName** - البحث عن الأكواد
18. **StoreCodeToolName** - حفظ الأكواد
19. **SearchGuideToolName** - البحث عن الأدلة
20. **StoreGuideToolName** - حفظ الأدلة

---

## 1️⃣4️⃣ استراتيجية اختيار النماذج

### المبادئ العامة:

1. **المهام الرئيسية والمعقدة**:
   - OpenAI: `gpt-5`, `gpt-5.2`, `o4-mini`
   - Anthropic: `claude-sonnet-4-5`, `claude-opus-4-5`
   - Gemini: `gemini-2.5-pro`

2. **المهام السريعة والبسيطة**:
   - OpenAI: `gpt-4.1-mini`
   - Anthropic: `claude-haiku-4-5`
   - Gemini: `gemini-2.0-flash`

3. **مهام التفكير العميق**:
   - OpenAI: `o1`, `o3`, `o4-mini`
   - Anthropic: `claude-opus-4-5`
   - Gemini: `gemini-2.5-pro` (high reasoning)

4. **مهام البرمجة**:
   - OpenAI: `gpt-5.2`
   - Anthropic: `claude-sonnet-4-5`
   - Gemini: `gemini-2.5-pro`

---

## 1️⃣5️⃣ معلومات التسعير (Pricing)

### مقارنة التكلفة (لكل مليون توكن)

#### النماذج الأغلى:
- OpenAI `o1`: $15 / $60
- Anthropic `claude-opus-4-5`: $5 / $25

#### النماذج المتوسطة:
- OpenAI `gpt-5`: $1.25 / $10
- OpenAI `gpt-5.2`: $1.75 / $14
- Anthropic `claude-sonnet-4-5`: $3 / $15
- Gemini `gemini-2.5-pro`: $1.25 / $10

#### النماذج الاقتصادية:
- OpenAI `gpt-4.1-mini`: $0.4 / $1.6
- Anthropic `claude-haiku-4-5`: $1 / $5
- Gemini `gemini-2.0-flash`: $0.1 / $0.4

---

## 📚 ملخص

### الإحصائيات الرئيسية:

- **عدد المزودين**: 6 (OpenAI, Anthropic, Gemini, Bedrock, Ollama, Custom)
- **عدد الوكلاء**: 15 وكيل متخصص
- **عدد قوالب البرومبتات**: 34 قالب
- **عدد الأدوات**: 20+ أداة متاحة
- **دعم Streaming**: ✅ نعم
- **دعم Tool Calling**: ✅ نعم
- **دعم JSON Mode**: ✅ نعم
- **دعم Vision**: ✅ نعم (OpenAI, Anthropic, Gemini)
- **دعم Reasoning**: ✅ نعم (OpenAI o-series, Anthropic Claude 4, Gemini 2.5)
- **دعم Embeddings**: ✅ نعم
- **دعم Observability**: ✅ نعم (Langfuse, OpenTelemetry)

---

## 🔗 الملفات المرجعية

### الملفات الرئيسية:
- `backend/pkg/providers/provider.go` - الواجهة الأساسية
- `backend/pkg/providers/pconfig/config.go` - نظام التكوين
- `backend/pkg/providers/providers.go` - مدير المزودين
- `backend/pkg/providers/performer.go` - منطق تنفيذ الوكلاء
- `backend/pkg/templates/templates.go` - نظام القوالب

### ملفات المزودين:
- `backend/pkg/providers/openai/openai.go`
- `backend/pkg/providers/anthropic/anthropic.go`
- `backend/pkg/providers/gemini/gemini.go`
- `backend/pkg/providers/bedrock/bedrock.go`
- `backend/pkg/providers/ollama/ollama.go`
- `backend/pkg/providers/custom/custom.go`

---

## 📝 ملاحظات ختامية

هذا التحليل يغطي الجوانب الأساسية لبنية نماذج اللغة الكبيرة في مشروع PentAGI. النظام مصمم بشكل معياري ومرن للغاية، مع دعم متعدد المزودين ونظام وكلاء متطور يستخدم أحدث تقنيات الذكاء الاصطناعي.

**تاريخ التحليل**: يناير 2026
**إصدار المشروع**: PentAGI
**المصدر**: https://github.com/raglox/pentagi

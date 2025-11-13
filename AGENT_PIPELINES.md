# Ollama Agent Pipelines Documentation

Этот документ описывает все пайплайны и схемы работы агентов в Ollama. Каждый пайплайн описывает полный путь обработки запроса от пользовательского ввода до генерации ответа модели.

---

## Содержание

1. [Базовый Chat Pipeline](#1-базовый-chat-pipeline)
2. [Tool Calling Pipeline](#2-tool-calling-pipeline)
3. [Multimodal Pipeline](#3-multimodal-pipeline)
4. [Thinking Mode Pipeline](#4-thinking-mode-pipeline)
5. [Streaming Pipeline](#5-streaming-pipeline)
6. [Template Rendering Pipeline](#6-template-rendering-pipeline)
7. [Context Window Management](#7-context-window-management)
8. [Message Truncation Pipeline](#8-message-truncation-pipeline)

---

## 1. Базовый Chat Pipeline

**Назначение:** Обработка простого текстового чата без инструментов, изображений или thinking mode.

**Точка входа:** `server/routes.go:ChatHandler()` (строки 1844-2335)

### 1.1 Общая схема потока

```
┌─────────────────────────────────────────────────────────────────┐
│                    БАЗОВЫЙ CHAT PIPELINE                         │
└─────────────────────────────────────────────────────────────────┘

   [User Request]
        │
        ├─ POST /api/chat
        │  {
        │    "model": "llama3",
        │    "messages": [
        │      {"role": "system", "content": "..."},
        │      {"role": "user", "content": "Hello"}
        │    ],
        │    "stream": true
        │  }
        │
        ▼
   ┌─────────────────────┐
   │  1. ChatHandler     │  server/routes.go:1844
   │  - Validate request │
   │  - Load model       │
   │  - Parse options    │
   └──────────┬──────────┘
              │
              ▼
   ┌─────────────────────────────┐
   │  2. GetModel()              │  server/routes.go:1900
   │  - Load model from registry │
   │  - Check if loaded          │
   │  - Return *Model            │
   └──────────┬──────────────────┘
              │
              ▼
   ┌─────────────────────────────────────┐
   │  3. chatPrompt()                    │  server/prompt.go:23
   │  - Truncate messages to fit context │
   │  - Keep system messages             │
   │  - Keep last message                │
   │  - Calculate tokens                 │
   └──────────┬──────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │  4. renderPrompt()                   │  server/prompt.go:107
   │  ┌──────────────┬──────────────────┐ │
   │  │ Has Renderer?│                  │ │
   │  └───────┬──────┴──────────────────┘ │
   │          │                            │
   │    YES   │   NO                       │
   │    ┌─────▼────────┐  ┌─────────────┐ │
   │    │ Custom       │  │ Template    │ │
   │    │ Renderer     │  │ Execution   │ │
   │    │ (Qwen3VL)    │  │ (.gotmpl)   │ │
   │    └──────────────┘  └─────────────┘ │
   └──────────┬───────────────────────────┘
              │
              ▼
   ┌──────────────────────────────┐
   │  5. Template.Execute()       │  template/template.go:254
   │  - Iterate over messages     │
   │  - Apply template format     │
   │  - Insert system prompts     │
   │  - Format roles              │
   │  Output: formatted prompt    │
   └──────────┬───────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │  6. llm.Completion()                 │  llm/server.go:1464
   │  - Send to LLM backend               │
   │  - Configure sampling params         │
   │  - Setup streaming or batch          │
   └──────────┬───────────────────────────┘
              │
              ▼
   ┌─────────────────────────────────────┐
   │  7. Stream Response                 │
   │  ┌──────────────────────────────┐   │
   │  │ While generating:            │   │
   │  │ - Read chunk from LLM        │   │
   │  │ - Write to response stream   │   │
   │  │ - Format as NDJSON           │   │
   │  │ - Send to client             │   │
   │  └──────────────────────────────┘   │
   └──────────┬──────────────────────────┘
              │
              ▼
   [Client receives response]
   {
     "model": "llama3",
     "created_at": "...",
     "message": {
       "role": "assistant",
       "content": "Hello! How can I help you?"
     },
     "done": true
   }
```

### 1.2 Детальное описание этапов

#### Этап 1: ChatHandler - Валидация и инициализация

**Расположение:** `server/routes.go:1844-1900`

**Входные данные:**
```go
type ChatRequest struct {
    Model    string        // Имя модели
    Messages []api.Message // История сообщений
    Stream   bool          // Потоковый режим
    Options  map[string]any // Опции генерации
}
```

**Процесс:**
1. Декодирование JSON запроса
2. Валидация наличия модели и сообщений
3. Проверка формата сообщений (role, content)
4. Установка значений по умолчанию

**Выходные данные:** Валидированный `ChatRequest`

---

#### Этап 2: GetModel - Загрузка модели

**Расположение:** `server/routes.go:1900-1950`

**Процесс:**
1. Поиск модели в реестре
2. Если модель не загружена → загрузка с диска
3. Чтение конфигурации модели (Modelfile)
4. Инициализация параметров модели
5. Загрузка весов в память (если нужно)

**Ключевые поля Model:**
```go
type Model struct {
    Config          ModelConfig
    Template        *template.Template
    ProjectorPaths  []string  // Для мультимодальных моделей
    AdapterPaths    []string  // Для LoRA адаптеров
}
```

---

#### Этап 3: chatPrompt - Подготовка сообщений

**Расположение:** `server/prompt.go:23-105`

**Алгоритм:**
```
1. n = индекс последнего сообщения
2. FOR i FROM n DOWN TO 0:
     a. Если i == n, пропустить (всегда включаем последнее)
     b. Собрать все system сообщения до i
     c. Склеить system + messages[i:]
     d. Отрендерить промт
     e. Подсчитать токены
     f. Если tokens > context_window:
           BREAK (нашли точку обрезания)
     g. ELSE:
           n = i (можно включить ещё одно сообщение)
3. RETURN messages[n:]
```

**Принципы:**
- **Всегда включаем:** последнее сообщение пользователя
- **Всегда сохраняем:** все system сообщения
- **Обрезаем:** старые user/assistant сообщения

**Пример:**
```
Входящие сообщения:
  [0] system: "You are helpful"
  [1] user: "What's 2+2?"
  [2] assistant: "4"
  [3] user: "What's 5+5?"
  [4] assistant: "10"
  [5] user: "What's 10+10?"

Context window: 500 tokens

После обрезания:
  [0] system: "You are helpful"  ← сохранено
  [3] user: "What's 5+5?"        ← поместилось
  [4] assistant: "10"
  [5] user: "What's 10+10?"      ← всегда включено
```

---

#### Этап 4: renderPrompt - Маршрутизация рендеринга

**Расположение:** `server/prompt.go:107-127`

**Решение:**
```
IF model.Config.Renderer != "":
    USE renderers.RenderWithRenderer()
    ↓
    Custom renderer (например Qwen3VL)
ELSE:
    USE template.Execute()
    ↓
    Go text/template система
```

**Custom Renderers:**
- `qwen3-coder` → XML tool format
- `qwen3-vl-instruct` → Vision tokens
- `qwen3-vl-thinking` → Thinking tags

---

#### Этап 5: Template.Execute - Форматирование промпта

**Расположение:** `template/template.go:254-347`

**Процесс:**
```
1. Создать Values структуру:
   {
     Messages: [...],
     Tools: [...],
     Prompt: "...",
     System: "..."
   }

2. collate() - Объединить смежные сообщения:
   [user, user] → [user (объединённый)]

3. Execute template:
   FOR each message in Messages:
       IF role == "system":
           output += "<|im_start|>system\n" + content + "<|im_end|>\n"
       ELIF role == "user":
           output += "<|im_start|>user\n" + content + "<|im_end|>\n"
       ELIF role == "assistant":
           output += "<|im_start|>assistant\n" + content + "<|im_end|>\n"

4. Добавить приглашение к генерации:
   output += "<|im_start|>assistant\n"

5. RETURN formatted prompt
```

**Пример вывода (ChatML):**
```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
```

---

#### Этап 6: llm.Completion - Генерация

**Расположение:** `llm/server.go:1464-1625`

**Процесс:**
1. Создать CompletionRequest:
   ```go
   {
       Prompt: formatted_prompt,
       Options: {
           Temperature: 0.7,
           TopP: 0.9,
           TopK: 40,
           NumPredict: -1,
       }
   }
   ```

2. Отправить запрос в LLM backend (llama.cpp)

3. Если streaming:
   ```
   WHILE not done:
       chunk = read_from_backend()
       YIELD chunk
   ```

4. Если batch:
   ```
   WAIT for full response
   RETURN complete text
   ```

---

#### Этап 7: Stream Response - Отправка клиенту

**Формат NDJSON:**
```json
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":"H"},"done":false}
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":"e"},"done":false}
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":"l"},"done":false}
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":"l"},"done":false}
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":"o"},"done":false}
...
{"model":"llama3","created_at":"...","message":{"role":"assistant","content":""},"done":true,"total_duration":123,"load_duration":45}
```

**Структура последнего сообщения:**
- `done: true`
- `total_duration` - общее время
- `load_duration` - время загрузки модели
- `prompt_eval_count` - количество токенов в промпте
- `eval_count` - количество сгенерированных токенов

---

### 1.3 Точки принятия решений

```
Decision Tree:

1. Streaming enabled?
   ├─ YES → Setup chunked response writer
   └─ NO  → Accumulate full response

2. Has custom renderer?
   ├─ YES → Use renderers.RenderWithRenderer()
   └─ NO  → Use template.Execute()

3. Context overflow?
   ├─ YES → Truncate older messages
   └─ NO  → Use all messages

4. System messages present?
   ├─ YES → Always include in context
   └─ NO  → Continue without system

5. Model loaded in memory?
   ├─ YES → Reuse loaded model
   └─ NO  → Load from disk
```

---

### 1.4 Промты в pipeline

**Системный промт:** Устанавливается в первом сообщении или через Modelfile

**Пример flow промптов:**
1. User отправляет: `"Hello"`
2. System промт добавляется: `"You are a helpful assistant"`
3. Template рендерит:
   ```
   <|im_start|>system
   You are a helpful assistant<|im_end|>
   <|im_start|>user
   Hello<|im_end|>
   <|im_start|>assistant
   ```
4. LLM генерирует ответ начиная с позиции после `<|im_start|>assistant`

---

### 1.5 Данные передаваемые между этапами

```
ChatHandler → GetModel:
  - model_name: string

GetModel → chatPrompt:
  - *Model (full model struct)
  - []api.Message
  - Options

chatPrompt → renderPrompt:
  - *Model
  - []api.Message (truncated)
  - []api.Tool (empty for basic chat)

renderPrompt → Template.Execute:
  - []api.Message
  - template string
  - Values struct

Template.Execute → llm.Completion:
  - formatted_prompt: string

llm.Completion → Client:
  - []byte chunks (streaming)
  OR
  - complete response (batch)
```

---

## 2. Tool Calling Pipeline

**Назначение:** Обработка запросов с вызовом инструментов (functions/tools). Модель может вызывать функции для получения дополнительной информации.

**Точка входа:** `server/routes.go:ChatHandler()` с `tools` в запросе

### 2.1 Общая схема Tool Calling

```
┌─────────────────────────────────────────────────────────────────┐
│                   TOOL CALLING PIPELINE                          │
└─────────────────────────────────────────────────────────────────┘

   [User Request with Tools]
        │
        ├─ POST /api/chat
        │  {
        │    "model": "qwen3-coder",
        │    "messages": [
        │      {"role": "user", "content": "What's the weather?"}
        │    ],
        │    "tools": [
        │      {
        │        "type": "function",
        │        "function": {
        │          "name": "get_weather",
        │          "description": "Get weather for location",
        │          "parameters": {...}
        │        }
        │      }
        │    ]
        │  }
        │
        ▼
   ┌─────────────────────────────────┐
   │  1. ChatHandler                 │
   │  - Validate tools               │
   │  - Check model capabilities     │
   │  - Prepare tool definitions     │
   └──────────┬──────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────────┐
   │  2. renderPrompt() with Tools            │
   │  ┌────────────────────────────────────┐  │
   │  │ Route to model-specific renderer:  │  │
   │  │                                    │  │
   │  │ qwen3-coder  → XML format          │  │
   │  │ qwen3-vl     → JSON in XML         │  │
   │  │ command-r    → Python format       │  │
   │  └────────────────────────────────────┘  │
   └──────────┬───────────────────────────────┘
              │
              ▼
   ┌─────────────────────────────────────────────┐
   │  3. Inject Tool Definitions into System     │
   │  ┌───────────────────────────────────────┐  │
   │  │ Qwen Format:                          │  │
   │  │                                       │  │
   │  │ # Tools                               │  │
   │  │ You have access to:                   │  │
   │  │ <tools>                               │  │
   │  │   <function>                          │  │
   │  │     <name>get_weather</name>          │  │
   │  │     <description>...</description>    │  │
   │  │     <parameters>...</parameters>      │  │
   │  │   </function>                         │  │
   │  │ </tools>                              │  │
   │  │                                       │  │
   │  │ Format: <tool_call>...</tool_call>    │  │
   │  └───────────────────────────────────────┘  │
   └──────────┬──────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────┐
   │  4. LLM Generates Response       │
   │                                  │
   │  Output может содержать:         │
   │  a) Tool call                    │
   │  b) Text response                │
   │  c) Both                         │
   └──────────┬─────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │  5. Parse Response                   │
   │  ┌────────────────────────────────┐  │
   │  │ Stream chunks through parsers: │  │
   │  │                                │  │
   │  │ Chunk → BuiltinParser          │  │
   │  │         (для встроенных tools) │  │
   │  │      ↓                          │  │
   │  │      GenericToolParser          │  │
   │  │         (для JSON/XML)          │  │
   │  └────────────────────────────────┘  │
   └──────────┬───────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────────┐
   │  6. Extract Tool Calls                   │
   │  ┌────────────────────────────────────┐  │
   │  │ Example extraction:                │  │
   │  │                                    │  │
   │  │ <tool_call>                        │  │
   │  │ <function=get_weather>             │  │
   │  │ <parameter=location>               │  │
   │  │ San Francisco                      │  │
   │  │ </parameter>                       │  │
   │  │ </function>                        │  │
   │  │ </tool_call>                       │  │
   │  │                                    │  │
   │  │ Parsed to:                         │  │
   │  │ {                                  │  │
   │  │   "name": "get_weather",           │  │
   │  │   "arguments": {                   │  │
   │  │     "location": "San Francisco"    │  │
   │  │   }                                │  │
   │  │ }                                  │  │
   │  └────────────────────────────────────┘  │
   └──────────┬───────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────┐
   │  7. Tool Call Decision           │
   │  ┌────────────────────────────┐  │
   │  │ IF tool calls found:       │  │
   │  │   - Send to client         │  │
   │  │   - Wait for tool results  │  │
   │  │ ELSE:                      │  │
   │  │   - Send text response     │  │
   │  └────────────────────────────┘  │
   └──────────┬─────────────────────────┘
              │
       ┌──────┴────────┐
       │               │
    Tool Call      Text Response
       │               │
       ▼               ▼
   [Client]        [Client]
   Receives:       Receives:
   {               {
     "message": {    "message": {
       "role": "..     "role": "ass..",
       "tool_calls": [ "content": "..."
         {             },
           "function": {"done": true
             "name": ".."
             "argumen.."
           }
         }
       ]
     }
   }

   │
   ▼
   [Client Executes Tool]
   result = get_weather("San Francisco")

   │
   ▼
   [Client Sends Tool Result]
   POST /api/chat
   {
     "messages": [
       {"role": "user", "content": "What's the weather?"},
       {"role": "assistant", "tool_calls": [...]},
       {"role": "tool", "content": "Temperature: 72°F, Sunny"}
     ]
   }

   │
   ▼
   ┌────────────────────────────────────┐
   │  8. Second LLM Call                │
   │  - Include tool result in context  │
   │  - Generate final answer           │
   └────────────┬───────────────────────┘
                │
                ▼
   [Client receives final response]
   {
     "message": {
       "role": "assistant",
       "content": "The weather in San Francisco is 72°F and sunny."
     },
     "done": true
   }
```

### 2.2 Детальное описание этапов

#### Этап 1: Валидация инструментов

**Проверки:**
1. Формат tool definition:
   ```go
   type Tool struct {
       Type     string   // "function"
       Function Function
   }

   type Function struct {
       Name        string
       Description string
       Parameters  Parameters
   }
   ```

2. Capabilities модели:
   ```go
   if !model.SupportsTools() {
       return error("model doesn't support tools")
   }
   ```

---

#### Этап 2-3: Рендеринг с инструментами

**Qwen3-Coder формат:**

**Расположение:** `model/renderers/qwen3coder.go:77-136`

```xml
<|im_start|>system
You are Qwen, a helpful AI assistant that can interact with a computer to solve tasks.

# Tools

You have access to the following functions:

<tools>
<function>
<name>get_weather</name>
<description>Get current weather for a location</description>
<parameters>
<parameter>
<name>location</name>
<type>string</type>
<description>City name</description>
</parameter>
<parameter>
<name>unit</name>
<type>string</type>
<description>Temperature unit (celsius or fahrenheit)</description>
</parameter>
</parameters>
</function>
</tools>

If you choose to call a function ONLY reply in the following format with NO suffix:

<tool_call>
<function=example_function_name>
<parameter=example_parameter_1>
value_1
</parameter>
<parameter=example_parameter_2>
This is the value for the second parameter
that can span
multiple lines
</parameter>
</function>
</tool_call>

<IMPORTANT>
Reminder:
- Function calls MUST follow the specified format
- Required parameters MUST be specified
- You may provide optional reasoning BEFORE the function call, but NOT after
</IMPORTANT>
<|im_end|>
<|im_start|>user
What's the weather in San Francisco?<|im_end|>
<|im_start|>assistant
```

**Command-R формат:**

**Расположение:** `template/command-r.gotmpl:13-48`

```python
## Available Tools
Here is a list of tools that you have available to you:

```python
def get_weather(location: str, unit: str) -> List[Dict]:
    '''Get current weather for a location

    Args:
        location (str): City name
        unit (str): Temperature unit (celsius or fahrenheit)
    '''
    pass
```

Write 'Action:' followed by a json-formatted list of actions...
```json
[
    {
        "tool_name": "get_weather",
        "parameters": {
            "location": "San Francisco",
            "unit": "fahrenheit"
        }
    }
]```
```

---

#### Этап 4-5: Парсинг tool calls

**Parser Chain:**

**Расположение:** `tools/tools.go`

```go
// Builtin Parser - для встроенных инструментов
type BuiltinParser struct {
    ToolNames map[string]bool
}

// Generic Parser - для пользовательских инструментов
type GenericToolParser struct {
    State      parserState
    Buffer     strings.Builder
    ToolCalls  []ToolCall
}
```

**State Machine:**
```
States:
  - Outside: ищем начало tool call
  - InToolCall: внутри <tool_call>
  - InFunction: внутри <function=name>
  - InParameter: внутри <parameter=name>
  - InJSON: парсим JSON (для некоторых форматов)

Transitions:
  Outside → InToolCall:  видим "<tool_call>"
  InToolCall → InFunction:  видим "<function="
  InFunction → InParameter:  видим "<parameter="
  InParameter → InFunction:  видим "</parameter>"
  InFunction → InToolCall:  видим "</function>"
  InToolCall → Outside:  видим "</tool_call>"
```

**Пример парсинга:**

Input stream:
```
I'll check the weather for you.

<tool_call>
<function=get_weather>
<parameter=location>
San Francisco
</parameter>
<parameter=unit>
fahrenheit
</parameter>
</function>
</tool_call>

Let me get that information.
```

Parsed output:
```go
ToolCalls: [
  {
    Function: {
      Name: "get_weather",
      Arguments: {
        "location": "San Francisco",
        "unit": "fahrenheit"
      }
    }
  }
]

RemainingText: "I'll check the weather for you.\n\nLet me get that information."
```

---

#### Этап 6-7: Отправка результатов клиенту

**Response format:**

```json
{
  "model": "qwen3-coder",
  "created_at": "2024-01-15T10:30:00Z",
  "message": {
    "role": "assistant",
    "content": "I'll check the weather for you.",
    "tool_calls": [
      {
        "function": {
          "name": "get_weather",
          "arguments": {
            "location": "San Francisco",
            "unit": "fahrenheit"
          }
        }
      }
    ]
  },
  "done": true
}
```

---

#### Этап 8: Второй раунд с результатами

**Client sends tool results:**

```json
{
  "model": "qwen3-coder",
  "messages": [
    {"role": "user", "content": "What's the weather in San Francisco?"},
    {
      "role": "assistant",
      "tool_calls": [
        {
          "function": {
            "name": "get_weather",
            "arguments": {
              "location": "San Francisco",
              "unit": "fahrenheit"
            }
          }
        }
      ]
    },
    {
      "role": "tool",
      "content": "{\"temperature\": 72, \"condition\": \"sunny\", \"humidity\": 45}"
    }
  ]
}
```

**Prompt для второго раунда:**

```
<|im_start|>system
[system prompt with tools]<|im_end|>
<|im_start|>user
What's the weather in San Francisco?<|im_end|>
<|im_start|>assistant
<tool_call>
<function=get_weather>
<parameter=location>San Francisco</parameter>
<parameter=unit>fahrenheit</parameter>
</function>
</tool_call><|im_end|>
<|im_start|>tool
{"temperature": 72, "condition": "sunny", "humidity": 45}<|im_end|>
<|im_start|>assistant
```

**Final response:**

```
The weather in San Francisco is currently 72°F and sunny, with 45% humidity.
```

---

### 2.3 Поток данных для tool calls

```
User Request
    ↓
[Tools Definitions] + [Messages]
    ↓
Renderer adds tool prompts to system
    ↓
Formatted Prompt with tool instructions
    ↓
LLM generates response (possibly with tool calls)
    ↓
Parser extracts tool calls from stream
    ↓
IF tool calls found:
    Send tool_calls to client
    ↓
    Client executes tools
    ↓
    Client sends results back
    ↓
    New request with tool results in messages
    ↓
    LLM generates final answer
ELSE:
    Send text response to client
```

---

### 2.4 Промты для разных моделей

**Qwen3-Coder:** XML-based, подробные инструкции
**Qwen3-VL:** JSON в XML тегах, короче
**Command-R:** Python function signatures, JSON actions
**Generic:** Зависит от шаблона модели

**Критические элементы в промпте:**
1. Список доступных функций
2. Формат вызова функций
3. Формат параметров
4. Правила использования (когда вызывать, когда нет)

---

## 3. Multimodal Pipeline

**Назначение:** Обработка запросов с изображениями (vision models).

**Точка входа:** `server/routes.go:ChatHandler()` с images в messages

### 3.1 Схема Multimodal Pipeline

```
   [User Request with Images]
        │
        ├─ messages: [{
        │    "role": "user",
        │    "content": "What's in this image?",
        │    "images": [base64_encoded_image]
        │  }]
        │
        ▼
   ┌────────────────────────────────────┐
   │  1. chatPrompt()                   │  server/prompt.go:72-96
   │  - Process each message            │
   │  - For each image:                 │
   │    • Create llm.ImageData with ID  │
   │    • Generate tag [img-N]          │
   │    • Replace [img] placeholder OR  │
   │    • Prepend tag to content        │
   │  - Count 768 tokens per image      │
   └────────────┬───────────────────────┘
                │
                ▼
   ┌────────────────────────────────────┐
   │  2. Validate Image Count           │
   │  - mllama: max 1 image             │
   │  - others: unlimited               │
   └────────────┬───────────────────────┘
                │
                ▼
   ┌────────────────────────────────────────────┐
   │  3. renderPrompt() - Vision Renderer       │
   │  ┌──────────────────────────────────────┐  │
   │  │ Qwen3-VL:                            │  │
   │  │  IF useImgTags:                      │  │
   │  │    Insert "[img]" for each image     │  │
   │  │  ELSE:                               │  │
   │  │    Insert "<|vision_start|>          │  │
   │  │           <|image_pad|>              │  │
   │  │           <|vision_end|>"            │  │
   │  └──────────────────────────────────────┘  │
   └────────────┬───────────────────────────────┘
                │
                ▼
   ┌────────────────────────────────────┐
   │  4. Template with Image Tags       │
   │                                    │
   │  <|im_start|>user                  │
   │  <|vision_start|><|image_pad|>     │
   │  <|vision_end|>                    │
   │  What's in this image?<|im_end|>   │
   │  <|im_start|>assistant             │
   └────────────┬───────────────────────┘
                │
                ▼
   ┌────────────────────────────────────┐
   │  5. llm.Completion()               │
   │  - Send prompt + image data        │
   │  - LLM processes vision inputs     │
   │  - Generate description            │
   └────────────┬───────────────────────┘
                │
                ▼
   [Response: "The image shows..."]
```

**Ключевые моменты:**
- Изображения = 768 токенов каждое
- Всегда в начале сообщения
- Модель-специфичные токены vision
- Проверка лимитов модели

---

## 4. Thinking Mode Pipeline

**Назначение:** Обработка моделей с режимом размышлений (reasoning).

**Точка входа:** `server/routes.go:ChatHandler()` для thinking-capable models

### 4.1 Схема Thinking Pipeline

```
   [User Request]
        │
        ▼
   ┌────────────────────────────────────┐
   │  1. Check Model Supports Thinking  │
   │  - Qwen3-VL-Thinking: YES          │
   │  - Others: Optional                │
   └────────────┬───────────────────────┘
                │
                ▼
   ┌────────────────────────────────────────┐
   │  2. Template.InferTags()               │  thinking/template.go:57
   │  - Parse template for {{.Thinking}}    │
   │  - Find surrounding text nodes         │
   │  - Extract opening/closing tags        │
   │  Example: <think> / </think>           │
   └────────────┬─────────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │  3. Render Prompt with Think Tags   │
   │                                     │
   │  <|im_start|>assistant              │
   │  <think>                            │
   │  [LLM generates reasoning here]     │
   │  </think>                           │
   │                                     │
   │  [Final answer here]                │
   │  <|im_end|>                         │
   └────────────┬────────────────────────┘
                │
                ▼
   ┌──────────────────────────────────────┐
   │  4. Stream Response Through Parser   │  thinking/parser.go:57
   │  ┌────────────────────────────────┐  │
   │  │ State Machine:                 │  │
   │  │                                │  │
   │  │ LookingForOpening              │  │
   │  │   ↓ see "<think>"              │  │
   │  │ ThinkingStartedEatingWhitespace│  │
   │  │   ↓ skip whitespace            │  │
   │  │ Thinking                       │  │
   │  │   ↓ accumulate content         │  │
   │  │ ThinkingDoneEatingWhitespace   │  │
   │  │   ↓ see "</think>"             │  │
   │  │ ThinkingDone                   │  │
   │  └────────────────────────────────┘  │
   └────────────┬─────────────────────────┘
                │
                ▼
   ┌─────────────────────────────────────┐
   │  5. Separate Thinking from Content  │
   │                                     │
   │  thinking: "Step 1: analyze...      │
   │             Step 2: conclude..."    │
   │                                     │
   │  content: "The answer is X"         │
   └────────────┬────────────────────────┘
                │
                ▼
   [Client receives both thinking + content]
   {
     "message": {
       "role": "assistant",
       "thinking": "[reasoning process]",
       "content": "[final answer]"
     }
   }
```

**Data Flow:**
- Thinking tagged by model during generation
- Parser extracts in real-time
- Separated in API response
- Can be hidden or shown to user

---

## 5. Streaming vs Non-Streaming

### 5.1 Streaming Mode (Default)

```
Client Request: stream=true
    ↓
Setup chunked transfer encoding
    ↓
For each token generated:
    ├─ Format as NDJSON
    ├─ {"model":"...","message":{"content":"H"},"done":false}
    ├─ Write to response stream
    └─ Flush immediately
    ↓
Final chunk:
    {"model":"...","message":{"content":""},"done":true,"total_duration":...}
```

**Benefits:**
- Lower latency to first token
- Real-time user feedback
- Can cancel mid-stream

### 5.2 Non-Streaming Mode

```
Client Request: stream=false
    ↓
Accumulate all tokens
    ↓
Wait for generation complete
    ↓
Send single response:
    {"model":"...","message":{"content":"[full text]"},"done":true}
```

**Benefits:**
- Simpler client code
- Single response object
- Easier error handling

---

## 6. Template Rendering Pipeline

**Расположение:** `template/template.go`

### 6.1 Execution Flow

```
   renderPrompt()
        ↓
   ┌──────────────────────────────┐
   │  1. Create Values{}          │
   │  {                           │
   │    Messages: []Message       │
   │    Tools: []Tool             │
   │    System: string            │
   │    Prompt: string            │
   │  }                           │
   └──────────┬───────────────────┘
              │
              ▼
   ┌──────────────────────────────┐
   │  2. collate()                │  template.go:354
   │  - Merge consecutive msgs    │
   │  - Extract system messages   │
   └──────────┬───────────────────┘
              │
              ▼
   ┌──────────────────────────────┐
   │  3. template.Execute()       │
   │  - Range over Messages       │
   │  - Apply format per role     │
   │  - Insert special tokens     │
   └──────────┬───────────────────┘
              │
              ▼
   [Formatted Prompt String]
```

**Template Functions:**
- `{{.Messages}}` - iterate messages
- `{{.System}}` - system prompt
- `{{.Tools}}` - tool definitions
- `{{.Thinking}}` - thinking mode flag

---

## 7. Context Window Management

**Расположение:** `server/prompt.go:chatPrompt()`

### 7.1 Token Budget Algorithm

```
Available Context = NumCtx (e.g., 2048)
    ↓
Calculate token costs:
    ├─ Each message: tokenize(render(message))
    ├─ Each image: 768 tokens
    └─ System messages: always included
    ↓
Backward Fitting:
    FOR i = len(messages)-1 DOWN TO 0:
        cost = tokens(system + messages[i:])
        IF cost > context:
            BREAK
        FIT messages[i:]
    ↓
Always include:
    - Last user message (messages[n])
    - All system messages
```

**Example:**
```
Context: 500 tokens
Messages:
  [0] system: 50 tokens  ← always
  [1] user: 100 tokens
  [2] asst: 150 tokens
  [3] user: 100 tokens
  [4] asst: 150 tokens
  [5] user: 80 tokens    ← always

Fit check:
  [0]+[5] = 130 ✓
  [0]+[4]+[5] = 280 ✓
  [0]+[3]+[4]+[5] = 380 ✓
  [0]+[2]+[3]+[4]+[5] = 530 ✗ STOP

Final: messages[3:5] + system
```

---

## 8. Summary: Complete Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     COMPLETE FLOW                            │
└─────────────────────────────────────────────────────────────┘

1. Client Request
    ↓
2. ChatHandler
    ├─ Validate
    ├─ Load Model
    └─ Parse Options
    ↓
3. chatPrompt()
    ├─ Truncate to context
    ├─ Handle images (768 tok ea)
    └─ Keep system + last
    ↓
4. renderPrompt()
    ├─ Custom Renderer OR
    └─ Template Execution
    ↓
5. Inject Prompts
    ├─ System prompt
    ├─ Tool definitions
    └─ Format markers
    ↓
6. llm.Completion()
    ├─ Send to backend
    └─ Configure sampling
    ↓
7. Stream Response
    ├─ Parse thinking (if enabled)
    ├─ Parse tool calls (if present)
    └─ Format NDJSON
    ↓
8. Client Receives
    ├─ Text response OR
    ├─ Tool calls (→ execute → return) OR
    └─ Thinking + content
```

---

## Заключение

Все пайплайны в Ollama следуют общим принципам:
1. **Валидация входных данных**
2. **Загрузка и проверка модели**
3. **Подготовка контекста с учетом ограничений**
4. **Рендеринг промпта в нужном формате**
5. **Генерация с потоковой обработкой**
6. **Парсинг и структурирование вывода**
7. **Доставка результата клиенту**

Каждый пайплайн расширяет базовый chat flow дополнительными возможностями, сохраняя общую архитектуру.

---

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

# Ollama AI Prompts Documentation

Этот документ содержит полную документацию всех AI промптов, используемых в приложении Ollama. Промты сгруппированы по тематикам для удобной навигации.

---

## 1. Системные промты (Default System Prompts)

Системные промты определяют базовое поведение AI модели. Они устанавливаются по умолчанию, если пользователь не предоставил свой собственный системный промт.

### 1.1 Общий системный промт (Generic)

**Назначение:** Используется как базовый системный промт для большинства моделей с форматом ChatML.

**Расположение:**
- `template/index.json:71` (ChatML template)
- `template/index.json:30` (ChatML template variant)
- `template/index.json:35` (ChatML template variant)

**Промт:**
```
You are a helpful assistant.
```

---

### 1.2 Qwen системный промт (Qwen AI Assistant)

**Назначение:** Используется для моделей семейства Qwen (особенно Qwen3-Coder), когда предоставлены инструменты (tools), но отсутствует пользовательский системный промт. Этот промт соответствует референсной реализации Qwen и устанавливает контекст для взаимодействия с компьютером.

**Расположение:** `model/renderers/qwen3coder.go:82`

**Промт:**
```
You are Qwen, a helpful AI assistant that can interact with a computer to solve tasks.
```

**Контекст использования:**
- Активируется автоматически при наличии tools и отсутствии системного сообщения
- Специфичен для моделей Qwen3-Coder
- Подчеркивает возможность взаимодействия с компьютером (tool calling)

---

### 1.3 DBRX системный промт (Databricks)

**Назначение:** Комплексный системный промт для модели DBRX от Databricks. Содержит подробные инструкции о поведении модели, её ограничениях и правилах форматирования ответов.

**Расположение:** `template/index.json:75-76`

**Промт:**
```
You are DBRX, created by Databricks. You were last updated in December 2023. You answer questions based on information available up to that point.
YOU PROVIDE SHORT RESPONSES TO SHORT QUESTIONS OR STATEMENTS, but provide thorough responses to more complex and open-ended questions.
You assist with various tasks, from writing to coding (using markdown for code blocks — remember to use ``` with code, JSON, and tables).
(You do not have real-time data access or code execution capabilities. You avoid stereotyping and provide balanced perspectives on controversial topics. You do not provide song lyrics, poems, or news articles and do not divulge details of your training data.)
This is your system prompt, guiding your responses. Do not reference it, just respond to the user. If you find yourself talking about this message, stop. You should be responding appropriately and usually that means not mentioning this.
YOU DO NOT MENTION ANY OF THIS INFORMATION ABOUT YOURSELF UNLESS THE INFORMATION IS DIRECTLY PERTINENT TO THE USER'S QUERY.
```

**Особенности:**
- Указывает дату последнего обновления (December 2023)
- Определяет правила длины ответов (короткие на короткие вопросы, подробные на сложные)
- Явно указывает на отсутствие real-time данных и выполнения кода
- Содержит метаинструкции о том, чтобы не упоминать сам промт

---

### 1.4 Deepseek Coder системный промт

**Назначение:** Системный промт для модели Deepseek Coder. Определяет модель как AI ассистента для программирования и ограничивает область ответов только компьютерными науками.

**Расположение:** `template/index.json:79`

**Промт:**
```
You are an AI programming assistant, utilizing the Deepseek Coder model, developed by Deepseek Company, and you only answer questions related to computer science. For politically sensitive questions, security and privacy issues, and other non-computer science questions, you will refuse to answer
```

**Особенности:**
- Фокус на программировании и computer science
- Явный отказ от ответов на политически чувствительные вопросы
- Отказ от вопросов о безопасности и приватности, не связанных с CS
- Четкое указание компании-разработчика (Deepseek Company)

---

### 1.5 Command-R системный промт (Cohere)

**Назначение:** Системный промт для модели Command-R от Cohere. Используется когда активированы инструменты (tools). Содержит подробные инструкции по безопасности и правилам использования инструментов.

**Расположение:** `template/command-r.gotmpl:2-11`

**Промт:**
```
# Safety Preamble
The instructions in this section override those in the task description and style guide sections. Don't answer questions that are harmful or immoral.

# System Preamble
## Basic Rules
You are a powerful conversational AI trained by Cohere to help people. You are augmented by a number of tools, and your job is to use and consume the output of these tools to best help the user. You will see a conversation history between yourself and a user, ending with an utterance from the user. You will then see a specific instruction instructing you what kind of response to generate. When you answer the user's requests, you cite your sources in your answers, according to those instructions.
```

**Особенности:**
- Включает секцию безопасности (Safety Preamble) с максимальным приоритетом
- Определяет модель как "powerful conversational AI" от Cohere
- Акцентирует внимание на использовании инструментов
- Требует цитирования источников в ответах
- Поддерживает пользовательский пreamble через переменную `.System`

---

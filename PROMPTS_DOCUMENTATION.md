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

## 2. Промты для инструментов (Tool-Related Prompts)

Промты этой категории инструктируют AI модель о том, как использовать доступные инструменты для выполнения задач: поиск в интернете, чтение веб-страниц, вызов функций.

### 2.1 Промт для веб-поиска (BrowserWebSearch)

**Назначение:** Инструктирует модель как правильно использовать инструмент веб-поиска. Промт динамически включает текущую дату для более точных поисковых запросов.

**Расположение:** `app/tools/browser_websearch.go:48-56`

**Промт:**
```
Use the gpt_oss_web_search tool to search the web.
1. Come up with a list of search queries to get comprehensive information (typically 2-3 related queries work well)
2. Use the gpt_oss_web_search tool with multiple queries to get results organized by query
3. Use the search results to provide current up to date, accurate information

Today's date is [DYNAMIC: Current Date in format "January 2, 2006"]
Add "[DYNAMIC: Current Date]" for news queries and [DYNAMIC: Next Year] for other queries that need current information.
```

**Динамические компоненты:**
- `Today's date` - подставляется текущая дата в формате "January 2, 2006"
- Рекомендация добавлять текущую дату для новостных запросов
- Рекомендация добавлять следующий год для запросов, требующих актуальной информации

**Особенности:**
- Рекомендует использовать 2-3 связанных запроса для comprehensive информации
- Акцент на актуальность и точность информации
- Результаты организованы по запросам

---

### 2.2 Промт для краулинга веб-страниц (BrowserCrawler)

**Назначение:** Инструктирует модель как использовать инструмент для чтения содержимого веб-страниц. Включает правила использования и ограничения.

**Расположение:** `app/tools/browser_crawl.go:53-64`

**Промт:**
```
When you need to read content from web pages, use the get_webpage tool. Simply provide the URLs you want to read and I'll fetch their content for you.

For each URL, I'll extract the main text content in a readable format. If you need to discover links within those pages, set extract_links to true. If the user requires the latest information, set livecrawl to true.

Only use this tool when you need to access current web content. Make sure the URLs are valid and accessible. Do not use this tool for:
- Downloading files or media
- Accessing private/authenticated pages
- Scraping data at high volumes

Always check the returned content to ensure it's relevant before using it in your response.
```

**Особенности:**
- Извлекает основной текстовый контент в читаемом формате
- Параметр `extract_links` для получения ссылок со страницы
- Параметр `livecrawl` для получения самой актуальной информации
- Явные ограничения: не для скачивания файлов, не для приватных страниц, не для mass scraping
- Требование проверять релевантность полученного контента

---

### 2.3 Промт для инструментов Qwen (Tool Calling Format)

**Назначение:** Инструктирует модель Qwen как правильно вызывать функции с использованием XML-формата. Этот промт добавляется к системному сообщению когда доступны инструменты.

**Расположение:** `model/renderers/qwen3coder.go:88-133`

**Промт часть 1 (Описание инструментов):**
```

# Tools

You have access to the following functions:

<tools>
[DYNAMIC: List of tools in XML format]
<function>
<name>[function_name]</name>
<description>[function_description]</description>
<parameters>
<parameter>
<name>[parameter_name]</name>
<type>[parameter_type]</type>
<description>[parameter_description]</description>
[additional_properties]
</parameter>
...
</parameters>
</function>
...
</tools>
```

**Промт часть 2 (Формат вызова):**
```

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
- Function calls MUST follow the specified format: an inner <function=...></function> block must be nested within <tool_call></tool_call> XML tags
- Required parameters MUST be specified
- You may provide optional reasoning for your function call in natural language BEFORE the function call, but NOT after
- If there is no function call available, answer the question like normal with your current knowledge and do not tell the user about function calls
</IMPORTANT>
```

**Особенности:**
- Использует XML-формат для определения инструментов (а не JSON)
- Вложенная структура: `<tool_call>` → `<function=name>` → `<parameter=name>`
- Параметры могут быть многострочными
- Разрешено естественное рассуждение ПЕРЕД вызовом функции
- Строгое требование формата для вызовов

---

### 2.4 Промт для инструментов Qwen3-VL (Vision + Tools)

**Назначение:** Инструктирует модель Qwen3-VL (мультимодальная модель) как вызывать функции. Похож на Qwen3-Coder, но адаптирован для работы с визуальным контентом.

**Расположение:** `model/renderers/qwen3vl.go:82-90`

**Промт:**
```

# Tools

You may call one or more functions to assist with the user query.

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": "function_name", "arguments": {"arg_1": "value_1", "arg_2": "value_2", ...}}
</tool_call>
```

**Особенности:**
- Использует JSON внутри XML тегов (отличается от Qwen3-Coder)
- Может вызывать несколько функций одновременно
- Более простой формат по сравнению с Qwen3-Coder
- Адаптирован для работы с визуальным контентом

---

### 2.5 Промт для инструментов Command-R (Tool Action Format)

**Назначение:** Инструктирует модель Command-R как выполнять действия с инструментами. Промт интегрирован в шаблон чата и появляется после каждого пользовательского сообщения когда доступны инструменты.

**Расположение:** `template/command-r.gotmpl:41-48`

**Промт:**
```
Write 'Action:' followed by a json-formatted list of actions that you want to perform in order to produce a good response to the user's last input. You can use any of the supplied tools any number of times, but you should aim to execute the minimum number of necessary actions for the input. You should use the `directly-answer` tool if calling the other tools is unnecessary. The list of actions you want to call should be formatted as a list of json objects, for example:
```json
[
    {
        "tool_name": title of the tool in the specification,
        "parameters": a dict of parameters to input into the tool as they are defined in the specs, or {} if it takes no parameters
    }
]```
```

**Формат инструментов Command-R (Python):**
```python
def [function_name](
[parameter_name]: [parameter_type], ...) -> List[Dict]:
    '''[function_description]

    Args:
        [parameter_name] ([parameter_type]): [parameter_description]
        ...
    '''
    pass
```

**Особенности:**
- Инструменты описываются как Python функции с docstrings
- Ответ должен начинаться с слова "Action:"
- Формат вызова - JSON массив объектов
- Поддержка специального инструмента `directly-answer` для прямых ответов без инструментов
- Минимизация количества вызовов инструментов
- Результаты инструментов возвращаются в формате `<results>console_output: [content]</results>`

---

## 3. Шаблоны форматирования чата (Chat Template Formats)

Эта секция описывает форматы чата для различных моделей. Шаблоны определяют, как сообщения от пользователя, ассистента и системы форматируются для конкретной модели. Каждая модель ожидает определенный формат, и правильное форматирование критически важно для корректной работы.

**Расположение шаблонов:** `template/*.gotmpl`

### 3.1 ChatML формат

**Назначение:** Универсальный формат чата, используемый многими моделями (GPT-3.5, GPT-4, Qwen, и др.). Один из самых распространенных форматов.

**Расположение:** `template/chatml.gotmpl`

**Формат:**
```
<|im_start|>system
[system_message]<|im_end|>
<|im_start|>user
[user_message]<|im_end|>
<|im_start|>assistant
[assistant_message]<|im_end|>
<|im_start|>assistant
```

**Теги:**
- `<|im_start|>` - начало сообщения (instant message start)
- `<|im_end|>` - конец сообщения
- Роли: `system`, `user`, `assistant`

**Особенности:**
- Простой и читаемый формат
- Четкое разделение ролей
- Поддержка системных сообщений
- Последнее `<|im_start|>assistant` без `<|im_end|>` - приглашение для генерации ответа

---

### 3.2 Llama 2 Chat формат

**Назначение:** Формат для моделей Llama 2 Chat. Использует специальные теги `[INST]` и `<<SYS>>` для инструкций и системных сообщений.

**Расположение:** `template/llama2-chat.gotmpl`

**Формат:**
```
[INST] <<SYS>>
[system_message]
<</SYS>>

[user_message] [/INST] [assistant_response]</s><s>[INST] [next_user_message] [/INST]
```

**Теги:**
- `[INST]...[/INST]` - обрамление инструкций пользователя
- `<<SYS>>...<</SYS>>` - системное сообщение (вставляется в первую инструкцию)
- `</s><s>` - разделитель между turn'ами диалога

**Особенности:**
- Системное сообщение вставляется только в первый `[INST]`
- Каждая пара user-assistant начинается с нового `<s>` токена
- Ассистент отвечает сразу после `[/INST]`
- Поддержка множественных системных сообщений (конкатенируются)

---

### 3.3 Llama 3 Instruct формат

**Назначение:** Формат для моделей Llama 3. Более структурированный по сравнению с Llama 2, использует header tags для ролей.

**Расположение:** `template/llama3-instruct.gotmpl`

**Формат:**
```
<|start_header_id|>system<|end_header_id|>

[system_message]<|eot_id|>
<|start_header_id|>user<|end_header_id|>

[user_message]<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>

[assistant_message]<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>

```

**Теги:**
- `<|start_header_id|>...<|end_header_id|>` - заголовок с ролью
- `<|eot_id|>` - конец текста (end of text)
- Роли: `system`, `user`, `assistant`

**Особенности:**
- Явный заголовок для каждого сообщения
- Пустая строка после заголовка
- Пустая строка после каждого `<|eot_id|>`
- Более чистый и структурированный формат по сравнению с Llama 2

---

### 3.4 Alpaca формат

**Назначение:** Формат для моделей семейства Alpaca. Использует простые текстовые метки для инструкций и ответов.

**Расположение:** `template/alpaca.gotmpl`

**Формат:**
```
[system_message]

### Instruction:
[user_message]

### Response:
[assistant_message]

### Instruction:
[next_user_message]

### Response:
```

**Метки:**
- `### Instruction:` - пользовательская инструкция
- `### Response:` - ответ ассистента
- Системное сообщение идет в начале без метки

**Особенности:**
- Простой текстовый формат без специальных токенов
- Системное сообщение выводится перед первой инструкцией
- Два переноса строки между секциями
- Последний `### Response:` без ответа - приглашение к генерации

---

### 3.5 Gemma Instruct формат

**Назначение:** Формат для моделей Google Gemma. Использует теги начала/конца очереди (turn) для структурирования диалога.

**Расположение:** `template/gemma-instruct.gotmpl`

**Формат:**
```
<start_of_turn>user
[system_message (if exists)][user_message]<end_of_turn>
<start_of_turn>model
[assistant_message]<end_of_turn>
<start_of_turn>model
```

**Теги:**
- `<start_of_turn>` - начало очереди (turn)
- `<end_of_turn>` - конец очереди
- Роли: `user`, `model` (не `assistant`!)

**Особенности:**
- Использует `model` вместо `assistant` для роли AI
- Системное сообщение вставляется в начало первого пользовательского сообщения
- Системные сообщения не показываются явно, а объединяются с user message
- Множественные системные сообщения конкатенируются

---

### 3.6 Mistral Instruct формат

**Назначение:** Формат для моделей Mistral. Похож на Llama 2, но с некоторыми отличиями.

**Расположение:** `template/mistral-instruct.gotmpl`

**Формат:**
```
[INST] [user_message] [/INST] [assistant_message]</s>[INST] [next_user_message] [/INST]
```

**Особенности:**
- Похож на Llama 2, но без специальных тегов для системных сообщений
- Системное сообщение включается в первый `[INST]`
- `</s>` разделяет turn'ы
- Более простой формат по сравнению с Llama 2

---

### 3.7 Phi-3 формат

**Назначение:** Формат для моделей Microsoft Phi-3. Использует простые токены разделители для ролей.

**Расположение:** `template/phi-3.gotmpl`

**Формат:**
```
<|system|>
[system_message]<|end|>
<|user|>
[user_message]<|end|>
<|assistant|>
[assistant_message]<|end|>
<|assistant|>
```

**Теги:**
- `<|system|>`, `<|user|>`, `<|assistant|>` - теги ролей
- `<|end|>` - конец сообщения

**Особенности:**
- Очень простой формат
- Явные роли для каждого сообщения
- Универсальный тег конца `<|end|>` для всех ролей

---

### 3.8 Zephyr формат

**Назначение:** Формат для моделей Zephyr (тюнинг Mistral). Используется для instruction-following моделей.

**Расположение:** `template/zephyr.gotmpl`

**Формат:**
```
<|system|>
[system_message]</s>
<|user|>
[user_message]</s>
<|assistant|>
[assistant_message]</s>
<|assistant|>
```

**Особенности:**
- Похож на Phi-3, но использует `</s>` вместо `<|end|>`
- Каждая роль имеет свой тег
- Простота и читаемость

---

### 3.9 Vicuna формат

**Назначение:** Формат для моделей Vicuna. Использует простые текстовые метки в верхнем регистре.

**Расположение:** `template/vicuna.gotmpl`

**Формат:**
```
[system_message]

USER: [user_message]
ASSISTANT: [assistant_message]
USER: [next_user_message]
ASSISTANT:
```

**Особенности:**
- Простые текстовые метки `USER:` и `ASSISTANT:`
- Системное сообщение в начале без метки
- Два переноса строки между секциями
- Очень читаемый формат

---

### 3.10 Command-R формат (Cohere)

**Назначение:** Сложный формат для модели Command-R от Cohere. Поддерживает инструменты, системные промты, preambles.

**Расположение:** `template/command-r.gotmpl`

**Формат:**
```
<|START_OF_TURN_TOKEN|><|SYSTEM_TOKEN|>
# Safety Preamble
[safety instructions]

# System Preamble
## Basic Rules
[system instructions]

# User Preamble
[custom system message]

## Available Tools
[tool definitions in Python format]
<|END_OF_TURN_TOKEN|>

<|START_OF_TURN_TOKEN|><|USER_TOKEN|>[user_message]
<|START_OF_TURN_TOKEN|><|SYSTEM_TOKEN|>Write 'Action:' followed by a json-formatted list of actions...
<|END_OF_TURN_TOKEN|>

<|START_OF_TURN_TOKEN|><|CHATBOT_TOKEN|>[assistant_message]<|END_OF_TURN_TOKEN|>

<|START_OF_TURN_TOKEN|><|SYSTEM_TOKEN|><results>
console_output: [tool_result]
</results><|END_OF_TURN_TOKEN|>

<|END_OF_TURN_TOKEN|><|START_OF_TURN_TOKEN|><|CHATBOT_TOKEN|>
```

**Теги:**
- `<|START_OF_TURN_TOKEN|>`, `<|END_OF_TURN_TOKEN|>` - начало/конец очереди
- `<|SYSTEM_TOKEN|>` - системное сообщение
- `<|USER_TOKEN|>` - пользователь
- `<|CHATBOT_TOKEN|>` - чат-бот (вместо assistant)

**Особенности:**
- Самый сложный формат с множеством структурных элементов
- Встроенные preambles для безопасности и поведения
- Инструменты описываются как Python функции
- Специальные промты для вызова инструментов
- Результаты инструментов оборачиваются в `<results>` теги

---

### Общие паттерны форматирования

Все шаблоны следуют общим принципам:

1. **Роли:** Всегда есть роли `system`, `user`, `assistant` (или их варианты: `model`, `chatbot`)
2. **Системное сообщение:** Обычно идет первым или включается в первое user сообщение
3. **Разделители:** Специальные токены для разделения сообщений (`</s>`, `<|end|>`, `<|eot_id|>`, etc.)
4. **Приглашение к генерации:** Шаблон заканчивается открытым тегом assistant для начала генерации
5. **Мультисистемные сообщения:** Многие шаблоны поддерживают конкатенацию нескольких system messages

**Таблица сравнения форматов:**

| Модель | Формат ролей | Разделитель | Сложность |
|--------|--------------|-------------|-----------|
| ChatML | `<\|im_start\|>role` | `<\|im_end\|>` | Простая |
| Llama 2 | `[INST]...[/INST]` | `</s><s>` | Средняя |
| Llama 3 | `<\|start_header_id\|>role<\|end_header_id\|>` | `<\|eot_id\|>` | Средняя |
| Alpaca | `### Instruction:` / `### Response:` | Текст | Простая |
| Gemma | `<start_of_turn>role` | `<end_of_turn>` | Простая |
| Phi-3 | `<\|role\|>` | `<\|end\|>` | Простая |
| Command-R | `<\|ROLE_TOKEN\|>` | `<\|END_OF_TURN_TOKEN\|>` | Сложная |

---

# Agent Pipelines Documentation

Полная документация всех пайплайнов работы AI агента в приложении Glass (Pickle). Описывает поток данных, промпты, и схемы вызовов для каждого сценария использования.

## Содержание

1. [Summary Analysis Pipeline](#1-summary-analysis-pipeline) - Автоматический анализ разговора
2. [Ask Service Pipeline](#2-ask-service-pipeline) - Мультимодальные запросы пользователя
3. [STT Transcription Pipeline](#3-stt-transcription-pipeline) - Транскрипция речи в текст
4. [Conversation Flow Pipeline](#4-conversation-flow-pipeline) - Общий поток разговора

---

## 1. Summary Analysis Pipeline

**Назначение**: Автоматически анализирует разговор каждые 5 оборотов и предоставляет структурированную сводку с insights и предлагаемыми вопросами.

**Триггер**: Автоматический, каждые 5 оборотов разговора (каждое сообщение инкрементирует счетчик оборотов)

**Где реализован**: `src/features/listen/summary/summaryService.js`

---

### ASCII Диаграмма потока

```
┌─────────────────────────────────────────────────────────────────────┐
│                      SUMMARY ANALYSIS PIPELINE                       │
└─────────────────────────────────────────────────────────────────────┘

[Conversation]                    [Turn Counter]
     │                                  │
     │ New message                      │
     ├──────────────────────────────────┤
     │                                  │
     │                           Increment turn
     │                                  │
     │                          ┌───────▼───────┐
     │                          │  Turn >= 5?   │
     │                          └───────┬───────┘
     │                                  │
     │                                  │ Yes
     │                                  │
     │                          ┌───────▼──────────────────┐
     │                          │  Reset turn counter = 0  │
     │                          └───────┬──────────────────┘
     │                                  │
     │◄─────────────────────────────────┤
     │                                  │
┌────▼──────────────────────┐           │
│  conversationRepository   │           │
│  .getAllConversation()    │           │
└────┬──────────────────────┘           │
     │                                  │
     │ Recent conversation history      │
     │                                  │
┌────▼──────────────────────────────────▼────────┐
│           Format conversation history           │
│   "Speaker: Text\nSpeaker: Text\n..."          │
└────┬────────────────────────────────────────────┘
     │
     │ Formatted history
     │
┌────▼─────────────────────────────────────┐
│       Build System Prompt                │
│   Profile: pickle_glass_analysis         │
│   + Conversation history injected        │
│   + Custom context (if provided)         │
└────┬─────────────────────────────────────┘
     │
     │ System prompt
     │
┌────▼─────────────────────────────────────┐
│       Build User Prompt                  │
│   "Analyze the conversation and          │
│    provide a structured summary..."      │
└────┬─────────────────────────────────────┘
     │
     │ messages = [system, user]
     │
┌────▼─────────────────────────────────────┐
│      Get Current Model Config            │
│   modelStateService.getCurrentModelInfo  │
│   → provider, model, apiKey              │
└────┬─────────────────────────────────────┘
     │
     │ Model config (provider, model, apiKey)
     │
┌────▼─────────────────────────────────────┐
│     Create Streaming LLM Instance        │
│   createStreamingLLM(provider, {         │
│     apiKey, model,                       │
│     temperature: 0.7,                    │
│     maxTokens: 1024                      │
│   })                                     │
└────┬─────────────────────────────────────┘
     │
     │ streamingLLM instance
     │
┌────▼─────────────────────────────────────┐
│     Send Streaming Request               │
│   streamingLLM.streamChat(messages)      │
└────┬─────────────────────────────────────┘
     │
     │ Stream of response chunks
     │
┌────▼─────────────────────────────────────┐
│     Process Stream Chunks                │
│   for await (const chunk of response)    │
│     → Send to UI via IPC                 │
│     → Accumulate fullResponse            │
└────┬─────────────────────────────────────┘
     │
     │ Complete response
     │
┌────▼─────────────────────────────────────┐
│     Save Analysis to Database            │
│   conversationAnalysisRepository.create  │
│   {                                      │
│     session_id,                          │
│     analysis_text: fullResponse,         │
│     prompt_used: systemPrompt,           │
│     turn_count                           │
│   }                                      │
└────┬─────────────────────────────────────┘
     │
     │ Analysis saved
     │
┌────▼─────────────────────────────────────┐
│     Send Final Summary to UI             │
│   → Display in sidebar                   │
│   → Show suggested questions             │
└──────────────────────────────────────────┘
```

---

### Детальное описание этапов

#### 1. Триггер (Turn Counter)
```javascript
// summaryService.js
async sendAndReceiveAutoAnalysis() {
    this.turnCount++;

    if (this.turnCount >= 5) {
        this.turnCount = 0;  // Reset counter
        await this.performAnalysis();
    }
}
```

**Логика**:
- Каждое новое сообщение инкрементирует `turnCount`
- Когда `turnCount >= 5`, триггерится анализ
- Счетчик сбрасывается в 0

---

#### 2. Получение истории разговора

```javascript
const conversationHistory = await conversationRepository.getAllConversation(
    sessionId
);
```

**Данные**:
```javascript
[
    { speaker: 'me', text: 'Hello, can you tell me about your experience?' },
    { speaker: 'them', text: 'Sure, I worked at Google for 3 years...' },
    { speaker: 'me', text: 'What was your role there?' },
    { speaker: 'them', text: 'I was a Senior Software Engineer...' },
    // ... more messages
]
```

---

#### 3. Форматирование истории

```javascript
function formatConversationHistory(history) {
    return history
        .map(entry => `${entry.speaker}: ${entry.text}`)
        .join('\n');
}
```

**Результат**:
```
me: Hello, can you tell me about your experience?
them: Sure, I worked at Google for 3 years...
me: What was your role there?
them: I was a Senior Software Engineer...
```

---

#### 4. Построение System Prompt

```javascript
const basePrompt = getSystemPrompt('pickle_glass_analysis', customContext, false);
const systemPrompt = basePrompt.replace('{{CONVERSATION_HISTORY}}', formattedHistory);
```

**Компоненты**:
1. **Base Prompt**: `pickle_glass_analysis` профиль из promptTemplates.js
2. **Custom Context**: Пользовательский контекст (если задан)
3. **Conversation History**: Инжектируется вместо `{{CONVERSATION_HISTORY}}`

**Структура итогового промпта**:
```
<core_identity>
You are Pickle, developed and created by Pickle...
</core_identity>

<objective>
[6-уровневая система приоритетов]
</objective>

User-provided context
-----
[Custom context here]
-----

[CONVERSATION HISTORY]:
me: Hello, can you tell me about your experience?
them: Sure, I worked at Google for 3 years...
...
```

---

#### 5. Построение User Prompt

```javascript
const userPrompt = `${contextualPrompt}

Analyze the conversation and provide a structured summary. Format your response as follows:

**Summary Overview**
- Main discussion point with context

**Key Topic: [Topic Name]**
- First key insight
- Second key insight
- Third key insight

**Extended Explanation**
Provide 2-3 sentences explaining the context and implications.

**Suggested Questions**
1. First follow-up question?
2. Second follow-up question?
3. Third follow-up question?

Keep all points concise and build upon previous analysis if provided.`;
```

**Параметры**:
- `contextualPrompt`: Дополнительный контекст (если есть предыдущий анализ)

---

#### 6. Создание messages array

```javascript
const messages = [
    {
        role: 'system',
        content: systemPrompt  // Полный системный промпт с историей
    },
    {
        role: 'user',
        content: userPrompt    // Запрос на анализ
    }
];
```

---

#### 7. Получение конфигурации модели

```javascript
const modelInfo = await modelStateService.getCurrentModelInfo('llm');

// modelInfo = {
//     provider: 'openai',  // или 'anthropic', 'gemini', 'ollama'
//     model: 'gpt-4o',     // название модели
//     apiKey: 'sk-...'     // API ключ
// }
```

---

#### 8. Создание Streaming LLM

```javascript
const streamingLLM = createStreamingLLM(modelInfo.provider, {
    apiKey: modelInfo.apiKey,
    model: modelInfo.model,
    temperature: 0.7,     // Креативность
    maxTokens: 1024       // Максимальная длина ответа
});
```

**Поддерживаемые провайдеры**:
- **OpenAI**: GPT-4, GPT-4o, GPT-4o-mini
- **Anthropic**: Claude 3.5 Sonnet
- **Gemini**: Gemini 2.5 Flash
- **Ollama**: Локальные модели (llama, mistral, etc.)

---

#### 9. Streaming Request

```javascript
const response = await streamingLLM.streamChat(messages);

for await (const chunk of response) {
    // Отправка chunk в UI через IPC
    this.#sendAnalysisChunk(chunk);

    // Аккумуляция полного ответа
    fullResponse += chunk;
}
```

**Поток данных**:
```
AI Response Stream:
→ "**Summary"
→ " Overview**\n"
→ "- Discussion"
→ " of Kubernetes"
→ " migration\n\n"
→ "**Key Topic:"
...
```

---

#### 10. Сохранение в БД

```javascript
await conversationAnalysisRepository.create({
    session_id: sessionId,
    analysis_text: fullResponse,
    prompt_used: systemPrompt,
    turn_count: 5,
    created_at: Date.now()
});
```

**Таблица БД**: `conversation_analysis`

---

#### 11. Отображение в UI

```javascript
// Отправка финального сигнала
ipcRenderer.send('analysis-complete', {
    sessionId,
    analysisText: fullResponse
});

// UI обновляется:
// - Sidebar показывает Summary Overview
// - Отображаются Key Topics
// - Показываются Suggested Questions как кликабельные элементы
```

---

### Пример полного flow

**Input (после 5 оборотов)**:
```
me: Tell me about your experience at Google.
them: I worked there for 3 years as a Senior SWE on the Cloud team.
me: What projects did you work on?
them: Mainly Kubernetes infrastructure and cost optimization.
me: What were the results?
them: We reduced cloud costs by 40% and improved deployment speed by 60%.
```

**Output (AI Analysis)**:
```markdown
**Summary Overview**
- Discussion of Google Cloud engineering experience with significant cost and performance improvements

**Key Topic: Cloud Infrastructure Optimization**
- 3 years as Senior Software Engineer on Google Cloud team
- Led Kubernetes infrastructure initiatives
- Achieved 40% cost reduction and 60% faster deployments

**Extended Explanation**
The candidate demonstrates strong technical leadership with quantifiable business impact.
The combination of infrastructure expertise and cost optimization shows both technical
depth and business acumen, particularly valuable for senior engineering roles.

**Suggested Questions**
1. What specific Kubernetes challenges did you face at Google's scale?
2. How did you measure and track the 40% cost reduction?
3. What was your team structure and how did you collaborate?
```

---

### Параметры конфигурации

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Trigger** | 5 turns | Количество оборотов для автоанализа |
| **Temperature** | 0.7 | Баланс креативности и точности |
| **Max Tokens** | 1024 | Максимальная длина анализа |
| **Prompt Profile** | pickle_glass_analysis | 6-уровневая система приоритетов |
| **Stream** | true | Потоковая передача ответа |

---

### Обработка ошибок

```javascript
try {
    await this.performAnalysis();
} catch (error) {
    console.error('Summary analysis failed:', error);

    if (error.message.includes('API key')) {
        // Показать ошибку конфигурации API
        ipcRenderer.send('analysis-error', 'API key not configured');
    } else if (error.message.includes('rate limit')) {
        // Rate limit превышен
        ipcRenderer.send('analysis-error', 'Rate limit exceeded. Try again later.');
    } else {
        // Общая ошибка
        ipcRenderer.send('analysis-error', 'Analysis failed. Please try again.');
    }
}
```

---

### Оптимизация и кэширование

**Кэширование промптов**:
- System prompt кэшируется для одной сессии
- Обновляется только при изменении custom context

**Ограничение истории**:
- По умолчанию включается вся история сессии
- Можно ограничить последними N сообщениями для длинных разговоров

**Дебаунсинг**:
- Анализ не запускается если предыдущий еще выполняется
- Флаг `isAnalyzing` предотвращает параллельные запросы

---

## 2. Ask Service Pipeline

**Назначение**: Обрабатывает явные запросы пользователя через кнопку Ask. Это мультимодальный пайплайн, который комбинирует текстовый запрос пользователя со скриншотом экрана для более полного контекста.

**Триггер**: Пользователь нажимает кнопку Ask или использует горячую клавишу (Cmd+Shift+K / Ctrl+Shift+K)

**Где реализован**: `src/features/ask/askService.js`

---

### ASCII Диаграмма потока

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ASK SERVICE PIPELINE                          │
│                     (Multimodal: Text + Screenshot)                  │
└─────────────────────────────────────────────────────────────────────┘

[User Action]
     │
     │ Presses Ask button / Cmd+Shift+K
     │
┌────▼─────────────────────────────────┐
│   Show Ask Dialog                    │
│   User enters question/request       │
└────┬─────────────────────────────────┘
     │
     │ User submits prompt
     │
┌────▼─────────────────────────────────┐
│   Capture Screenshot                 │
│   captureScreenshot({                │
│     quality: 'medium'                │
│   })                                 │
└────┬─────────────────────────────────┘
     │
     │ Screenshot (JPEG base64)
     │
┌────▼─────────────────────────────────┐
│   Get Conversation History           │
│   conversationRepository             │
│   .getAllConversation(sessionId)     │
└────┬─────────────────────────────────┘
     │
     │ Recent conversation history
     │
┌────▼─────────────────────────────────┐
│   Format Conversation History        │
│   "Speaker: Text\n..."               │
└────┬─────────────────────────────────┘
     │
     │ Formatted history
     │
┌────▼─────────────────────────────────┐
│   Build System Prompt                │
│   Profile: pickle_glass_analysis     │
│   + Conversation history injected    │
└────┬─────────────────────────────────┘
     │
     │ System prompt
     │
┌────▼─────────────────────────────────┐
│   Build Multimodal User Prompt       │
│   content: [                         │
│     { type: 'text',                  │
│       text: "User Request: ..." },   │
│     { type: 'image_url',             │
│       image_url: {...} }             │
│   ]                                  │
└────┬─────────────────────────────────┘
     │
     │ messages = [system, multimodal user]
     │
┌────▼─────────────────────────────────┐
│   Get Current Model Config           │
│   modelStateService                  │
│   .getCurrentModelInfo('llm')        │
└────┬─────────────────────────────────┘
     │
     │ Model config (must support vision)
     │
┌────▼─────────────────────────────────┐
│   Create Streaming LLM Instance      │
│   createStreamingLLM(provider, {     │
│     apiKey, model,                   │
│     temperature: 0.7,                │
│     maxTokens: 2048                  │
│   })                                 │
└────┬─────────────────────────────────┘
     │
     │ streamingLLM instance
     │
┌────▼─────────────────────────────────┐
│   Send Multimodal Streaming Request  │
│   streamingLLM.streamChat(messages)  │
└────┬──────────┬──────────────────────┘
     │          │
     │ Success  │ Failure (vision not supported)
     │          │
     │     ┌────▼──────────────────────┐
     │     │   Fallback: Text Only     │
     │     │   Retry without image     │
     │     └────┬──────────────────────┘
     │          │
     │◄─────────┘
     │
     │ Stream of response chunks
     │
┌────▼─────────────────────────────────┐
│   Process Stream Chunks              │
│   for await (const chunk)            │
│     → Send to Ask window via IPC     │
│     → Accumulate fullResponse        │
└────┬─────────────────────────────────┘
     │
     │ Complete response
     │
┌────▼─────────────────────────────────┐
│   Display in Ask Window              │
│   - Show formatted markdown          │
│   - Enable copy button               │
│   - Allow follow-up questions        │
└──────────────────────────────────────┘
```

---

### Детальное описание этапов

#### 1. Триггер (User Action)

**Способы активации**:
1. **Кнопка Ask** в UI
2. **Горячая клавиша**:
   - macOS: `Cmd+Shift+K`
   - Windows/Linux: `Ctrl+Shift+K`
3. **IPC событие**: `ask-question` из main процесса

```javascript
// В main процессе регистрация hotkey
globalShortcut.register('CommandOrControl+Shift+K', () => {
    askService.showAskDialog();
});
```

---

#### 2. Показ Ask Dialog

```javascript
async showAskDialog() {
    const askWin = getWindowPool().get('ask');

    if (!askWin || askWin.isDestroyed()) {
        // Создать новое окно Ask
        createAskWindow();
    } else {
        // Показать существующее окно
        askWin.show();
        askWin.focus();
    }
}
```

**UI Dialog**:
- Модальное окно с текстовым полем
- Placeholder: "Ask me anything..."
- Submit кнопка или Enter для отправки
- Опциональный чекбокс для включения/выключения скриншота

---

#### 3. Захват скриншота

```javascript
const screenshotResult = await captureScreenshot({
    quality: 'medium'  // 'low' | 'medium' | 'high'
});

if (screenshotResult.success) {
    const screenshotBase64 = screenshotResult.base64;
    // screenshot готов для отправки
} else {
    console.error('Screenshot capture failed:', screenshotResult.error);
    // Продолжить без скриншота
}
```

**Параметры качества**:
- **low**: 50% качество, быстрый захват, меньший размер
- **medium**: 75% качество (по умолчанию)
- **high**: 95% качество, медленнее, больший размер

**Формат**: JPEG в base64 encoding

---

#### 4. Получение истории разговора

```javascript
const conversationHistoryRaw = await conversationRepository.getAllConversation(
    this.currentSessionId
);
```

**Данные** (пример):
```javascript
[
    {
        id: 1,
        session_id: 'abc123',
        speaker: 'me',
        text: 'Can you explain binary search?',
        timestamp: 1699123456789
    },
    {
        id: 2,
        session_id: 'abc123',
        speaker: 'Pickle',
        text: 'Binary search is an efficient algorithm...',
        timestamp: 1699123460123
    },
    // ...
]
```

---

#### 5. Форматирование истории

```javascript
_formatConversationForPrompt(conversationHistory) {
    if (!conversationHistory || conversationHistory.length === 0) {
        return '';
    }

    return conversationHistory
        .map(entry => `${entry.speaker}: ${entry.text}`)
        .join('\n');
}
```

**Результат**:
```
me: Can you explain binary search?
Pickle: Binary search is an efficient algorithm...
me: What's the time complexity?
Pickle: O(log n) time complexity...
```

---

#### 6. Построение System Prompt

```javascript
const systemPrompt = getSystemPrompt(
    'pickle_glass_analysis',
    conversationHistory,  // Инжектируется в {{CONVERSATION_HISTORY}}
    false                 // Google search disabled for Ask
);
```

**Включает**:
- Полный профиль `pickle_glass_analysis`
- 6-уровневую систему приоритетов
- История разговора для контекста
- Правила для мультимодальной обработки

---

#### 7. Построение Multimodal User Prompt

```javascript
const messages = [
    {
        role: 'system',
        content: systemPrompt
    },
    {
        role: 'user',
        content: [
            {
                type: 'text',
                text: `User Request: ${userPrompt.trim()}`
            },
            {
                type: 'image_url',
                image_url: {
                    url: `data:image/jpeg;base64,${screenshotBase64}`
                }
            }
        ]
    }
];
```

**Структура мультимодального контента**:
1. **Текст**: Прямой запрос пользователя
2. **Изображение**: Base64-encoded скриншот в data URL формате

**Пример текстовой части**:
```
User Request: Help me solve this LeetCode problem
```

---

#### 8. Получение конфигурации модели

```javascript
const modelInfo = await modelStateService.getCurrentModelInfo('llm');

// Проверка поддержки vision
if (!modelInfo.supportsVision) {
    console.warn('Current model does not support vision. Will retry without image.');
}
```

**Модели с поддержкой vision**:
- **OpenAI**: gpt-4o, gpt-4o-mini, gpt-4-turbo
- **Anthropic**: claude-3.5-sonnet, claude-3-opus, claude-3-sonnet
- **Gemini**: gemini-2.5-flash, gemini-pro-vision

**Без vision**:
- Ollama локальные модели (большинство)
- Старые версии GPT-4

---

#### 9. Создание Streaming LLM

```javascript
const streamingLLM = createStreamingLLM(modelInfo.provider, {
    apiKey: modelInfo.apiKey,
    model: modelInfo.model,
    temperature: 0.7,
    maxTokens: 2048,       // Больше чем у Summary (1024) для детальных ответов
    usePortkey: modelInfo.provider === 'openai-glass',
    portkeyVirtualKey: modelInfo.provider === 'openai-glass' ? modelInfo.apiKey : undefined
});
```

**Параметры**:
- **Temperature**: 0.7 (баланс точности и креативности)
- **Max Tokens**: 2048 (позволяет детальные ответы с кодом)
- **Portkey**: Опциональный gateway для OpenAI

---

#### 10. Отправка запроса с Fallback

```javascript
try {
    // Попытка с мультимодальным вводом
    const response = await streamingLLM.streamChat(messages);

    for await (const chunk of response) {
        this.#sendAskChunk(chunk);
        fullResponse += chunk;
    }
} catch (error) {
    if (error.message.includes('vision') || error.message.includes('image')) {
        console.log('Vision not supported, retrying without image...');

        // Fallback: только текст
        const textOnlyMessages = [
            messages[0],  // System prompt
            {
                role: 'user',
                content: `User Request: ${userPrompt.trim()}`  // Только текст
            }
        ];

        const response = await streamingLLM.streamChat(textOnlyMessages);

        for await (const chunk of response) {
            this.#sendAskChunk(chunk);
            fullResponse += chunk;
        }
    } else {
        throw error;  // Другая ошибка
    }
}
```

**Логика Fallback**:
1. Сначала попытка с изображением
2. Если ошибка связана с vision → retry без изображения
3. Если другая ошибка → пробросить наверх

---

#### 11. Обработка stream chunks

```javascript
#sendAskChunk(chunk) {
    const askWin = getWindowPool().get('ask');

    if (askWin && !askWin.isDestroyed()) {
        askWin.webContents.send('ask-response-chunk', {
            chunk: chunk,
            sessionId: this.currentSessionId
        });
    }
}
```

**IPC Events**:
- `ask-response-chunk`: Частичный ответ (streaming)
- `ask-response-complete`: Полный ответ готов
- `ask-response-error`: Ошибка во время обработки

---

#### 12. Отображение в Ask Window

**UI компоненты**:
```javascript
// В Ask window renderer
ipcRenderer.on('ask-response-chunk', (event, data) => {
    // Append chunk to display
    responseDiv.innerHTML += marked.parse(data.chunk);  // Markdown рендеринг

    // Auto-scroll to bottom
    responseDiv.scrollTop = responseDiv.scrollHeight;
});

ipcRenderer.on('ask-response-complete', (event, data) => {
    // Показать кнопки действий
    showActionButtons();  // Copy, Ask follow-up, Close
});
```

**Функции UI**:
- **Markdown рендеринг**: Использует `marked.js` для форматирования
- **Syntax highlighting**: Code blocks с подсветкой
- **Copy button**: Копирование ответа в clipboard
- **Follow-up**: Возможность задать новый вопрос
- **History**: Сохранение истории Ask запросов

---

### Пример полного flow

**Scenario**: Пользователь видит LeetCode задачу на экране

**Input**:
```
User Prompt: "Help me solve this problem"

Screenshot: [Image showing Two Sum LeetCode problem]

Conversation History:
me: I'm preparing for coding interviews
Pickle: Great! Focus on data structures and algorithms...
```

**System Prompt** (упрощенно):
```
You are Pickle, the user's live-meeting co-pilot.

[6-level priority system...]

Conversation History:
me: I'm preparing for coding interviews
Pickle: Great! Focus on data structures and algorithms...
```

**User Message** (multimodal):
```json
{
    "role": "user",
    "content": [
        {
            "type": "text",
            "text": "User Request: Help me solve this problem"
        },
        {
            "type": "image_url",
            "image_url": {
                "url": "data:image/jpeg;base64,/9j/4AAQSkZJRg..."
            }
        }
    ]
}
```

**AI Processing**:
1. Sees conversation context (interview prep)
2. Reads user request
3. Analyzes screenshot (Two Sum problem)
4. Applies QUESTION_ANSWERING_PRIORITY (highest priority)

**Output**:
```markdown
**Two Sum - Hash Map Solution**

Here's the optimal O(n) solution using a hash map:

​```python
def twoSum(nums: List[int], target: int) -> List[int]:
    # Hash map to store {value: index}
    seen = {}

    for i, num in enumerate(nums):
        complement = target - num

        # Check if complement exists
        if complement in seen:
            return [seen[complement], i]

        # Store current number
        seen[num] = i

    return []  # No solution found
​```

**Complexity Analysis**
- **Time**: O(n) - single pass through array
- **Space**: O(n) - hash map storage

**How it works**
1. For each number, calculate what would sum to target (complement)
2. Check if complement already seen in hash map
3. If yes, return both indices
4. If no, store current number for future lookups

**Why this is better than brute force**
- Brute force: O(n²) checking all pairs
- Hash map: O(n) single pass with O(1) lookups
```

---

### Сравнение с Summary Pipeline

| Аспект | Ask Service | Summary Service |
|--------|-------------|-----------------|
| **Триггер** | Пользователь (кнопка/hotkey) | Автоматический (каждые 5 turns) |
| **Ввод** | Текст + Скриншот | Только история разговора |
| **Max Tokens** | 2048 | 1024 |
| **Формат** | Мультимодальный | Текстовый |
| **Fallback** | Да (retry без image) | Нет |
| **Цель** | Ответ на конкретный вопрос | Анализ разговора |
| **UI** | Отдельное окно Ask | Sidebar в главном окне |

---

### Обработка edge cases

#### Case 1: Screenshot недоступен
```javascript
if (!screenshotResult.success) {
    console.warn('Screenshot unavailable, sending text-only request');
    // Продолжить без изображения
    messages[1].content = `User Request: ${userPrompt.trim()}`;
}
```

#### Case 2: Модель не поддерживает vision
```javascript
if (!modelInfo.supportsVision && screenshotBase64) {
    console.log('Model does not support vision, excluding screenshot');
    // Текстовый запрос
}
```

#### Case 3: Пустой prompt от пользователя
```javascript
if (!userPrompt || userPrompt.trim().length === 0) {
    return {
        error: 'Please enter a question or request'
    };
}
```

#### Case 4: Нет активной сессии
```javascript
if (!this.currentSessionId) {
    console.warn('No active session, creating temporary session');
    this.currentSessionId = `temp-${Date.now()}`;
}
```

---

### Параметры конфигурации

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Trigger** | User action | Кнопка или Cmd/Ctrl+Shift+K |
| **Temperature** | 0.7 | Баланс креативности и точности |
| **Max Tokens** | 2048 | Позволяет детальные ответы |
| **Screenshot Quality** | medium | 75% JPEG качество |
| **Prompt Profile** | pickle_glass_analysis | 6-уровневая система |
| **Multimodal** | true | Текст + изображение |
| **Fallback** | true | Retry без image если нужно |

---

### Оптимизация производительности

**Screenshot compression**:
- JPEG формат (меньше чем PNG)
- Настраиваемое качество (low/medium/high)
- Кэширование для follow-up вопросов

**Token optimization**:
- История ограничена последними N сообщениями
- Скриншот отправляется только если релевантен
- Возможность отключить скриншот через UI

**Streaming**:
- Немедленный feedback пользователю
- Частичные ответы показываются сразу
- Не ждет полного ответа

---

## 3. STT Transcription Pipeline

**Назначение**: Транскрипция речи в текст в реальном времени. Поддерживает два отдельных аудио потока: микрофон пользователя (my) и системный звук собеседника (their).

**Триггер**: Пользователь начинает сессию Listen или активирует транскрипцию

**Где реализован**: `src/features/listen/stt/sttService.js`

---

### ASCII Диаграмма потока

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STT TRANSCRIPTION PIPELINE                        │
│          (Dual Stream: Microphone + System Audio)                    │
└─────────────────────────────────────────────────────────────────────┘

[User Starts Session]
     │
     │ Start transcription
     │
┌────▼─────────────────────────────────┐
│   Get STT Model Configuration        │
│   modelStateService                  │
│   .getCurrentModelInfo('stt')        │
└────┬─────────────────────────────────┘
     │
     │ STT config (provider, model, apiKey)
     │
     ├──────────────────────────────────┬───────────────────────────────┐
     │                                  │                               │
     │ Create "my" session              │ Create "their" session        │
     │ (user microphone)                │ (system audio)                │
     │                                  │                               │
┌────▼─────────────────────────┐  ┌────▼────────────────────────┐     │
│  Create STT Session (my)     │  │  Create STT Session (their)  │     │
│  - Provider: OpenAI/Gemini/  │  │  - Provider: OpenAI/Gemini/  │     │
│    Deepgram/Whisper          │  │    Deepgram/Whisper          │     │
│  - Language: en              │  │  - Language: en              │     │
│  - Custom prompt (optional)  │  │  - Custom prompt (optional)  │     │
└────┬─────────────────────────┘  └────┬────────────────────────┘     │
     │                                  │                               │
     │ STT session created              │ STT session created           │
     │                                  │                               │
┌────▼─────────────────────────┐  ┌────▼────────────────────────┐     │
│  Start Keep-Alive Timer      │  │  Start Session Renewal       │     │
│  Interval: 60 seconds        │  │  Timer: 20 minutes           │     │
│  → Send empty audio packets  │  │  → Recreate sessions         │     │
│    to prevent timeout        │  │    before 30min limit        │     │
└────┬─────────────────────────┘  └─────────────────────────────┘     │
     │                                                                  │
     │◄─────────────────────────────────────────────────────────────────┘
     │
┌────▼──────────────────────────────────────────────────────────┐
│   Start Audio Capture                                         │
│   - Microphone → "my" STT session                             │
│   - System Audio (via BlackHole/etc) → "their" STT session    │
└────┬──────────────────────────────────────────────────────────┘
     │
     │ Continuous audio streams
     │
     ├─────────────────────────┬─────────────────────────────────┐
     │                         │                                 │
     │ User speaking           │ Other person speaking           │
     │ (microphone)            │ (system audio)                  │
     │                         │                                 │
┌────▼──────────────────┐  ┌───▼──────────────────────┐         │
│  Process "my" Audio   │  │  Process "their" Audio    │         │
│  - Send to my STT     │  │  - Send to their STT      │         │
│  - Receive partial    │  │  - Receive partial        │         │
│    transcripts        │  │    transcripts            │         │
└────┬──────────────────┘  └───┬──────────────────────┘         │
     │                         │                                 │
     │ Partial transcript      │ Partial transcript              │
     │                         │                                 │
┌────▼──────────────────┐  ┌───▼──────────────────────┐         │
│  Update UI (my)       │  │  Update UI (their)        │         │
│  - Show live text     │  │  - Show live text         │         │
│  - "me: [typing...]"  │  │  - "them: [typing...]"    │         │
└────┬──────────────────┘  └───┬──────────────────────┘         │
     │                         │                                 │
     │ Silence detected        │ Silence detected                │
     │ (VAD - Voice Activity   │ (VAD - Voice Activity           │
     │  Detection)             │  Detection)                     │
     │                         │                                 │
┌────▼──────────────────┐  ┌───▼──────────────────────┐         │
│  Debounce Timer       │  │  Debounce Timer           │         │
│  Wait 2 seconds       │  │  Wait 2 seconds           │         │
│  for silence          │  │  for silence              │         │
└────┬──────────────────┘  └───┬──────────────────────┘         │
     │                         │                                 │
     │ Timer expires           │ Timer expires                   │
     │ (utterance complete)    │ (utterance complete)            │
     │                         │                                 │
┌────▼──────────────────┐  ┌───▼──────────────────────┐         │
│  Finalize Transcript  │  │  Finalize Transcript      │         │
│  myCompletionBuffer   │  │  theirCompletionBuffer    │         │
└────┬──────────────────┘  └───┬──────────────────────┘         │
     │                         │                                 │
     │◄────────────────────────┘                                 │
     │                                                            │
     │ Both finalized                                            │
     │                                                            │
┌────▼──────────────────────────────────────────────────────────┐
│   Save to Database (conversationRepository)                   │
│   {                                                            │
│     session_id,                                                │
│     speaker: 'me' or 'them',                                   │
│     text: finalizedTranscript,                                 │
│     timestamp                                                  │
│   }                                                            │
└────┬───────────────────────────────────────────────────────────┘
     │
     │ Saved
     │
┌────▼───────────────────────────────────────────────────────────┐
│   Trigger Callback: onTranscriptionComplete                    │
│   → Increment turn counter                                     │
│   → Potentially trigger Summary Analysis (if turn >= 5)        │
└────────────────────────────────────────────────────────────────┘
```

---

### Детальное описание компонентов

#### 1. Dual Stream Architecture

**Два независимых STT потока**:

1. **"my" Stream** - Микрофон пользователя
   - Захватывает речь пользователя
   - Speaker: `me`
   - Аудио источник: Default microphone

2. **"their" Stream** - Системный звук
   - Захватывает речь собеседника из системного аудио
   - Speaker: `them`
   - Аудио источник: BlackHole (macOS) / Virtual Audio Cable (Windows)

```javascript
constructor() {
    this.mySttSession = null;        // User microphone STT
    this.theirSttSession = null;     // System audio STT
    this.myCurrentUtterance = '';    // Current partial transcript (my)
    this.theirCurrentUtterance = ''; // Current partial transcript (their)
}
```

---

#### 2. STT Provider Support

**OpenAI Realtime API**:
```javascript
// Конфигурация в providers/openai.js
{
    type: 'transcription_session.update',
    session: {
        input_audio_format: 'pcm16',
        input_audio_transcription: {
            model: 'gpt-4o-mini-transcribe',
            prompt: customPrompt,    // Опционально
            language: 'en'
        },
        turn_detection: {
            type: 'server_vad',      // Voice Activity Detection
            threshold: 0.5,
            prefix_padding_ms: 200,
            silence_duration_ms: 100
        }
    }
}
```

**Google Gemini Live**:
```javascript
// Встроенная STT с мультимодальностью
{
    model: 'gemini-2.5-flash',
    config: {
        audioTranscription: true,
        language: 'en-US'
    }
}
```

**Deepgram**:
```javascript
// Специализированный STT сервис
{
    model: 'nova-3',
    encoding: 'linear16',
    sample_rate: 16000,
    channels: 1,
    language: 'en'
}
```

**Whisper (локальный)**:
```javascript
// Запускается как subprocess
const whisperProcess = spawn('whisper', [
    '--model', 'medium',  // tiny/base/small/medium
    '--language', 'en',
    '--output_format', 'json',
    '--'  // stdin
]);
```

---

#### 3. Keep-Alive & Session Renewal

**Проблема**: Многие STT провайдеры закрывают соединение после периода бездействия или имеют жесткий лимит времени сессии (30 минут).

**Решение**:

**Keep-Alive Timer** (60 секунд):
```javascript
this.keepAliveInterval = setInterval(() => {
    // Отправка пустого аудио пакета для поддержания соединения
    if (this.mySttSession) {
        this.mySttSession.sendKeepAlive();
    }
    if (this.theirSttSession) {
        this.theirSttSession.sendKeepAlive();
    }
}, KEEP_ALIVE_INTERVAL_MS); // 60 seconds
```

**Session Renewal Timer** (20 минут):
```javascript
this.sessionRenewTimeout = setTimeout(async () => {
    console.log('Renewing STT sessions to avoid 30min timeout...');

    // Создать новые сессии
    const newMySttSession = await createSTT(this.modelInfo);
    const newTheirSttSession = await createSTT(this.modelInfo);

    // 2-секундное перекрытие для плавного перехода
    setTimeout(() => {
        // Закрыть старые сессии
        this.mySttSession.close();
        this.theirSttSession.close();

        // Переключиться на новые
        this.mySttSession = newMySttSession;
        this.theirSttSession = newTheirSttSession;
    }, SOCKET_OVERLAP_MS);

    // Рекурсивно установить следующее обновление
    this.startSessionRenewal();
}, SESSION_RENEW_INTERVAL_MS); // 20 minutes
```

---

#### 4. Voice Activity Detection (VAD)

**Server-side VAD** (OpenAI):
- Провайдер определяет начало и конец речи
- Автоматически отправляет события `speech_started`, `speech_stopped`
- Не требует клиентской логики

**Client-side VAD** (Whisper/Deepgram):
```javascript
function detectSpeech(audioBuffer) {
    const volume = calculateRMS(audioBuffer);
    const threshold = 0.01; // Настраиваемый порог

    if (volume > threshold) {
        return 'speaking';
    } else {
        return 'silence';
    }
}
```

---

#### 5. Debouncing & Turn Completion

**Проблема**: Частые паузы в речи могут привести к преждевременному завершению utterance.

**Решение**: Debounce timer на 2 секунды

```javascript
handlePartialTranscript(speaker, text) {
    if (speaker === 'me') {
        this.myCompletionBuffer = text;

        // Clear existing timer
        clearTimeout(this.myCompletionTimer);

        // Set new 2-second timer
        this.myCompletionTimer = setTimeout(() => {
            this.finalizeTranscript('me', this.myCompletionBuffer);
        }, COMPLETION_DEBOUNCE_MS); // 2000ms
    } else {
        // Same for "their"
        this.theirCompletionBuffer = text;
        clearTimeout(this.theirCompletionTimer);
        this.theirCompletionTimer = setTimeout(() => {
            this.finalizeTranscript('them', this.theirCompletionBuffer);
        }, COMPLETION_DEBOUNCE_MS);
    }
}
```

**Результат**:
- Паузы < 2 секунд: продолжение того же utterance
- Паузы >= 2 секунд: завершение utterance, сохранение в БД

---

#### 6. Сохранение в БД

```javascript
async finalizeTranscript(speaker, text) {
    if (!text || text.trim().length === 0) {
        return; // Игнорировать пустые транскрипты
    }

    // Сохранить в БД
    await conversationRepository.create({
        session_id: this.currentSessionId,
        speaker: speaker,  // 'me' или 'them'
        text: text.trim(),
        timestamp: Date.now()
    });

    // Уведомить UI
    this.onStatusUpdate?.({
        speaker,
        text,
        status: 'finalized'
    });

    // Вызвать callback для триггера Summary
    this.onTranscriptionComplete?.({
        speaker,
        text
    });
}
```

---

#### 7. System Audio Capture

**macOS** (BlackHole):
```bash
# Установка
brew install blackhole-2ch

# Настройка в Audio MIDI Setup:
# 1. Create Multi-Output Device
# 2. Select: Built-in Output + BlackHole 2ch
# 3. Set as system default output
```

**Windows** (VB-Cable):
```powershell
# Установка VB-Audio Virtual Cable
# Настройка:
# 1. Set VB-Cable as default recording device
# 2. Configure app to output to VB-Cable
# 3. Listen to VB-Cable output on speakers
```

**Захват в коде**:
```javascript
const { spawn } = require('child_process');

// Запустить системный аудио захват
this.systemAudioProc = spawn('ffmpeg', [
    '-f', 'avfoundation',          // macOS
    '-i', ':BlackHole 2ch',        // Input device
    '-f', 's16le',                 // 16-bit PCM
    '-ar', '16000',                // 16kHz sample rate
    '-ac', '1',                    // Mono
    '-'                            // Output to stdout
]);

// Pipe аудио в STT session
this.systemAudioProc.stdout.on('data', (audioChunk) => {
    this.theirSttSession.sendAudio(audioChunk);
});
```

---

### Параметры конфигурации

| Параметр | Значение | Описание |
|----------|----------|----------|
| **Keep-Alive Interval** | 60 seconds | Частота keep-alive пакетов |
| **Session Renewal** | 20 minutes | Период пересоздания сессий |
| **Socket Overlap** | 2 seconds | Перекрытие при переключении сессий |
| **Completion Debounce** | 2 seconds | Задержка до финализации utterance |
| **VAD Threshold** | 0.5 | Порог определения речи (0.0-1.0) |
| **Sample Rate** | 16kHz | Частота дискретизации аудио |
| **Audio Format** | PCM16 | Формат аудио данных |

---

## 4. Conversation Flow Pipeline

**Назначение**: Общий поток управления разговором, интеграция всех компонентов (STT, Summary, Ask) в единую систему.

**Где реализован**: Распределено по `sttService.js`, `summaryService.js`, `askService.js`

---

### ASCII Диаграмма общего потока

```
┌─────────────────────────────────────────────────────────────────────┐
│                   COMPLETE CONVERSATION FLOW                         │
│           (Integration of STT → Summary → Ask)                       │
└─────────────────────────────────────────────────────────────────────┘

[User Starts Session]
     │
     ├─────────────────────────────────┐
     │                                 │
     ▼                                 ▼
┌─────────────────┐          ┌─────────────────────┐
│  STT Pipeline   │          │  UI Initialization  │
│  (Continuous)   │          │  - Create windows   │
└────┬────────────┘          │  - Setup IPC        │
     │                       └─────────────────────┘
     │ Real-time transcription
     │
     ├──────────────┬──────────────┐
     │              │              │
     ▼              ▼              ▼
  Speaker: me   Speaker: them   Silence
     │              │              │
     │              │              │
┌────▼──────────────▼──────────────▼─────┐
│   Partial Transcripts Display          │
│   - Live updating in UI                │
│   - "me: [typing...]"                  │
│   - "them: [typing...]"                │
└────┬───────────────────────────────────┘
     │
     │ 2 seconds of silence
     │
┌────▼───────────────────────────────────┐
│   Finalize Utterance                   │
│   - Save to database                   │
│   - Mark as complete in UI             │
└────┬───────────────────────────────────┘
     │
     │ Increment turn counter
     │
     ▼
┌─────────────────┐
│  Turn Counter   │
│  turnCount++    │
└────┬────────────┘
     │
     │ Check if turnCount >= 5
     │
     ├─────────────┬──────────────────┐
     │             │                  │
     │ Yes         │ No               │
     │             │                  │
     ▼             ▼                  │
┌─────────────────┐                  │
│  Summary        │      Continue    │
│  Pipeline       │      listening   │
│  (Auto)         │                  │
└────┬────────────┘                  │
     │                               │
     │ AI Analysis                   │
     │                               │
┌────▼────────────────────────┐      │
│  Display Summary            │      │
│  - Summary Overview         │      │
│  - Key Topics               │      │
│  - Suggested Questions      │      │
└────┬────────────────────────┘      │
     │                               │
     │◄──────────────────────────────┘
     │
     │ Continue conversation
     │
     ├────────────────────────────────┐
     │                                │
     │ Meanwhile...                   │
     │                                │
     ▼                                ▼
[User presses Ask]         [Conversation continues]
     │                                │
     ▼                                │
┌─────────────────────┐               │
│  Ask Pipeline       │               │
│  (On-Demand)        │               │
└────┬────────────────┘               │
     │                                │
     │ Capture screenshot             │
     │ + User prompt                  │
     │                                │
┌────▼────────────────────────┐       │
│  AI Response (Multimodal)   │       │
│  - Text + Image analysis    │       │
└────┬────────────────────────┘       │
     │                                │
┌────▼────────────────────────┐       │
│  Display in Ask Window      │       │
│  - Formatted markdown       │       │
│  - Code highlighting        │       │
└─────────────────────────────┘       │
                                      │
                 ┌────────────────────┘
                 │
                 ▼
          [Loop continues...]
```

---

### Поток данных между компонентами

#### 1. STT → Database → Summary

```javascript
// STT finalizes transcript
sttService.finalizeTranscript('me', 'Hello, can you tell me about your experience?');
    │
    ├─> conversationRepository.create({...})
    │       │
    │       └─> SQLite: INSERT INTO conversation (...)
    │
    └─> onTranscriptionComplete callback
            │
            └─> summaryService.increment turnCount
                    │
                    └─> if (turnCount >= 5) → trigger analysis
```

#### 2. User → Ask → Response

```javascript
// User presses Ask button
userAction: Cmd+Shift+K
    │
    ├─> askService.showAskDialog()
    │       │
    │       └─> User enters prompt
    │               │
    │               └─> askService.sendQuestion(prompt)
    │                       │
    │                       ├─> Capture screenshot
    │                       ├─> Get conversation history
    │                       ├─> Build multimodal prompt
    │                       └─> Stream AI response
    │                               │
    │                               └─> Display in Ask window
```

#### 3. Summary → UI → User Action

```javascript
// Summary analysis complete
summaryService.analysisComplete(response)
    │
    ├─> Save to conversation_analysis table
    │
    └─> Send to UI via IPC
            │
            └─> UI displays:
                    - Summary Overview
                    - Key Topics
                    - Suggested Questions (clickable)
                            │
                            └─> User clicks question
                                    │
                                    └─> Auto-populate Ask dialog
```

---

### Состояния сессии

```javascript
// Session State Machine
const SessionState = {
    IDLE: 'idle',                    // Нет активной сессии
    INITIALIZING: 'initializing',    // Создание STT сессий
    LISTENING: 'listening',          // Активная транскрипция
    ANALYZING: 'analyzing',          // Summary в процессе
    ASKING: 'asking',                // Ask запрос в процессе
    ERROR: 'error'                   // Ошибка
};

// Transitions
IDLE → INITIALIZING → LISTENING
        ↓                ↓
      ERROR          ANALYZING
                        ↓
                    LISTENING
                        ↓
                     ASKING
                        ↓
                    LISTENING
```

---

### IPC Events (Inter-Process Communication)

**Main Process → Renderer**:
- `transcription-partial`: Частичный транскрипт
- `transcription-complete`: Финализированный транскрипт
- `analysis-chunk`: Частичный Summary анализ
- `analysis-complete`: Полный Summary анализ
- `ask-response-chunk`: Частичный Ask ответ
- `ask-response-complete`: Полный Ask ответ

**Renderer → Main Process**:
- `start-session`: Начать новую сессию
- `stop-session`: Остановить сессию
- `ask-question`: Отправить Ask запрос
- `update-model-config`: Обновить конфигурацию модели

---

### Оптимизация всего потока

**Параллельная обработка**:
- STT работает непрерывно
- Summary триггерится асинхронно (не блокирует STT)
- Ask выполняется в отдельном окне (не блокирует основной поток)

**Кэширование**:
- Системные промпты кэшируются для сессии
- История разговора загружается инкрементально
- Скриншоты кэшируются для follow-up Ask запросов

**Дебаунсинг**:
- STT utterance completion: 2 секунды
- Summary не триггерится если предыдущий еще выполняется
- Ask запросы queued если множественные

---

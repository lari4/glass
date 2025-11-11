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

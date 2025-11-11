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

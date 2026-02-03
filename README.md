# QuestionCard — устойчивый UI (Next.js 14 / React 18 / TS)

Тестовое задание: **QuestionCard** для экзаменационной платформы.  
Контент вопроса приходит как **TipTap JSON**, формулы рендерятся через **KaTeX**.  
Пользователь кликает быстро, API иногда возвращает ошибки → UI должен быть устойчивым.

---

## Цели компонента

- Быстро отображать stem + варианты ответов
- Разрешать выбор ответа
- Проверять ответ через API
- Показать explanation **только после check**
- Устойчиво переживать:
  - быстрые клики
  - двойные запросы
  - смену вопроса в процессе запроса
  - падение KaTeX / ошибки рендера
  - demo режим с paywall

---

## Стек (для MVP, веб → далее RN)

- **Next.js 14+ / React 18**
- **TypeScript**
- **Tailwind + shadcn/ui** (компоненты)
- **Zustand** (глобальное состояние)
- **axios** (REST)
- (опционально) **SWR / React Query** для кэша вопроса, если API нестабилен

---

## Архитектура: UI vs Domain state

### Почему разделяем state
`QuestionCard` должен быть **быстрым и предсказуемым**, а прогресс экзамена — **долгоживущим**.

- **UI state (локальный)** — "как сейчас отображается карточка"
- **Domain state (глобальный)** — "что пользователь реально сделал в сессии"

Это важно, потому что:
- локальный state должен сбрасываться при новом `questionId`
- глобальный state должен переживать навигацию/обновление страницы (в будущем)

---

## Структура компонентов

> Важно: `QuestionCard` — контейнер с логикой, вложенные компоненты максимально "dumb".

```
QuestionCard
├── QuestionHeader
│   ├── QuestionMeta
│   └── QuestionControls
│       ├── MarkForReviewButton
│       └── ReportIssueButton (optional)
│
├── QuestionStem
│   ├── TipTapRenderer (JSON → HTML)
│   └── MathRenderer (KaTeX inline/block)
│
├── AnswerOptions
│   └── AnswerOptionItem (x N)
│
├── ActionBar
│   ├── CheckAnswerButton
│   ├── NextButton / PrevButton (optional)
│   └── RequestStateIndicator
│
└── ExplanationSection (conditional)
    ├── ExplanationResultBanner (Correct/Incorrect)
    ├── ExplanationContent (TipTap + KaTeX)
    └── DemoPaywallOverlay (demo only)
```

---

## Где хранится состояние

### Локально в `QuestionCard` (UI state)
- `selectedAnswerId: string | null`
- `isChecked: boolean`
- `checkStatus: "idle" | "loading" | "success" | "error"`
- `checkResult: { isCorrect; correctAnswerId; explanationJson? } | null`
- `errorText: string | null`

✅ Локально — потому что:
- это состояние отображения **одного вопроса**
- оно должно сбрасываться при смене `questionId`
- оно не должно "протекать" в другие экраны

---

### Глобально (Zustand / domain state)
- `role: guest | demo | subscriber | admin`
- `answersByQuestionId: Record<string, string>` (выбранные ответы)
- `markedForReview: Set<questionId>`
- `attempt/session meta` (при наличии exam режима)

✅ Глобально — потому что:
- ответ надо помнить при переходе назад/вперед
- ответы нужны для результатов и аналитики
- логика затем переиспользуется в RN

---

## Что сбрасывается при смене questionId

При `questionId` change:

- `isChecked = false`
- `checkStatus = "idle"`
- `checkResult = null`
- `errorText = null`
- отменяем in-flight request (AbortController) или игнорируем устаревший ответ (requestId)

### Важно про selectedAnswerId
- если продукт "помнит выбор":
  - `selectedAnswerId` берём из `answersByQuestionId[questionId]`
- если продукт "не помнит выбор":
  - `selectedAnswerId = null`

---

## Конкурентность и быстрые клики

### Проблема
Если пользователь кликает быстро:

1) **Double submit**  
`checkAnswer()` отправился 2 раза → ответы приходят в другом порядке → UI ломается.

2) **Смена ответа во время check**  
Пользователь выбрал B, нажал check, успел выбрать C → сервер вернул результат для B, UI показывает для C.

3) **Смена вопроса во время check**  
Пользователь ушёл на следующий вопрос → прилетает ответ от прошлого → мутируем state не того вопроса.

---

## Решение: requestId + snapshot + disable

### 1) Disable на время check
- `CheckAnswerButton.disabled = loading`
- `AnswerOptions.disabled = loading`
- (опционально) `AnswerOptions.disabled = isChecked` в строгом режиме

Это банально, но это **самая дешёвая защита** от 80% гонок.

### 2) Snapshot выбранного ответа
В момент запроса фиксируем:

- `checkedAnswerId = selectedAnswerId`

> Редкая ошибка в UI: читать `selectedAnswerId` после await, когда пользователь уже поменял выбор.

### 3) requestId (soft-cancel)
Даже если API/axios нельзя отменить — можно **не принимать устаревший ответ**.

---

## Псевдокод логики

### local state
```ts
selectedAnswerId = null
isChecked = false
checkStatus = "idle"
checkResult = null
errorText = null

lastRequestId = 0
```

### derived state
```ts
canSelectAnswer = checkStatus !== "loading" && !isChecked
canCheck = selectedAnswerId != null && checkStatus !== "loading" && !isChecked
showExplanation = isChecked && checkStatus === "success"
isDemo = role === "demo"
```

### onQuestionChange(newId)
```ts
onQuestionChange(newId):
  cancelInFlightIfAny() // AbortController or ignore via requestId

  selectedAnswerId = answersByQuestionId[newId] ?? null

  isChecked = false
  checkStatus = "idle"
  checkResult = null
  errorText = null
```

### onSelectAnswer(answerId)
```ts
onSelectAnswer(answerId):
  if (!canSelectAnswer) return

  selectedAnswerId = answerId
  store.saveAnswer(questionId, answerId)
```

### onCheckAnswer()
```ts
onCheckAnswer():
  if (!canCheck) return

  isChecked = true
  checkStatus = "loading"
  errorText = null

  checkedAnswerId = selectedAnswerId // snapshot
  requestId = ++lastRequestId

  res = api.checkAnswer(questionId, checkedAnswerId)

  if (requestId !== lastRequestId):
    return // stale response

  if res.ok:
    checkStatus = "success"
    checkResult = {
      isCorrect: res.isCorrect,
      correctAnswerId: res.correctAnswerId,
      explanationJson: res.explanationJson ?? question.explanationJson ?? null
    }
  else:
    // важно: UI не должен "залипать" в checked=true при ошибке запроса
    isChecked = false
    checkStatus = "error"
    checkResult = null
    errorText = "Не удалось проверить ответ. Попробуйте ещё раз."
```

---

## UI правила (disabled/conditional)

### Check button
disabled если:
- нет selectedAnswerId
- loading
- уже isChecked === true

### Explanation
- нельзя показывать до check: `showExplanation = isChecked && status === success`
- если нет explanation: текст "Объяснение недоступно для этого вопроса"
- demo режим:
  - explanation скрыт/blur
  - текст "Доступно по подписке"
  - CTA "Оформить подписку"

---

## Edge cases / UX поведение

**1) Explanation отсутствует**  
Показываем: Correct/Incorrect (результат) + "Объяснение недоступно".  
Не показываем пустую область — это выглядит как баг.

**2) Stem состоит только из формул**  
KaTeX renderer поддерживает inline + block, корректные отступы/line-height. Не должно быть "пустого текста" над/под формулой.

**3) Очень длинный stem**  
Нормальный max width + переносы. ActionBar можно сделать sticky (если это тренажёр). На мобиле — удобные кликабельные варианты.

**4) KaTeX упал с ошибкой**  
Fail-soft стратегия: карточка не ломается целиком. Вместо формулы: "Не удалось отрендерить формулу", показываем исходный latex как текст.  
Редкий, но важный UX момент: падение KaTeX НЕ должно блокировать ответы. Пользователь всё равно может решить задачу по тексту или интуитивно.

**5) Пользователь меняет ответ после check**  
Два режима:
- **A: Экзамен (strict)** — после check варианты disabled, "Ответ зафиксирован"
- **B: Обучение (learning)** — при смене ответа: сбрасываем isChecked=false, скрываем explanation, снова доступен check  

Для MVP можно начать с режима A, а затем добавить B в тренировочном режиме.

**6) Demo режим**  
После check: результат показываем. Explanation: blur overlay, текст "Объяснение доступно по подписке", CTA кнопка оплаты.

---

## Нюансы реализации в Next.js 14

**1) SSR + KaTeX / TipTap**  
TipTap рендер и KaTeX могут зависеть от DOM. Правильный подход: рендерить "content" безопасно (или через client-only компонент), избегать hydration mismatch.  
Практика: QuestionStem можно сделать `use client` и держать рендер полностью на клиенте.

**2) Не хранить "всё подряд" в Zustand**  
Zustand — не свалка UI state. Храним в глобальном: ответы пользователя, роли, прогресс. А transient UI state — локально.

**3) API "косячит" → ошибки должны быть конечным состоянием**  
Если check упал: кнопки снова активны, сообщение понятно, можно retry без перезагрузки.

---

## Мини-итог

QuestionCard устойчив к:
- быстрым кликам
- двойным check
- смене вопроса во время запроса
- ошибкам API
- падениям KaTeX

И готов для масштабирования:
- тренировки
- mock exam
- демо/paywall
- переиспользование логики в RN

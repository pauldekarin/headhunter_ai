---
tags: [stage/1]
status: done
---

# Stage 1 — MVP

**Цель**: пользователь может запустить приложение, авторизоваться в hh.ru, получить список вакансий по фильтру, сгенерировать письма через внутренний **[[AI Layer]]** (LiteLLM, multi-provider + fallback), и отправить отклики вручную с подтверждением.

> [!summary] Текущий прогресс (см. таски ниже): **1.1–1.12 — done**. Stage 1 функционально закрыт: AI-таб в Settings с редактором `llm_deployments` (shadcn Accordion + `@dnd-kit/svelte` reorder) + полями `resume_text` / `letter_style` / `system_prompt`. Backend invalidate-hook на PUT /settings подключён (`api/routes/settings.py:24` вызывает `ai_layer.rebuild(...)`) — Router пересобирается без рестарта. Дальше — MVP-демо.

> [!note] AI в MVP — внутренний [[AI Layer]] поверх LiteLLM с мульти-провайдером и fallback-цепочкой. MCP-канал для внешних AI-клиентов (Claude Desktop, Claude Code) перенесён в [[Stage 2 - Расширение]].

## Задачи

### 1.1 BrowserCore с персистентным профилем — `M` — **done**

Запуск Playwright Chromium с user-data-dir в `~/.headhunter_ai/chrome-profile/`. Stealth-патчи. См. [[Chromium]].

**AC**: Логин в hh.ru сохраняется между запусками; `navigator.webdriver === undefined`.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.2 Auth-flow — `M` — **done**

Проверить логин по cookie `hhrole`, если не залогинен — открыть видимый Chromium на странице логина hh.ru и ждать, пока пользователь залогинится. См. [[Chromium]].

**AC**: [[UI]] отображает иконку авторизации в шапке (3 состояния: anonymous, waiting, authenticated). Клик по иконке в состоянии anonymous запускает логин. Backend открывает hh.ru-страницу логина в видимом Chromium с автоматическим bring-to-front. После успешного логина иконка обновляется реактивно через WS-событие.

**Note**: Авторизация **опциональна** для парсинга и генерации писем — обязательна только для отправки откликов (см. 1.8 Writer, 1.10 LetterReview UI).

**Зависимости**: 1.1, 1.4.

---

### 1.3 Parser module — `L` — **done**

По SearchFilter перейти на страницу выдачи, извлечь все 10 полей вакансии для каждой карточки + детали со страницы вакансии. См. [[Parser service]].

**AC**: Возвращает 50 вакансий из тестового запроса; селекторы вынесены в отдельный модуль `browser/selectors.py`.

**Зависимости**: 1.1.

---

### 1.4 FastAPI backend skeleton — `M` — **done**

REST endpoints под префиксом `/api/v1`: `POST /vacancies/search`, `GET /vacancies/`, `POST /vacancies/{vacancy_id}/cover_letter`, `POST /vacancies/{vacancy_id}/submit`. WebSocket `/ws/vacancies` для стрима. См. [[REST]].

**AC**: OpenAPI spec доступен на `/docs`; WebSocket стримит mock-данные.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.5 SQLite + миграции — `S` — **done**

Таблицы: `vacancies`, `applications`, `prompts`, `audit_log`. Alembic init. См. [[Storage]].

**AC**: `alembic upgrade head` создаёт схему; CRUD для `vacancies` работает.

**Зависимости**: 1.4.

---

### 1.6 Orchestrator + очередь — `M` — **done**

State machine для каждой вакансии (см. [[Domain Model]]): `parsed → letter_pending → letter_ready → sent / error`. asyncio.Queue с персистенцией статусов в [[Storage|SQLite]].

**SearchService** управляет жизненным циклом поискового таска:
- Двухшаговый picker: `POST /api/v1/search/picker/new` открывает hh.ru в Chromium → пользователь руками выставляет фильтры → `POST /api/v1/search/picker/{id}/confirm` читает URL из вкладки и валидирует домен.
- Запуск поиска: `POST /api/v1/search/vacancies/new` с URL и опциональными `max_pages`/`max_vacancies` (дефолты из settings).
- Активный таск: `GET /api/v1/search/vacancies/current` (204 если нет), `DELETE /api/v1/search/vacancies/{id}` отменяет.
- Каждая вакансия привязана к запустившему её поиску через `Vacancy.search_id` (FK → `searches`). Очередь UI скоупится через `GET /api/v1/vacancies/?search_id=latest|all|<uuid>`.

История завершённых/прерванных поисков пишется в таблицу `searches` (см. [[Storage]]). REST-эндпоинт истории — отдельная задача (см. [[REST]]).

**AC**: При перезапуске бэка очередь восстанавливается из SQLite; активный таск доступен через `/search/vacancies/current`; UI показывает только вакансии текущего поиска по умолчанию.

**Зависимости**: 1.4, 1.5.

---

### 1.7 Backend API foundation — `M` — **done**

Реальные эндпоинты поверх каркаса 1.4: убрать моки, подключить SQLite и стейт-машину 1.6, добавить недостающие таблицы и API настроек.

- Таблицы `cover_letters` (версии писем по `application_id`) и `settings` (синглтон с `CHECK(id=1)`).
- Реальные `GET /api/v1/vacancies`, `GET /api/v1/vacancies/{id}`, WebSocket `/ws/vacancies` — из БД, не моки.
- Lifecycle-эндпоинты на стейт-машине: `POST /api/v1/vacancies/{id}/queue_for_letter`, `/cover_letter`, `/submit`, `/retry`, `GET /{id}/status`.
- `GET`/`PUT /api/v1/settings` поверх таблицы `settings` (get-or-create с дефолтами из pydantic).

См. [[REST]], [[Storage]], [[Domain Model]].

**AC**: pytest зелёный по эндпоинтам; UI может ходить в реальные `vacancies` и `settings` без моков.

**Зависимости**: 1.4, 1.5, 1.6.

---

### 1.8 Writer module — `L` — **done**

По vacancy_id и тексту письма: открыть страницу вакансии, нажать «Откликнуться», вставить письмо, submit, проверить успех (поиск success-фразы в DOM с NFKC-нормализацией + латин-кириллица confusables). См. [[Writer service]].

Consumer-loop в `Orchestrator` пуллит заявки из очереди, проверяет auth + rate-limit ([[Anti-bot]]), дёргает Writer и пишет результат в БД + WS (`submission_event`, `captcha_event`). Детект капчи → `pause`; `POST /api/v1/orchestrator/resume` снимает паузу. Rate-limit использует таблицу `rate_limits` (token-bucket по sliding window из БД).

**AC**: pytest зелёный (consumer-loop по всем веткам, rate-limiter, pause/resume, resume-endpoint); E2E на тестовой вакансии — отклик отправлен, статус `letter_sent`, строка в `rate_limits`.

**Зависимости**: 1.1, 1.5, 1.6, 1.7.

---

### 1.9 UI: SearchForm + VacancyList — `M` — **done**

Поиск стартуется не формой полей, а **двухшаговым picker'ом**: пользователь жмёт «Новый поиск», бэк открывает hh.ru в Chromium, юзер вручную выставляет фильтры в открытой вкладке, возвращается в UI и подтверждает — backend читает URL вкладки. Лимиты (`max_pages`/`max_vacancies`) задаются в picker-форме поверх дефолтов из settings. Список вакансий идёт через TanStack Query (`vacanciesQueryKey`) + WS `/ws/events` для realtime-обновлений. См. [[UI]].

**AC**: ✅ Picker запускается из `routes/queue/+page.svelte`, вакансии появляются по мере парсинга через WS `vacancy_new` event, очередь скоупится в `?search_id=latest` по умолчанию.

**Зависимости**: 1.4, 1.6.

---

### 1.10 UI: LetterReview — `M` — **done**

Реализован как **Sheet-drawer** (`apps/desktop/src/lib/components/letter-review-sheet.svelte`), монтируется в `+layout.svelte`, открывается по клику на карточку из Queue через глобальный rune-store `lib/stores/letter_review.svelte.ts` (`letterReview.open(vacancyId)`).

**Внутри:**
- Три TanStack-query реактивно на `letterReview.vacancyId`: `applicationStatus`, `latestCoverLetter`, `coverLettersHistory` (см. `lib/queries/applications.ts`). Все три возвращают `null`/`[]` при 404 — UI трактует это как «application'а ещё нет».
- Tabs: «Письмо» (textarea always-editable, sync `localText` ← `latest.text` по версии) + «История» (список версий DESC, кнопка «Восстановить» с `AlertDialog`-confirm — грузит текст версии в draft, новая версия создаётся при «Сохранить»).
- Кнопки Footer'а ветвятся по `ProcessingState`: `parsed` → [Сгенерировать]; `letter_pending` → disabled; `letter_ready`/`letter_reviewing` → [Пропустить][Регенерация][Сохранить][Отправить]; `error` → [Пропустить][Повторить]; `letter_sending`/`letter_sent`/`skipped` → [Закрыть].
- Auto-save on close: если `isDirty && isEditable` — `POST /cover_letter` перед `letterReview.close()`.
- Generate-flow: если `applicationStatus.data === null` — сначала `POST /queue_for_letter` (создаёт application), потом `POST /ai/create_cover_letter/{id}` (blocking ~5–15с).
- Submit-flow: автосейв dirty-текста перед `POST /submit`.
- Vacancy для заголовка — snapshot из `queryClient.getQueryData(vacanciesQueryKey)` (Sheet открывается из Queue, кеш свежий).

**AC**: ✅ Письмо генерируется, редактируется, отправляется; статус обновляется через invalidate `applicationStatusQueryKey`/`latestCoverLetterQueryKey`/`coverLettersHistoryQueryKey` после каждой мутации.

**Известные ограничения** (можно закрыть отдельной задачей):
- WS-event `application_event` пока не обновляет TanStack-кеш через `applyApplicationStatus` — статус подтягивается только через invalidate после ручных мутаций. Для auto-режима (`AutoApplyService` меняет статус в фоне) нужен глобальный WS-handler.

Карточка вакансии с превью письма, кнопки «Редактировать», «Сгенерировать заново» (вызов `POST /api/v1/ai/create_cover_letter/{vacancy_id}`), «Отправить», «Пропустить».

**AC**: Письмо генерируется, редактируется и отправляется; статус обновляется в списке.

**Зависимости**: 1.6, 1.8, 1.9, 1.12.

---

### 1.11 UI: Settings — `S` — **done**

Реализовано: `routes/settings/+page.svelte` через **shadcn-svelte Tabs + Superforms + Zod v4 (`zod4` адаптер)** в SPA-режиме (`SPA: true`, `dataType: "json"`, `resetForm: false`). Источник истины — TanStack Query (`settingsQueryKey`), `$effect` синхронит `formData` ← `settings.data` (чтение `$formData` обёрнуто в `untrack` для разрыва loop'а с `formData.set`), после PUT — `setQueryData(saved)` без refetch.

Четыре таба: **search** (`max_pages`, `max_vacancies`), **user** (`auto_submit` switch), **limits** (`hourly_limit`, `daily_limit`, `min_delay_ms`, `delay_jitter_ms`), **AI** (см. ниже).

**AI tab** (`lib/components/settings-ai-tab.svelte`) — три textarea/input (`resume_text` rows=10, `letter_style` input, `system_prompt` rows=6) + редактор **LLM deployments**:
- shadcn `<Accordion type="multiple">` — каждый item раскрывается на `model` / `api_key` / `api_base`.
- Drag-and-drop reorder через `@dnd-kit/svelte` (v0.5+, Svelte 5 attachments) — обёртка `lib/components/sortable-deployment.svelte` инкапсулирует `createSortable({id, index, type})`; в parent'е `<DragDropProvider {onDragEnd}>` + ручной `arrayMove(source.index → target.index)` через `isSortableOperation` guard. `OptimisticSortingPlugin` отключён адаптером — визуальный reorder идёт через реактивный `{#each}` с keyed `deployment.id`, не через DOM-mutation.
- Per-deployment Eye/EyeOff toggle для `api_key` (password ↔ text), стейт keyed by `deployment.id`.
- Badge **Primary / Fallback N** по позиции в массиве (первый — primary, остальные — fallback chain).
- ID-preservation: `apiDeploymentToForm` генерит `crypto.randomUUID()`, `$effect` сохраняет existing ID по позиции при resync — Accordion-state не сбрасывается на save.
- Empty-state разрешён (`z.array(...).default([])` без `.min(1)`); AILayer ответит `no_deployments` через `/api/v1/ai/health`.

Skeleton при `isPending`, error-state с retry-кнопкой при `isError`. Все строки через Paraglide (`m.settings_*`).

**AC**: Выполнен — `PUT /api/v1/settings` работает, AILayer пересобирается через `ai_layer.rebuild(...)` в `api/routes/settings.py:24`; UI-редактор `llm_deployments` (add/remove/reorder + model/api_key/api_base) и поля `resume_text` / `letter_style` / `system_prompt` подключены.

**Зависимости**: 1.4, 1.12.

---

### 1.12 AI Layer — multi-provider + fallback — `L` — **done (модуль)**, **open gap по rebuild on settings change**

Backend-модуль под `services/backend/src/headhunter_backend/ai/` с собственным router `/api/v1/ai`. Унифицированный фасад над LLM-провайдерами через LiteLLM: мульти-провайдер (Anthropic / OpenAI / Ollama), fallback-цепочка через `litellm.Router`, retry с exponential backoff. См. [[AI Layer]].

**Эндпоинты**:
- `POST /api/v1/ai/create_cover_letter/{vacancy_id}` — синхронно генерирует письмо по vacancy_id из path. Берёт резюме, стиль и `system_prompt` из `settings`, дёргает `AILayer.generate_cover_letter`, возвращает `AICoverLetterResponseAPISchema` (text, model_used, prompt/completion/total tokens, was_fallback, cost_usd). 409 если AI слой не готов (нет deployments) или для вакансии нет открытой `application`. 404 если vacancy не найдена.
- `GET /api/v1/ai/health` — отдаёт статус AI-слоя (`AIHealthStatusAPISchema`): `healthy` / `unhealthy` / `no_deployments`. Под капотом — `ping`-промпт через primary deployment.

**Расширение `settings`** (отдаётся через существующий `PUT /api/v1/settings`):
- `llm_deployments: list[{model, api_key, api_base?}]` — упорядоченный список deployments, primary первый, остальные — fallback chain.
- `llm_system_prompt: str` — system-промпт, перекрывает / дополняет `letter_style`.

**Реализация** (фактическая):
- `AILayer.generate_cover_letter(vacancy_model, resume, style, system_prompt) -> AICoverLetterResult` поверх `litellm.Router`. Возвращаемый dataclass: `text`, `model_used`, `prompt_tokens`, `completion_tokens`, `total_tokens`, `was_fallback`, `cost_usd`.
- `AILayer.get_health_status() -> AILayerHealthStatus` — `ping`-промпт через primary; `HEALTHY` / `UNHEALTHY` / `NO_DEPLOYMENTS`.
- `AILayer.rebuild(deployments)` вызывается из route handler `api/routes/settings.py:24` после `update_settings` — Router пересобирается на каждый успешный PUT /settings. Plus `bootstrap_ai_layer` на startup (`api/app.py:42-52`) подтягивает deployments из БД при старте.
- `PromptBuilder` ([[Headhunter Backend Code Map#PromptBuilder|`ai/prompts.py`]]) формирует system + user message; default system-промпт — личный кириллический cover letter под ≈250 слов, без выдуманных квалификаций.
- `AutoApplyService` ([[Headhunter Backend Code Map#AutoApplyService|`orchestrator/apply_service.py`]]) подписан на `VacancyWSEvent` — при `auto_submit=true` создаёт Application, генерирует письмо через `AILayer`, переводит в `LETTER_SENDING` и кладёт в `Orchestrator.enqueue`.
- Hardcoded MVP-промпт (версионирование — Stage 2 [[Stage 2 - Расширение#2.2|2.2]]).

**AC**:
- pytest зелёный: `AILayer` (с моком `litellm.Router`), endpoints, fallback-логика при имитации падения primary.
- E2E на тестовой вакансии: настроены два deployments (Anthropic + OpenAI как fallback). При отзыве ключа primary письмо всё равно генерируется через fallback. Статус вакансии — `letter_ready`, запись в `cover_letters`.
- Settings UI (1.11) показывает список deployments с возможностью добавления / удаления / переупорядочения.

**Зависимости**: 1.6, 1.7.

**Стек**:

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| LLM proxy | **LiteLLM** | 100+ провайдеров, унифицированный API, Router из коробки | langchain (тяжёлый); прямые SDK (дублирование) |
| Router/fallback | `litellm.Router` | Встроено, без сторонних зависимостей | ручной retry поверх `litellm.completion` |
| Локальный LLM | **Ollama** | Опциональный provider, OpenAI-совместимый API | llama.cpp (low-level) |

---

## Стек этапа (детально)

См. [[Stack]] и каждую заметку компонента ([[Parser service]], [[REST]], [[MCP]], [[UI]], [[Storage]]).

## См. также
- [[Roadmap]]
- предыдущий: [[Stage 0 - Setup]]
- следующий: [[Stage 2 - Расширение]]
- [[Verification]] — как проверить MVP

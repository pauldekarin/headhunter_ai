---
tags: [stage/1]
status: planned
---

# Stage 1 — MVP

**Цель**: пользователь может запустить приложение, авторизоваться в hh.ru, получить список вакансий по фильтру, сгенерировать письма через **внешнего Claude Desktop по MCP**, и отправить отклики вручную с подтверждением.

> [!note] AI в MVP — только через [[MCP]] + Claude Desktop. [[AI Layer]] (LiteLLM) откладываем на [[Stage 2 - Расширение]].

## Задачи

### 1.1 BrowserCore с персистентным профилем — `M`

Запуск Playwright Chromium с user-data-dir в `~/.headhunter_ai/chrome-profile/`. Stealth-патчи. См. [[Chromium]].

**AC**: Логин в hh.ru сохраняется между запусками; `navigator.webdriver === undefined`.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.2 Auth-flow — `M`

Проверить логин по cookie `hhrole`, если не залогинен — открыть видимый Chromium на странице логина hh.ru и ждать, пока пользователь залогинится. См. [[Chromium]].

**AC**: [[UI]] отображает иконку авторизации в шапке (3 состояния: anonymous, waiting, authenticated). Клик по иконке в состоянии anonymous запускает логин. Backend открывает hh.ru-страницу логина в видимом Chromium с автоматическим bring-to-front. После успешного логина иконка обновляется реактивно через WS-событие.

**Note**: Авторизация **опциональна** для парсинга и генерации писем — обязательна только для отправки откликов (см. 1.8 Writer, 1.10 LetterReview UI).

**Зависимости**: 1.1, 1.4.

---

### 1.3 Parser module — `L`

По SearchFilter перейти на страницу выдачи, извлечь все 10 полей вакансии для каждой карточки + детали со страницы вакансии. См. [[Parser service]].

**AC**: Возвращает 50 вакансий из тестового запроса; селекторы вынесены в отдельный модуль `browser/selectors.py`.

**Зависимости**: 1.1.

---

### 1.4 FastAPI backend skeleton — `M`

REST endpoints: `POST /api/search`, `GET /api/vacancies`, `POST /api/write/{vacancy_id}`, `POST /api/submit/{vacancy_id}`. WebSocket `/ws/vacancies` для стрима. См. [[REST]].

**AC**: OpenAPI spec доступен на `/docs`; WebSocket стримит mock-данные.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.5 SQLite + миграции — `S`

Таблицы: `vacancies`, `applications`, `prompts`, `audit_log`. Alembic init. См. [[Storage]].

**AC**: `alembic upgrade head` создаёт схему; CRUD для `vacancies` работает.

**Зависимости**: 1.4.

---

### 1.6 Orchestrator + очередь — `M`

State machine для каждой вакансии (см. [[Domain Model]]): `parsed → letter_pending → letter_ready → sent / error`. asyncio.Queue с персистенцией статусов в [[Storage|SQLite]].

**AC**: При перезапуске бэка очередь восстанавливается из SQLite.

**Зависимости**: 1.4, 1.5.

---

### 1.7 MCP server — `L`

Экспонировать tools: `list_pending_vacancies`, `get_vacancy(id)`, `get_user_context()`, `submit_cover_letter(vacancy_id, text)`. Транспорт stdio + streamable HTTP. См. [[MCP]].

**AC**: Claude Desktop с конфигом видит tools; вызов tool пишет в DB.

**Зависимости**: 1.6.

---

### 1.8 Writer module — `L`

По vacancy_id и тексту письма: открыть страницу вакансии, нажать «Откликнуться», вставить письмо, submit, проверить успех. См. [[Writer service]].

**AC**: E2E на тестовой вакансии — отклик отправлен, статус сохранён.

**Зависимости**: 1.1, 1.5.

---

### 1.9 UI: SearchForm + VacancyList — `M`

Форма фильтра (text, salary, region), список вакансий с realtime-обновлением через WebSocket. См. [[UI]].

**AC**: Поиск запускается, вакансии появляются по мере парсинга.

**Зависимости**: 1.4.

---

### 1.10 UI: LetterReview — `M`

Карточка вакансии с превью письма, кнопки «Редактировать», «Отправить», «Пропустить».

**AC**: Письмо редактируется и отправляется; статус обновляется в списке.

**Зависимости**: 1.6, 1.8, 1.9.

---

### 1.11 UI: Settings — `S`

Контекст соискателя (резюме textarea/upload), стиль письма, лимиты ([[Anti-bot]]), путь к Chromium-профилю.

**AC**: Настройки сохраняются в YAML, применяются без рестарта.

**Зависимости**: 1.4.

---

### 1.12 Документация подключения Claude Desktop — `S`

README с готовым `claude_desktop_config.json` snippet. См. [[MCP]].

**AC**: Пользователь по README настраивает Claude Desktop за < 5 минут.

**Зависимости**: 1.7.

---

## Стек этапа (детально)

См. [[Stack]] и каждую заметку компонента ([[Parser service]], [[REST]], [[MCP]], [[UI]], [[Storage]]).

## См. также
- [[Roadmap]]
- предыдущий: [[Stage 0 - Setup]]
- следующий: [[Stage 2 - Расширение]]
- [[Verification]] — как проверить MVP

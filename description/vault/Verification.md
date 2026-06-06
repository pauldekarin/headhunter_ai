---
tags: [process]
status: planned
---

# Verification

Как проверить результаты этапов.

## После [[Stage 1 - MVP]]

1. **Сборка**: `pnpm tauri build` создаёт installer для текущей платформы без ошибок.
2. **Запуск**: запущенное приложение открывает [[UI]]; в фоне виден Python-процесс (sidecar).
3. **Auth flow**: нажатие «Подключить hh.ru» открывает [[Chromium]] с hh.ru; после ручного логина окно закрывается, статус в UI меняется на «Авторизован».
4. **Парсинг**: ввод фильтра (например «Python разработчик, Москва») и запуск возвращает ≥ 10 вакансий через WebSocket за < 30 сек ([[Parser service]]).
5. **AI Layer**: в Settings задан хотя бы один LLM-deployment (модель + ключ); `POST /api/v1/ai/health` отвечает успехом; `POST /api/v1/ai/cover_letter` для одной вакансии возвращает текст письма и сохраняет его в БД ([[AI Layer]]).
6. **Fallback**: в Settings заданы два deployments; отзыв ключа у primary → следующая генерация уходит на fallback без ошибки в UI.
7. **End-to-end**: AI Layer сгенерировал письмо → пользователь подтверждает в [[UI]] LetterReview → нажимает «Отправить» → отклик появляется в hh.ru-кабинете соискателя.
8. **Тесты**: `pnpm test` (Vitest) и `uv run pytest` зелёные; покрытие orchestrator > 80% (см. [[Testing]]).
9. **Lint/types**: `pnpm lint && uv run ruff check && uv run mypy && cargo clippy` — без ошибок.

## После [[Stage 2 - Расширение]]

1. **MCP**: добавление в `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) блока с нашим бинарём → Claude Desktop показывает наши tools ([[MCP]]); вызов `list_pending_vacancies` возвращает данные; письмо, поданное через `submit_cover_letter`, видно в [[UI]] LetterReview наравне с письмами из MVP AI Layer.
2. **Auto-submit**: включение режима — отклики уходят сами в пределах лимита ([[Anti-bot]]).
3. **Промпты**: создание второго шаблона, A/B-метрика «принято пользователем» сохраняется.
4. **Дедуп**: повторный поиск не возвращает вакансии в `applications`.
5. **Captcha**: триггерим капчу — UI показывает уведомление, после ручного решения очередь продолжается.
6. **Audit-log**: экспорт CSV за неделю выгружается, содержит все действия.
7. **E2E**: GitHub Actions проходит 3 e2e-сценария.
8. **Auto-update**: новый release триггерит уведомление в работающем приложении.

## После [[Stage 3 - Оптимизация]]

1. **Параллелизм**: 100 вакансий парсятся < 2 минут.
2. **Метрики**: `curl localhost:9090/metrics` возвращает все метрики из [[Observability]].
3. **Tracing**: spans видны в выбранном backend.
4. **i18n**: переключение RU↔EN мгновенно меняет интерфейс.
5. **Темы**: системная light/dark + override.
6. **Бэкапы**: `~/.headhunter_ai/backups/` содержит свежий снапшот.
7. **Плагин**: пример из доки загружается и работает.

## См. также
- [[Roadmap]]
- [[Testing]]

---
tags: [service]
status: planned
---

# UI

Десктопный фронтенд приложения. Обёртка над [[REST]] + WebSocket.

## Стек: Tauri 2 + SvelteKit (рекомендуется)

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Desktop shell | **Tauri 2** (Rust) | ~10MB бандл vs ~150MB Electron; меньше RAM; native menus; auto-updater | Electron (огромная экосистема, тяжёлый); Wails (Go, моложе); локальный веб-UI без shell (проще, но не desktop-app) |
| UI framework | **Svelte 5 + SvelteKit 2** (TS) | Меньше runtime, runes-реактивность, отличный DX в Tauri webview | React 19 (огромная экосистема, больше boilerplate); SolidJS (быстрее, меньше комьюнити) |
| Компоненты | **shadcn-svelte + Tailwind CSS 4** | Copy-paste, без vendor lock-in | Skeleton UI, daisyUI |
| i18n | **Paraglide JS** (inlang) | Codegen + типизированные ключи, tree-shaking, ICU plurals | typesafe-i18n (agnostic); svelte-i18n (runtime, без codegen) |
| Формы | **Superforms + Zod** | Лучшая для SvelteKit, server+client валидация | Felte |
| State | **Svelte runes + TanStack Query** | Локально — runes; сервер — TanStack Query (кеш, инвалидация, retry) | Zustand (если React); ручные stores |
| WebSocket | native API + svelte-store обёртка | Без зависимостей для localhost | socket.io (overkill) |

## Локализация (i18n)

Все user-facing строки UI идут через **Paraglide JS**:

- Словари: `apps/desktop/messages/{locale}.json` (стандарт inlang messageFormat).
- Codegen: `apps/desktop/src/lib/paraglide/` (генерируется через Vite-плагин при сборке).
- Использование: `import * as m from "$lib/paraglide/messages"; m.queue_title()` — ключ становится типизированной функцией; плюрализация через ICU (`{count, plural, one {…} few {…} many {…} other {…}}`).
- Текущий baseLocale: `ru`. Добавление новых языков — отредактировать `project.inlang/settings.json` и положить `messages/<locale>.json`.
- Скоуп MVP (Stage 1.6) — только `routes/queue/`. Auth/Profile/Settings экранируются в Stage 1.11+.

## Альтернатива 1: Electron

| Плюсы | Минусы |
|---|---|
| Огромная экосистема | ~150MB бандл |
| Легко интегрировать любые Node-библиотеки | Больше RAM |
| Зрелые auto-update решения (Sparkle, Squirrel) | Безопасность хуже из коробки |

Если в команде сильнее Node, чем Rust — стоит рассмотреть.

## Альтернатива 2: Локальный веб-UI

Бэк поднимает UI на `localhost:3000`, пользователь открывает в Chrome. Проще всего в разработке, но это **не desktop-app** в прямом смысле (нет иконки, native menus, system-tray, нет хорошего onboarding).

## Экраны (MVP)

1. **Onboarding** — дисклеймер ToS, выбор Chromium-профиля
2. **Settings** — резюме, контекст, стиль письма, лимиты, список LLM-deployments (provider+model+key+api_base, primary первый, остальные — fallback chain), system-промпт
3. **Search** — форма фильтра + кнопка запуска
4. **Queue** — список вакансий с realtime-обновлением через WebSocket; статусы из [[Domain Model]]
5. **LetterReview** — превью письма, edit, submit/skip
6. **History** — отправленные отклики с фильтрами и экспортом

## Связи
- [[REST]] — основной API-клиент
- [[Architecture]] — общая картина
- [[Stack]] — общий стек

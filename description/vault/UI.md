---
tags: [service]
status: live
---

# UI

Десктопный фронтенд приложения. Обёртка над [[REST]] + WebSocket.

## Стек (фактический)

| Задача | Библиотека | Почему |
|---|---|---|
| Desktop shell | **Tauri 2** (Rust) | ~10MB бандл vs ~150MB Electron; native menus; auto-updater |
| UI framework | **Svelte 5 (runes) + SvelteKit 2** (TS) | Меньше runtime, отличный DX в Tauri webview |
| Adapter | **`@sveltejs/adapter-static`** | SPA, без SSR; webview грузит из FS |
| Компоненты | **shadcn-svelte** (`src/lib/components/ui/*`) поверх `bits-ui` + `tailwind-variants` | Copy-paste, без vendor lock-in |
| Стили | **Tailwind CSS 4** + `tw-animate-css` + `tailwind-merge` | Утилитарный, без рантайма |
| Иконки | **`@lucide/svelte`** | Svelte 5 биндинги, передаём как component reference |
| Toast | **`svelte-sonner`** + shadcn-обёртка `<Toaster />` | `toast.success/error` через Paraglide-сообщения |
| Light/Dark | **`mode-watcher`** | Системная тема |
| i18n | **`@inlang/paraglide-js`** + `@inlang/cli` | Codegen в `src/lib/paraglide/`; baseLocale `ru` |
| Server cache | **`@tanstack/svelte-query`** | Источник истины для `settings`, `vacancies`, `search`. После PUT — `setQueryData(saved)` напрямую, без refetch |
| Формы | **`sveltekit-superforms` + `formsnap`** + **`zod` v4** через `zod4` адаптер | SPA-режим, `dataType: "json"`, `resetForm: false`, `$effect` sync `formData ← settings.data` |
| State | **Svelte runes** (`$state`, `$effect`) | Локально-компонентное. Глобально — `$lib/stores/*.svelte.ts` (`auth`, `search_picker`) |
| WebSocket | native `WebSocket` API + handler в `$lib/api/events.ts` | Без зависимостей |
| Шрифты | **`@fontsource-variable/jetbrains-mono`** | Bundled, оффлайн |

## Структура

```
apps/desktop/
├── src/
│   ├── routes/
│   │   ├── +layout.svelte         ← Sidebar + Inset + Toaster (Sidebar.Provider)
│   │   ├── +layout.ts             ← loadConsent() + i18n
│   │   ├── +page.svelte           ← redirect/landing
│   │   ├── onboarding/            ← ToS-disclaimer (Stage 0)
│   │   ├── queue/+page.svelte     ← 1.9 — picker + scoped vacancies list
│   │   └── settings/+page.svelte  ← 1.11 — Tabs (search/user/limits) + Superforms
│   ├── lib/
│   │   ├── api/                   ← client.ts (fetch), types.ts (Settings/Vacancy/...), events.ts (WS), error.ts
│   │   ├── queries/               ← TanStack Query factories: settings.ts, vacancies.ts, search.ts
│   │   ├── stores/                ← auth.svelte.ts, search_picker.svelte.ts (runes-based)
│   │   ├── schemas/               ← Zod schemas (settings.ts — для Superforms)
│   │   ├── components/ui/         ← shadcn-svelte (button, input, tabs, form, switch, skeleton, sidebar, dialog, ...)
│   │   └── paraglide/             ← codegen (commit-ignored)
│   └── messages/{locale}.json     ← inlang dictionaries (ru.json)
└── src-tauri/                     ← Rust shell
```

## Sidebar + layout

Реализован на уровне `+layout.svelte` через shadcn `Sidebar` primitive (`Sidebar.Provider` оборачивает всё, `Sidebar.Root` + `Sidebar.Inset`). Навигация — data-driven массив `items: { title, href, icon }`, где `title` — paraglide function, `icon` — component reference. Активный пункт — `isActive={page.url.pathname === item.href}`. ProfileButton — в `Sidebar.Footer`. Sidebar.Trigger — в sticky-баре внутри `Sidebar.Inset`.

> [!note] `Sidebar.Inset` рендерится как `<main>` — поэтому страницы (например `routes/queue/+page.svelte`) **не должны** иметь свой `<main>`.

## Локализация (i18n)

Все user-facing строки UI идут через **Paraglide JS**:

- Словари: `apps/desktop/messages/{locale}.json` (стандарт inlang messageFormat).
- Codegen: `apps/desktop/src/lib/paraglide/` (генерируется через Vite-плагин при сборке; git-ignored).
- Использование: `import * as m from "$lib/paraglide/messages"; m.queue_title()` — ключ становится типизированной функцией; плюрализация через ICU (`{count, plural, one {…} few {…} many {…} other {…}}`).
- Текущий baseLocale: `ru`. Добавление новых языков — отредактировать `project.inlang/settings.json` и положить `messages/<locale>.json`.
- Покрытие на текущий момент: `nav_*`, `queue_*`, `picker_*`, `dialog_replace_*`, `toast_*`, `settings_*`, `status_*`. История по веткам растёт по мере добавления экранов.

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

| Экран | Статус | Файл | Содержимое |
|---|---|---|---|
| **Onboarding** | done | `routes/onboarding/` | Дисклеймер ToS, согласие сохраняется в `consent.json` |
| **Queue** | done (1.9) | `routes/queue/+page.svelte` | Picker (`searchPicker` store) + список вакансий (TanStack Query `vacanciesQueryKey`) + WS `vacancy_new`/`search_event`. Скоуп `?search_id=latest` |
| **Settings** | partial (1.11) | `routes/settings/+page.svelte` | Tabs (search/user/limits) на shadcn `Tabs.Root`, Superforms + Zod v4, Skeleton при `isPending`, error-state с retry. LLM-секция **passthrough** — `defaultLlm` + cached `llm` подкладывается в PUT |
| **LetterReview** | done (1.10) | `lib/components/letter-review-sheet.svelte` + `lib/stores/letter_review.svelte.ts` | Sheet-drawer справа. Открывается из Queue по `letterReview.open(vacancyId)`. Tabs: «Письмо» (textarea always-editable, sync по `version`) + «История» (DESC, restore через `AlertDialog`). Кнопки по `ProcessingState`. Auto-save on close если dirty. Generate: `POST /queue_for_letter` (если нет app'а) → `POST /ai/create_cover_letter/{id}` |
| **History** | planned | — | Отправленные отклики с фильтрами и экспортом (Stage 2) |

## Связи
- [[REST]] — основной API-клиент
- [[Architecture]] — общая картина
- [[Stack]] — общий стек

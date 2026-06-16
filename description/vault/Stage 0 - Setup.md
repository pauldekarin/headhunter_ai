---
tags: [stage/0]
status: done
---

# Stage 0 — Setup

Фундамент: монорепа, tooling, CI, дисклеймер. **Этап закрыт.**

## Задачи

### 0.1 Монорепа и tooling — `S`

Создать репу с workspace:
- `apps/desktop/` — Tauri + Svelte
- `services/backend/` — Python

Тулчейн: **pnpm workspaces** для JS, **uv** для Python.

**AC**: `pnpm install && uv sync` работает; pre-commit (ruff, biome, clippy) на всех файлах.

**Зависимости**: —

**Стек**:

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Workspace JS | **pnpm workspaces** | Быстрее npm/yarn, экономит диск через CAS | npm workspaces, bun |
| Workspace Python | **uv** | В 10-100× быстрее pip/poetry, lock-файл из коробки | poetry, rye |
| Pre-commit | **pre-commit** | Стандарт | lefthook (быстрее) |
| Tauri scaffolding | **`create-tauri-app`** | Официальный | ручная сборка |

---

### 0.2 CI baseline — `S`

GitHub Actions: lint + типы + unit-тесты для всех трёх стеков.

**AC**: PR-status check проходит на пустом PR.

**Зависимости**: 0.1.

---

### 0.3 UI baseline (Tailwind 4 + shadcn-svelte) — `S`

Подключить базовый UI-стек, на котором дальше будут собираться все экраны ([[UI]]).

**AC**: `pnpm --filter desktop run build` зелёный; в `apps/desktop/src/lib/components/ui/` лежат сгенерированные shadcn-компоненты; `app.css` подключён через корневой `+layout.svelte`; Tailwind-классы реально применяются (визуально проверено в `tauri dev`).

**Зависимости**: 0.1.

**Стек**:

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| CSS framework | **Tailwind CSS 4** | Утилитарный, нулевой runtime, Vite-плагин без PostCSS | UnoCSS, vanilla CSS modules |
| Vite-интеграция | **`@tailwindcss/vite`** | Официальный плагин Tailwind 4, Lightning CSS внутри | PostCSS (deprecated в v4) |
| Компоненты | **shadcn-svelte** (CLI) | Компоненты копируются в репо, нет vendor lock-in, full ownership | Skeleton UI, daisyUI |
| Примитивы a11y | **bits-ui** (через shadcn) | Headless-примитивы под Svelte 5 | melt-ui (предшественник) |

---

### 0.4 Onboarding-модал с дисклеймером — `S`

Первый запуск показывает условия использования + ToS warning. См. [[Anti-bot]].

**AC**: модал виден на первом запуске; после Accept согласие сохраняется в `~/.headhunter_ai/consent.json` ([[Storage]]); при повторном запуске модал не показывается.

**Зависимости**: 0.3.

---

## См. также
- [[Roadmap]]
- следующий: [[Stage 1 - MVP]]

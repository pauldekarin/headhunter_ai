---
tags: [stage/0]
status: planned
---

# Stage 0 — Setup

Фундамент: монорепа, tooling, CI, дисклеймер.

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

### 0.3 Onboarding-модал с дисклеймером — `S`

Первый запуск показывает условия использования + ToS warning. См. [[Anti-bot]].

**AC**: модал виден; согласие сохраняется в `~/.headhunter_ai/consent.json` ([[Storage]]).

**Зависимости**: 0.1.

---

## См. также
- [[Roadmap]]
- следующий: [[Stage 1 - MVP]]

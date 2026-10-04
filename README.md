# Blueprint: Детерминированный пайплайн spec-driven разработки с AI-агентами

**Версия:** 1.0
**Статус:** Draft / Ready for Review
**Область:** OpenSpec + adr-kit + agnostic-ai
**Формат:** Blueprint-as-Code (markdown, версионируется в репозитории)

---

## 1. Overview / Goals

Единый детерминированный пайплайн для AI-агентной разработки, в котором:

- **Инструкции** доставляются агенту через портативный источник (`.agnostic-ai/`) и синхронизируются в нативные файлы (`.cursor/rules/`, `AGENTS.md`, `CLAUDE.md`).
- **Процесс** управляется через OpenSpec с кастомной схемой, расширенной шагами `adr` и `api-contract`.
- **Контроль качества** обеспечивается через `adr-kit` (MCP), который работает как gatekeeper, а не как post-factum документация.
- **Целостность артефактов** защищена через каскадную хэш-инвалидацию зависимостей.

**Цель:** исключить дрейф инструкций между инструментами, исключить «размазывание» архитектурных решений по спецификациям и обеспечить воспроизводимость пайплайна от `propose` до `archive`.

---

## 2. Context and Scope

### Входит в blueprint

- Иерархия инструкций (agnostic-ai → нативные файлы).
- OpenSpec воркфлоу с кастомной схемой `proposal-to-apply`.
- Интеграция `adr-kit` как MCP-gateway.
- Хэш-инвалидация артефактов.
- Привязка MCP-инструментов к фазам.

### Не входит

- Конкретные бизнес-требования фич.
- Реализация кода (это предмет `apply`).
- Политики CI/CD (кроме `sync --check`).
- Выбор конкретных моделей (Claude, GPT, Gemini).

---

## 3. Solution Strategy

**Трёхслойное разделение ответственности:**

```
+---------------------------------------------------------------+
|  СЛОЙ 1: ДОСТАВКА ИНСТРУКЦИЙ (agnostic-ai)                    |
|  .agnostic-ai/  -->  sync  -->  CLAUDE.md / AGENTS.md / .cursor|
+---------------------------------------------------------------+
|  СЛОЙ 2: ПРОЦЕСС (OpenSpec)                                   |
|  propose -> specs -> design -> adr -> api-contract -> tasks   |
|         -> apply -> archive                                    |
+---------------------------------------------------------------+
|  СЛОЙ 3: КОНТРОЛЬ КАЧЕСТВА (adr-kit, MCP-инструменты)         |
|  adr_preflight / adr_create / adr_planning_context            |
+---------------------------------------------------------------+
```

Слои изолированы: замена любого из них не ломает остальные.
`agnostic-ai` не знает о OpenSpec. OpenSpec не знает, какой инструмент читает файл.
`adr-kit` не знает, откуда пришло архитектурное решение.

---

## 4. Building Block View

### 4.1. Слой источника (agnostic-ai)

```
.agnostic-ai/
├── AGNOSTIC_AI.md              # Общий контекст проекта
├── rules/
│   ├── adr-required.md         # Правило: design change -> ADR
│   ├── api-contract.md         # Правило: contract-first
│   └── testing.md              # Правило: one behavior per test
├── skills/
│   └── release/SKILL.md
├── mcps/
│   ├── adr-kit.yaml            # MCP-конфиг для adr-kit
│   ├── github.yaml
│   └── openapi.yaml
└── overlays/
    └── claude.settings.json
agnostic-ai.yaml                 # targets: [claude, codex, gemini, cursor]
```

### 4.2. Слой нативных файлов (генерируется)

```
CLAUDE.md                        # для Claude Code
AGENTS.md                        # для Cursor, Codex, Windsurf
GEMINI.md                        # для Gemini CLI
.cursor/rules/*.md               # для Cursor (scoped rules)
.claude/settings.json            # overlays
.codex/config.toml               # overlays
```

Все файлы генерируются `sync`, все имеют маркеры `<!-- agnostic-ai:rules:start/end -->`
и `<!-- source: ... -->` для трассируемости.

### 4.3. Слой процесса (OpenSpec)

```
openspec/
├── AGENTS.md                    # Шлюз: инструкция по OpenSpec воркфлоу
├── schemas/
│   └── proposal-to-apply/
│       └── schema.yaml          # Кастомная схема (7 артефактов)
├── specs/                       # Актуальные спецификации (delta-based)
├── changes/
│   └── <change-id>/
│       ├── proposal.md
│       ├── specs/
│       ├── design.md
│       ├── adr.md
│       ├── api-contract.md
│       └── tasks.md
└── archive/                     # Завершённые change'и
```

### 4.4. Слой adr-kit (MCP Gateway)

```
adr-kit (MCP server)
├── adr_preflight(decision)      -> ALLOWED | REQUIRES_ADR | BLOCKED
├── adr_create(decision)         -> MADR + ESLint/Ruff rules
├── adr_planning_context()       -> Контекст для apply
└── adr_exists(decision_hash)    -> Bool (проверка дубликатов)
```

Сгенерированные правила сохраняются в `.adr/rules/` и **коммитятся**
(не синхронизируются через agnostic-ai, т.к. это рантайм-артефакты).

---

## 5. Runtime View (Pipeline)

### 5.1. Общая диаграмма (ASCII)

```
  +--------------+     +--------------+     +--------------+
  |  01.propose  |---->|  02.specs    |---->|  03.design   |
  |  MCP: git    |     |  MCP: github |     |              |
  +--------------+     +--------------+     +------+-------+
                                                   |
                                                   v
                                            +------+-------+
                                            | adr_preflight|
                                            +--+----+----+--+
                                               |    |    |
                          REQUIRES_ADR --------+    |    +-------- BLOCKED
                                               |    |              (Counter < N)
                                               v    v              |
                                     +---------+    +------+       v
                                     | adr_create|   |ALLOWED|  +--------+
                                     +-----+-----+   +---+--+  | 03.design|
                                           |             |     +--------+
                                           v             |
                                     +-----+-------------+--+
                                     |  04.adr              |
                                     |  MADR + .adr/rules/   |
                                     +----------+-----------+
                                                |
                                                v
                                     +----------+-----------+
                                     |  05.api-contract     |
                                     |  MCP: openapi        |
                                     |  openapi_validate()  |
                                     +----------+-----------+
                                                |
                                            valid? -------> invalid -> 04.api-contract
                                                |
                                                v
                                     +----------+-----------+
                                     |  06.tasks            |
                                     |  Hash = SHA256(      |
                                     |    design.md +       |
                                     |    adr.md +          |
                                     |    api.yaml)         |
                                     +----------+-----------+
                                                |
                                                v
                                     +----------+-----------+
                                     |  07.apply            |
                                     |  +-----------------+ |
                                     |  | hash_valid?     | |
                                     |  +---+----------+--+ |
                                     |      |          |    |
                                     |   yes|      no  |    |
                                     |      v          v    |
                                     |  adr_planning  ->06   |
                                     |  _context()          |
                                     |      |               |
                                     |      v               |
                                     |  MCP: filesystem/    |
                                     |  linter/runner       |
                                     |      |               |
                                     |      v               |
                                     |  Реализация кода     |
                                     +----------+-----------+
                                                |
                                                v
                                     +----------+-----------+
                                     |  08.archive          |
                                     |  MCP: github (PR)    |
                                     +----------------------+

  Counter >= N на BLOCKED --> [STOP: Human Intervention Required]
```

### 5.2. Точки вызова MCP по фазам

| Фаза            | MCP-инструменты              | Назначение                          |
|-----------------|------------------------------|-------------------------------------|
| 01. propose     | `git`                        | get_branch, diff                    |
| 02. specs       | `github`                     | fetch_issues, PRs                   |
| 03. design      | `adr-kit`                    | `adr_preflight()`                   |
| 04. adr         | `adr-kit`                    | `adr_create()`                      |
| 05. api-contract| `openapi`, `linter`          | генерация + валидация контракта     |
| 06. tasks       | —                            | расчёт хэша, расщепление задач      |
| 07. apply       | `adr-kit`, `filesystem`, `linter`, `runner` | контекст + реализация |
| 08. archive     | `github`                     | create_pr, merge                    |

### 5.3. Хэш-инвалидация

```
Hash_tasks = SHA256(design.md || adr.md || api.yaml)

На входе в apply:
    Hash_current = SHA256(design.md || adr.md || api.yaml)

    if Hash_tasks == Hash_current:
        proceed to adr_planning_context()
    else:
        rollback to 06.tasks
        recompute tasks
        recompute Hash_tasks
```

**Почему `design.md` включён в хэш:** если `design.md` меняется после
создания `adr`, это означает, что архитектурное решение могло быть
пересмотрено — `tasks` нужно пересобрать.

### 5.4. Обработка `BLOCKED`

```
Counter = 0
while adr_preflight() == BLOCKED:
    Counter += 1
    if Counter >= N:
        STOP -> Human Intervention
    return to 03.design
```

`N` — конфигурируемый параметр (по умолчанию 3).

### 5.5. Защита от дублирования ADR

Перед `adr_create()`:

```
if adr_exists(hash(decision)):
    reuse existing ADR
else:
    adr_create(decision)
```

---

## 6. Crosscutting Concepts

### 6.1. Детерминизм

- `agnostic-ai sync` даёт byte-stable output.
- `sync --check` в CI падает при расхождении.
- Все правила имеют `<!-- source: ... -->` для трассируемости.

### 6.2. Изоляция слоёв

- Слой 1 (доставка) не знает о слое 2 (процесс).
- Слой 2 не знает о слое 3 (контроль).
- Замена любого слоя возможна без правок в других.

### 6.3. Gatekeeper-паттерн

`adr_preflight` — не «документация после», а **фильтр перед**.
Три статуса: `ALLOWED`, `REQUIRES_ADR`, `BLOCKED`.
Ни один ADR не может быть пропущен, если решение требует фиксации.

### 6.4. Каскадная инвалидация

Изменение артефакта → пересчёт хэша → проверка на входе в `apply` →
откат на `tasks` при расхождении. Никаких ручных проверок.

### 6.5. MCP как единая точка контроля

Все внешние взаимодействия (git, github, openapi, filesystem, linter, runner)
проходят через MCP-серверы, объявленные в `.agnostic-ai/mcps/`.
Это делает пайплайн **инспектируемым** и **переносимым**.

---

## 7. Architectural Decisions

### ADR-001: OpenSpec как основа процесса

**Решение:** использовать OpenSpec вместо SpecKit или самописного воркфлоу.

**Обоснование:**
- Delta-механизм для brownfield-проектов (spec deltas вместо upfront спецификации).
- Кастомные схемы позволяют добавить шаги `adr` и `api-contract`.
- `AGENTS.md` как шлюз в OpenSpec воркфлоу.

### ADR-002: agnostic-ai для доставки инструкций

**Решение:** использовать agnostic-ai вместо symlink одного файла или ручного дублирования.

**Обоснование:**
- Разные инструменты читают разные форматы по разным путям.
- Symlink не транслирует байты между форматами.
- `sync --check` защищает от дрейфа в CI.

### ADR-003: Хэш-инвалидация вместо ручной синхронизации

**Решение:** вычислять SHA256 от `design.md + adr.md + api.yaml` на шаге `tasks`
и проверять на входе в `apply`.

**Обоснование:**
- Ручная проверка «не изменился ли ADR» — ненадёжна.
- Хэш даёт детерминированную защиту.
- Откат только на `tasks`, не на `design` — минимизирует потери.

### ADR-004: `adr_preflight` как gatekeeper

**Решение:** блокировать переход `design → adr` без валидации через `adr_preflight`.

**Обоснование:**
- Post-factum ADR — это фольклор.
- Gatekeeper с лимитом итераций предотвращает зацикливание.
- Эскалация к человеку при `Counter >= N`.

---

## 8. Risks and Technical Debt

### 8.1. Закрытые риски

- [x] **Зацикливание на `BLOCKED`** — решено через счётчик и эскалацию.
- [x] **Рассинхронизация ADR и tasks** — решено через хэш-инвалидацию.
- [x] **Дрейф инструкций между инструментами** — решено через `sync --check`.

### 8.2. Открытые риски

- [ ] **Кэш ADR** — при повторных итерациях может создаваться дубликат ADR.
  Решение: `adr_exists(decision_hash)` перед `adr_create`.
- [ ] **Gate на api-contract** — нет явной проверки `openapi_validate()`.
  Решение: добавить ветку `invalid → 04.api-contract`.
- [ ] **Директория для сгенерированных правил** — ESLint/Ruff из `adr_create`
  не описаны в структуре. Решение: `.adr/rules/`, коммитится, не синхронизируется.
- [ ] **`design.md` в хэше** — включён, но требует явного указания в schema.yaml.

### 8.3. Технический долг

- Нет автоматической проверки, что все MCP-серверы из `.agnostic-ai/mcps/`
  действительно доступны на каждой фазе.
- Нет метрик по количеству `BLOCKED`-итераций в разрезе проектов.
- Нет шаблонов MADR на русском языке (если команда русскоязычная).

---

## 9. Glossary

| Термин                | Значение                                                      |
|-----------------------|---------------------------------------------------------------|
| `adr_preflight`       | MCP-инструмент проверки решения перед фиксацией ADR           |
| `ALLOWED`             | Статус: решение не требует ADR, можно продолжать              |
| `REQUIRES_ADR`        | Статус: решение требует фиксации ADR                          |
| `BLOCKED`             | Статус: решение противоречит существующим ADR                 |
| `adr_create`          | MCP-инструмент создания MADR + правил линтера                 |
| `adr_planning_context`| MCP-инструмент загрузки контекста ADR перед `apply`           |
| Hash-invalidation     | Пересчёт SHA256 при изменении артефактов, откат при расхождении|
| Spec delta            | Механизм OpenSpec для описания изменений в спецификации       |
| Blueprint-as-Code     | Документ-спецификация, версионируемый в репозитории           |
| MCP Gateway           | Единая точка вызова MCP-инструментов через конфиг             |

---

## 10. Приложения

### 10.1. Пример `.agnostic-ai/rules/adr-required.md`

```markdown
---
name: adr-required
alwaysApply: true
---

Любое изменение в `openspec/changes/<id>/design.md`, затрагивающее:
- выбор библиотеки или фреймворка,
- изменение архитектурного паттерна,
- изменение публичного API,

должно быть проверено через `adr_preflight()` и, при статусе
`REQUIRES_ADR`, зафиксировано через `adr_create()`.
```

### 10.2. Пример `agnostic-ai.yaml`

```yaml
targets:
  - claude
  - codex
  - gemini
  - cursor

sources:
  instructions: .agnostic-ai/AGNOSTIC_AI.md
  rules: .agnostic-ai/rules/
  skills: .agnostic-ai/skills/
  mcps: .agnostic-ai/mcps/
  overlays: .agnostic-ai/overlays/
```

### 10.3. Пример кастомной схемы OpenSpec

```yaml
# openspec/schemas/proposal-to-apply/schema.yaml
name: proposal-to-apply
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []
  - id: specs
    generates: specs/**/*.md
    requires: [proposal]
  - id: design
    generates: design.md
    requires: [specs]
  - id: adr
    generates: adr.md
    requires: [design]
  - id: api-contract
    generates: api-contract.md
    requires: [design, adr]
  - id: tasks
    generates: tasks.md
    requires: [specs, design, adr, api-contract]
```

---

**Конец документа.**

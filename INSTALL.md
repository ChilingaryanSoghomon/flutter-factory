# Установка и инициализация

Что поставить, чтобы поднять агентную работу с Flutter-проектом: что и как
добавить, по строке на пункт. Подробные конфиги по агентам и разбор граблей сюда
не кладём — это reference.

## Один раз на машину / в агенте

- **superpowers** — плагин агента, ведёт процесс: https://github.com/obra/superpowers
- **Flutter MCP** (`dart mcp-server`, встроен в Dart SDK) —
  `claude mcp add --scope user --transport stdio dart -- dart mcp-server`;
  в Codex / Cursor / VS Code — тот же сервер в их конфиге.
- **Context7** (свежие доки пакетов, HTTP, диск ~0) —
  `claude mcp add --transport http --scope user context7 https://mcp.context7.com/mcp`

## В каждом проекте

- **flutter-factory** —
  `git clone git@github.com:ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory`,
  затем в `.gitignore` основного проекта: `.agents/flutter-factory/`
- **Точки входа** в корне проекта:
  - `CLAUDE.md`: `Always read AGENTS.md at the start of every session and follow it.`
  - `AGENTS.md`: ссылка на `.agents/flutter-factory/README.md` + строка про клон/обновление комплекта.
- **Официальные скиллы** (→ `.agents/skills`) —
  `npx skills add flutter/skills --skill '*' --agent universal` и то же для `dart-lang/skills`.
- **Строгий анализатор** — `analysis_options.yaml`:
  `very_good_analysis` + `strict-casts/strict-inference/strict-raw-types`.
- **Guardrails** — pre-commit:
  `dart format --set-exit-if-changed . && flutter analyze && flutter test`.

## Опционально (по нужде)

- **mcp_toolkit** — агент сам дёргает UI (тап, ввод, snapshot); правит `main.dart`, ставить по проекту.
- **DCM** — метрики и анти-паттерны (есть бесплатный tier).

## Обновлять и править комплект

`.agents/flutter-factory/` — полноценный клон отдельного репозитория.

- Подтянуть: `git -C .agents/flutter-factory pull --ff-only`
- Из проекта: правишь там же, затем `git -C .agents/flutter-factory commit -am "…"` и `… push`.
  Remote по SSH — токен не нужен, пуш идёт по ключу.

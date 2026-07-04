# Установка и инициализация

Что поставить, чтобы поднять агентную работу с Flutter-проектом: что и как
добавить, по строке на пункт. Подробные конфиги по агентам и разбор граблей сюда
не кладём — это reference.

## Один раз на машину / в агенте

Скиллы ставим в оба target-а:

- `universal` — общий каталог `~/.agents/skills` для Codex и других агентов,
  которые читают universal skills.
- `claude-code` — каталог `~/.claude/skills` для Claude Code.

После установки перезапусти соответствующий агент: текущая сессия обычно не
видит новые `SKILL.md`, пока не стартует заново.

- **superpowers** — плагин агента, ведёт процесс: https://github.com/obra/superpowers
- **Flutter MCP** (`dart mcp-server`, встроен в Dart SDK) —
  `claude mcp add --scope user --transport stdio dart -- dart mcp-server`;
  в Codex / Cursor / VS Code — тот же сервер в их конфиге.
- **Context7** (свежие доки пакетов, HTTP, диск ~0) —
  `claude mcp add --transport http --scope user context7 https://mcp.context7.com/mcp`
- **Официальные скиллы** (глобально, `-g`, для Codex/universal и Claude Code) —
  `npx skills add flutter/skills --agent universal claude-code -g --skill flutter-build-responsive-layout --skill flutter-fix-layout-issues --skill flutter-setup-localization --skill flutter-add-widget-test --skill flutter-add-integration-test`
  и `npx skills add dart-lang/skills --agent universal claude-code -g --skill dart-add-unit-test --skill dart-generate-test-mocks --skill dart-collect-coverage --skill dart-run-static-analysis --skill dart-resolve-package-conflicts --skill dart-use-pattern-matching`
- **Дизайн-скиллы** (облик до кода, глобально `-g`, для Codex/universal и Claude Code) —
  `npx skills add anthropics/skills --agent universal claude-code -g --skill frontend-design`
  и `npx skills add ehmo/platform-design-skills --agent universal claude-code -g --skill ios-design-guidelines --skill android-design-guidelines`
  (другие платформы — тем же образом по нужде).
- **Проверить установку** —
  `npx skills ls -g --json`.

## В каждом проекте

- **flutter-factory** —
  `git clone git@github.com:ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory`,
- **Точки входа** в корне проекта:
  - `CLAUDE.md`: `Always read AGENTS.md at the start of every session and follow it.`
  - `AGENTS.md`: flutter-factory — ОТДЕЛЬНОЙ секцией верхнего уровня, не пунктом
    внутри другого раздела (иначе тонет и агент проскакивает — проверено). Клон и
    обновление сюда НЕ пишем: в самой `AGENTS.md` flutter-factory считается уже на
    месте. Добавь в `AGENTS.md` ровно такой блок:

```markdown
## Flutter: обязательные рамки

Любое касание Dart/Flutter-кода (создание, изменение, багфикс, ревью) идёт по
flutter-factory. Это не рекомендация. ПЕРЕД тем как писать или менять код:

1. Прочитай `.agents/flutter-factory/README.md` и
   `.agents/flutter-factory/conventions.md`.
2. Держи `conventions.md` открытым на всех фазах — нейминг, слои, конструкторы
   берутся оттуда, а не из головы.
3. Перед «готово» — `flutter analyze` и `flutter test`, оба зелёные.

Пропуск этого шага — ошибка, а не экономия времени.
```

- **Анализатор** — `analysis_options.yaml`:
  `flutter_lints` + `strict-casts/strict-inference/strict-raw-types`.
- **mcp_toolkit** — `flutter pub add mcp_toolkit`; агент сам дёргает UI
  (тап, ввод, snapshot), инициализация описана в `flutter-app-scaffold`.

```dart
  // пример 
  if (kDebugMode) {
    MCPToolkitBinding.instance
      ..initialize()
      ..initializeFlutterToolkit();
  }
```
- **Guardrails** — pre-commit:
  `dart format --set-exit-if-changed . && flutter analyze && flutter test`.

## Опционально (по нужде)

- **DCM** — метрики и анти-паттерны (есть бесплатный tier).

## Обновлять и править комплект

`.agents/flutter-factory/` — полноценный клон отдельного репозитория.

- Подтянуть: `git -C .agents/flutter-factory pull --ff-only`
- Из проекта: правишь там же, затем `git -C .agents/flutter-factory commit -am "…"` и `… push`.
  Remote по SSH — токен не нужен, пуш идёт по ключу.

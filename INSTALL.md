# Установка flutter-factory в проект

Комплект распространяется через GitHub и подключается как обычный локальный
`git clone` внутри проекта. Основной репозиторий не должен отслеживать
`.agents/flutter-factory/`: это отдельный репозиторий с отдельной историей.

## 1. Подключить к Flutter-проекту

Из корня проекта:

```bash
mkdir -p .agents
git clone git@github.com:ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory
```

Добавь локальный клон в `.gitignore` основного проекта:

```gitignore
.agents/flutter-factory/
```

## 2. Создать точки входа в корне проекта

Без них агент не узнает про комплект. Скопируй два файла по шаблонам ниже.

**`CLAUDE.md`** (для Claude Code):

```markdown
Always read AGENTS.md at the start of every session and follow it.
```

**`AGENTS.md`** (универсальный — читают и другие агенты):

```markdown
# <Проект>

Для Flutter-работы сначала прочитай и выполняй:
`.agents/flutter-factory/README.md`.

flutter-factory подключён как отдельный локальный git clone. Если папки нет:
`mkdir -p .agents && git clone git@github.com:ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory`.

Обновить комплект:
`git -C .agents/flutter-factory pull --ff-only`.
```

## Поддерживаемые агенты

`AGENTS.md` — универсальная точка входа, её читают разные агенты (Claude Code,
Codex и другие). `CLAUDE.md` нужен дополнительно только для Claude Code. Для
Codex-проекта достаточно `AGENTS.md`.

## superpowers (процесс)

flutter-factory задаёт *рамки* Flutter-проекта, а *процесс* — дизайн,
планирование, диагностику, реализацию, ревью — ведёт superpowers. Они задуманы
**в паре**, поэтому поставь superpowers:
https://github.com/obra/superpowers (как плагин агента, по инструкции их репо).
Доменные скиллы читаются и без него как справка, но полный метод — вместе.

## 3. Обновить комплект в проекте

```bash
git -C .agents/flutter-factory pull --ff-only
```

## Править комплект прямо из проекта

`.agents/flutter-factory/` — это полноценный клон репозитория комплекта.
Правки делаешь там же и пушишь в общий источник:

```bash
cd .agents/flutter-factory
# изменения...
git commit -am "fix: ..."
git push
```

В других проектах потом подтянуть: `git -C .agents/flutter-factory pull --ff-only`.

## Клонирование проекта на новой машине

```bash
git clone <project-url>
cd <project-name>
mkdir -p .agents
git clone git@github.com:ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory
```

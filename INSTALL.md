# Установка flutter-factory в проект

Комплект распространяется через GitHub и подключается как git submodule.

## 1. Подключить к Flutter-проекту

Из корня проекта:

```bash
git submodule add https://github.com/ChilingaryanSoghomon/flutter-factory.git .agents/flutter-factory
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

flutter-factory подключён как git submodule. Обновить комплект:
`git submodule update --remote .agents/flutter-factory`.
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
git submodule update --remote .agents/flutter-factory
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

В других проектах потом подтянуть: `git submodule update --remote .agents/flutter-factory`.

## Клонирование проекта с уже подключённым комплектом

```bash
git clone --recurse-submodules <project-url>
# или, если проект уже склонирован:
git submodule update --init --recursive
```

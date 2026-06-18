---
name: flutter-theme
description: Использовать для оформления Flutter-приложения — тема-система, светлая и тёмная ThemeData, токены оформления, цвета, типографика, отступы, component themes и применение темы в UI. Про единый внешний вид приложения.
---

# Оформление и тема

Тема приложения живёт в `lib/src/common/theme/`. Здесь находятся визуальные
токены, `ThemeData`, светлая и тёмная темы, component themes и правила
применения темы в UI.

Сначала проверяй именно `lib/src/common/theme/`. Если папка и базовый файл темы
есть — работай с текущей системой темы и не создавай параллельный слой. Если
папки или базового файла темы нет — прочитай
`references/create-theme-from-scratch.md` и создай только минимальный слой темы.

## Раскладка

```text
lib/src/common/theme/
├── app_theme.dart              # сборка ThemeData: light/dark
├── app_color_scheme.dart       # optional: ColorScheme/seed/product colors
├── app_spacing.dart            # optional: отступы, радиусы, размеры
└── app_theme_extensions.dart   # optional: ThemeExtension, если ThemeData мало
```

В минимальном случае обязателен только `app_theme.dart`. Остальное вводи только
когда появляется повторяемый токен или явная продуктовая система.

## Типографика

Основной источник текстовых стилей — стандартный Flutter `TextTheme`:
`displayLarge`, `headlineMedium`, `titleLarge`, `bodyMedium`, `labelLarge` и
другие Material-токены.

- Не заводи параллельный `AppTextStyles` только ради размеров шрифта.
- Не пиши случайные `TextStyle(fontSize: ...)` в экранах и виджетах.
- Для особого текста бери ближайший смысловой токен и меняй только отличие:
  `Theme.of(context).textTheme.titleLarge?.copyWith(...)`.
- Повторяемый нестандартный стиль выноси в тему: настрой `textTheme` при сборке
  `ThemeData` или добавь `ThemeExtension`, если стандартных токенов реально не
  хватает.

Сначала выбирай стиль по роли текста в интерфейсе, а не по числу пикселей.

Правильно:

```dart
final theme = Theme.of(context);

Text(
  title,
  style: theme.textTheme.titleLarge?.copyWith(
    color: theme.colorScheme.primary,
  ),
);
```

Неправильно без причины:

```dart
Text(
  title,
  style: const TextStyle(fontSize: 21, color: Color(0xFF3366FF)),
);
```

## Цвета и компоненты

- Цвета бери из `ColorScheme`: `primary`, `secondary`, `surface`, `error`,
  `onSurface` и т.п.
- Продуктовые цвета, которых нет в `ColorScheme`, добавляй как
  `ThemeExtension`, а не как разрозненные константы в фичах.
- Базовые кнопки настраивай через component themes:
  `ElevatedButtonThemeData`, `OutlinedButtonThemeData`, `TextButtonThemeData`.
- Особые кнопки можно локально уточнять через `styleFrom`/`copyWith`, но база
  должна оставаться в теме.

## Дизайн-фильтр

Визуальное направление — Material-реализация с iOS-like сдержанностью. Это не
добавляет Cupertino API; это фильтр качества для UI на Material.

- Строй спокойную иерархию вместо декоративного шума.
- Держи текст читаемым, а отступы предсказуемыми.
- Делай touch targets около `44x44` или больше.
- Учитывай text scaling: текст не должен ломать layout при увеличении.
- Повторяемые радиусы и отступы держи через токены.
- Повторяемые элементы управления оформляй через component themes.
- Локальный visual override оставляй только для действительно уникального
  состояния.

## Использование в UI

Виджеты и экраны берут оформление из `Theme.of(context)`. Хардкод допустим
только как осознанное исключение рядом с конкретным уникальным UI-состоянием.

## Процесс

1. Проверь `lib/src/common/theme/`.
2. Если папки или базового файла темы нет, прочитай
   `references/create-theme-from-scratch.md`.
3. Если тема есть, работай с текущим `app_theme.dart` и соседними токенами.
4. Замени локальные хардкоды цветов, текста и кнопок на theme-based usage.
5. Вынеси повторяющиеся стили кнопок в component themes.
6. Оставь локальные overrides только там, где они действительно уникальны.

## Проверка

- Текстовая иерархия строится через `TextTheme`, а не через случайные размеры.
- Повторяющиеся кнопки получают стиль из темы.
- Светлая и тёмная темы имеют одинаковую структуру токенов.
- Нет второго параллельного слоя темы.
- Нет инструкций и правок вне `common/theme/`, если задача только про слой темы.
- Перед готовностью запусти `flutter analyze` и релевантные тесты.

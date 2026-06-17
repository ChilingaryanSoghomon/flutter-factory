# Создание темы с нуля

Читай этот reference только когда в приложении нет `lib/src/common/theme/` или
базового файла темы. Создай минимальный слой темы и не описывай файлы вне
`common/theme/`.

## Минимальная структура

```text
lib/src/common/theme/
└── app_theme.dart
```

Дополнительные файлы (`app_color_scheme.dart`, `app_spacing.dart`,
`app_theme_extensions.dart`) добавляй только когда появились повторяемые токены
или продуктовые значения.

## app_theme.dart

Минимальный вариант:

```dart
import 'package:flutter/material.dart';

final class AppTheme {
  const AppTheme._();

  static ThemeData light() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: const Color(0xFF0A84FF),
      brightness: Brightness.light,
    );

    return _build(colorScheme);
  }

  static ThemeData dark() {
    final colorScheme = ColorScheme.fromSeed(
      seedColor: const Color(0xFF0A84FF),
      brightness: Brightness.dark,
    );

    return _build(colorScheme);
  }

  static ThemeData _build(ColorScheme colorScheme) {
    final typography = Typography.material2021();
    final textTheme = colorScheme.brightness == Brightness.dark
        ? typography.white
        : typography.black;

    return ThemeData(
      useMaterial3: true,
      colorScheme: colorScheme,
      textTheme: textTheme,
      elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
          minimumSize: const Size(44, 44),
          textStyle: textTheme.labelLarge,
        ),
      ),
      outlinedButtonTheme: OutlinedButtonThemeData(
        style: OutlinedButton.styleFrom(
          minimumSize: const Size(44, 44),
          textStyle: textTheme.labelLarge,
        ),
      ),
      textButtonTheme: TextButtonThemeData(
        style: TextButton.styleFrom(
          minimumSize: const Size(44, 44),
          textStyle: textTheme.labelLarge,
        ),
      ),
    );
  }
}
```

## Правила

- Держи слой темы минимальным.
- Сначала используй `ColorScheme`, `TextTheme` и component themes.
- Не создавай отдельный набор текстовых стилей только ради размеров шрифта.
- Продуктовые цвета и токены добавляй позже через `ThemeExtension`, если
  `ColorScheme` и `ThemeData` уже не хватает.
- Не добавляй Cupertino API в этот reference.

# Виджет-тесты

Один экран или виджет в тестовой среде: что он рисует по состоянию и как реагирует на ввод. Без устройства и настоящих плагинов. Файл — рядом с виджетом, зеркаля путь в `test/`.

Нужен `flutter_test` (`testWidgets`, `WidgetTester`, finders, matchers); для подмены зависимостей — `mocktail`.

## Каркас

Виджет поднимаешь через `pumpWidget`, обернув в минимум окружения, которое ему нужно (`MaterialApp` для темы и локализаций, провайдер состояния и т.п.). Дальше: находишь элементы (`find`), действуешь, перестраиваешь дерево, проверяешь.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  testWidgets('рисует имя и реагирует на тап', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(home: ProfileView(name: 'Ann')),
    );

    expect(find.text('Ann'), findsOneWidget);

    await tester.tap(find.byIcon(Icons.refresh));
    await tester.pump();

    expect(find.text('Обновлено'), findsOneWidget);
  });
}
```

## Перестроение дерева — чем тыкать

- `pump()` — один кадр после смены состояния (обычный случай).
- `pumpAndSettle()` — крутит кадры, пока идут анимации; для переходов и спиннеров.
- `enterText()` — ввод в поле; `scrollUntilVisible()` — доскролл до элемента ленивого списка.

## Подмена состояния

Когда виджет читает состояние из BLoC, дай ему BLoC с подменёнными контрактами (или зафиксируй нужное состояние дублёром) — тогда рендер детерминирован и тест проверяет только UI, а не настоящие данные.

## Golden-проверка (по необходимости)

Если важен точный вид, а не только наличие элементов, сравни кадр с эталонным изображением: `await expectLater(find.byType(ProfileView), matchesGoldenFile('goldens/profile.png'))`. Эталон генерится `flutter test --update-goldens`.

## Прогон

`flutter test test/path/to/widget_test.dart`.

# Модульные тесты

Чистая логика без виджетов: мапперы, доменные функции, usecase'ы, реализации репозиториев. Файл кладёшь рядом с тестируемым, зеркаля путь из `lib/src/` в `test/`, с суффиксом `_test.dart`.

Нужны `flutter_test` (структура и `expect`) и инструмент дублирования контрактов — `mocktail`; оба в `dev_dependencies`.

## Каркас

`test/` повторяет структуру `lib/src/`. Внутри — `group` на единицу и `test` на случай; в каждом тесте три шага: подготовка → действие → проверка.

```dart
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('ProfileMapper', () {
    test('переводит DTO в доменную модель', () {
      const dto = ProfileDto(name: 'Ann', age: 30);

      final profile = ProfileMapper.toDomain(dto);

      expect(profile, const Profile(name: 'Ann', age: 30));
    });
  });
}
```

## Подмена источника через контракт

Реализация репозитория зависит от контракта источника, а не от конкретного пакета. В тесте на место контракта даёшь дублёр и задаёшь его ответ — настоящие сеть и база не участвуют.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class _MockProfileApi extends Mock implements ProfileApi {} // ProfileApi — контракт

void main() {
  late _MockProfileApi api;
  late ProfileRepositoryImpl repository;

  setUp(() {
    api = _MockProfileApi();
    repository = ProfileRepositoryImpl(api: api);
  });

  test('маппит ответ источника в доменную модель', () async {
    when(() => api.fetchProfile())
        .thenAnswer((_) async => const ProfileDto(name: 'Ann', age: 30));

    final profile = await repository.getProfile();

    expect(profile, const Profile(name: 'Ann', age: 30));
  });

  test('сбой источника приводит к доменной ошибке', () async {
    when(() => api.fetchProfile()).thenThrow(Exception('network'));

    expect(repository.getProfile(), throwsA(isA<AppFailure>()));
  });
}
```

Ошибку источника репозиторий приводит к единому доменному типу (`AppFailure`) — это и проверяешь, а не сырое исключение.

## Прогон

`flutter test test/path/to/file_test.dart` — один файл, или `flutter test` — все.

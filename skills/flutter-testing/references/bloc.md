# Тесты логики состояния

Логика фичи живёт в BLoC (или Cubit, когда событий нет). Тест подаёт события и проверяет **поток состояний**, который BLoC выдаёт в ответ. Файл — рядом с BLoC, зеркаля путь в `test/`.

Нужны `bloc_test` (помощник `blocTest`) и `flutter_test`, плюс `mocktail` для дублёров; всё в `dev_dependencies`.

## Каркас

`blocTest` описывает один сценарий декларативно: как собрать BLoC (`build`), что подать (`act`), какой список состояний ждём (`expect`). Зависимости BLoC — контракты — подменяешь дублёром в `build`.

```dart
import 'package:bloc_test/bloc_test.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class _MockProfileRepository extends Mock implements ProfileRepository {}

void main() {
  late ProfileRepository repository;

  setUp(() => repository = _MockProfileRepository());

  blocTest<ProfileBloc, ProfileState>(
    'успех: loading → loaded',
    build: () {
      when(() => repository.getProfile())
          .thenAnswer((_) async => const Profile(name: 'Ann'));
      return ProfileBloc(repository: repository);
    },
    act: (bloc) => bloc.add(const ProfileRequested()),
    expect: () => const [
      ProfileLoading(),
      ProfileLoaded(Profile(name: 'Ann')),
    ],
  );

  blocTest<ProfileBloc, ProfileState>(
    'сбой: loading → error',
    build: () {
      when(() => repository.getProfile()).thenThrow(const AppFailure.network());
      return ProfileBloc(repository: repository);
    },
    act: (bloc) => bloc.add(const ProfileRequested()),
    expect: () => const [
      ProfileLoading(),
      ProfileError(),
    ],
  );
}
```

## Заметки

- Проверяешь **наблюдаемые состояния по порядку**, а не внутренние вызовы. Если важно, что репозиторий вызван, — это отдельная проверка через `verify`, не основная.
- `expect` — ровно те состояния, что ждём после стартового; стартовое в список не входит.
- Нужно начать не с дефолта — задай `seed`. Нужно дождаться задержек внутри обработчика — добавь `wait`.
- Cubit тестируется тем же `blocTest`: в `act` зовёшь его методы вместо `add`.

## Прогон

`flutter test test/path/to/bloc_test.dart`.

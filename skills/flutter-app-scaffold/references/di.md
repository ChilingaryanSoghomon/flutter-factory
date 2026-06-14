# DI — шаблон каркаса

Кладётся в `lib/src/common/core/di/`. Весь код приложения зависит от **абстракции** (`Resolver` — чтение, `Registrar` — регистрация), а не от контейнера. Контейнер — деталь: тут он обёртка над `kiwi`, можно заменить, код этого не заметит.

## Абстракция и доступ из контекста

```dart
import 'package:flutter/widgets.dart';
import 'package:provider/provider.dart';

abstract class Resolver {
  T resolve<T>([String? name]);
}

abstract class Registrar {
  void registerFactory<T>(Factory<T> factory, {String? name});

  void registerSingleton<T>(T singleton, {String? name});

  void registerLazySingleton<T>(Factory<T> factory, {String? name});

  void unregisterSingleton<T>({String? name});

  void clear();
}

typedef Factory<T> = T Function(Resolver r);

typedef Getter<T> = T Function();

// UI достаёт зависимости из контекста; не-UI получает Resolver через конструктор.
extension BuildContextDiExt on BuildContext {
  Resolver get resolver => Provider.of<Resolver>(this, listen: false);

  T resolve<T>([String? name]) =>
      Provider.of<Resolver>(this, listen: false).resolve<T>(name);
}
```

## Контейнер (обёртка над kiwi)

```dart
import 'package:kiwi/kiwi.dart' as kiwi;

class DiContainer implements Registrar, Resolver {
  final _container = kiwi.KiwiContainer.scoped();
  DiContainer._internal();
  static DiContainer? instance;
  factory DiContainer() {
    instance ??= DiContainer._internal();
    return instance!;
  }

  @override
  void registerFactory<T>(Factory<T> factory, {String? name}) {
    _container.registerFactory((c) => factory(this), name: name);
  }

  @override
  void registerLazySingleton<T>(Factory<T> factory, {String? name}) {
    _container.registerSingleton((c) => factory(this), name: name);
  }

  @override
  void registerSingleton<T>(T singleton, {String? name}) {
    _container.registerInstance(singleton, name: name);
  }

  @override
  void unregisterSingleton<T>({String? name}) {
    _container.unregister<T>(name);
  }

  @override
  T resolve<T>([String? name]) => _container.resolve(name);

  @override
  void clear() {
    _container.clear();
  }
}
```

## Сборка (diSetup)

Берёт `AppConfig`, регистрирует реализации контрактов в `Registrar` и возвращает `Resolver`. Только здесь реализации соединяются с контрактами.

```dart
Future<Resolver> setup(AppConfig appConfig) async {
  final di = DiContainer();
  // di.registerSingleton<AppConfig>(appConfig);
  // di.registerLazySingleton<SomeRepository>((r) => SomeRepositoryImpl(r.resolve()));
  return di;
}
```

# Простой KV — shared_preferences

Папка: `common/storage/preferences/`. Несекретные значения по ключу. Контракт, его реализация и общий сторедж с конкретным хранением.

## 1. Контракт — что вообще умеем

`common/storage/preferences/key_value_storage.dart` — абстракция: какие операции доступны (положить/взять по ключу). Через неё приложение не привязано к конкретному пакету — сменишь способ хранения, потребителей не трогаешь.

```dart
abstract class KeyValueStorage {
  Future<bool?> getBool(String key);
  Future<void> setBool(String key, bool value);

  Future<String?> getString(String key);
  Future<void> setString(String key, String value);

  Future<void> remove(String key);
}
```

Набор минимальный — добавляй типы (`int`, `double`, список) по мере надобности.

## 2. Реализация — поверх `shared_preferences`

`common/storage/preferences/key_value_storage_impl.dart`. Конкретный `shared_preferences` виден только здесь.

```dart
import 'package:shared_preferences/shared_preferences.dart';

class KeyValueStorageImpl implements KeyValueStorage {
  KeyValueStorageImpl({required SharedPreferences prefs}) : _prefs = prefs;

  final SharedPreferences _prefs;

  @override
  Future<bool?> getBool(String key) async => _prefs.getBool(key);

  @override
  Future<void> setBool(String key, bool value) => _prefs.setBool(key, value);

  @override
  Future<String?> getString(String key) async => _prefs.getString(key);

  @override
  Future<void> setString(String key, String value) =>
      _prefs.setString(key, value);

  @override
  Future<void> remove(String key) => _prefs.remove(key);
}
```

## 3. Общий сторедж — где живёт конкретное хранение

`common/storage/preferences/shared_prefs_storage.dart`. Держит контракт (`KeyValueStorage`) и пишет на нём осмысленные методы под конкретные данные. **Это тот файл, где ты делаешь реализацию хранения**; какие операции доступны — смотри в контракте.

```dart
class SharedPrefsStorage {
  SharedPrefsStorage({required KeyValueStorage kv}) : _kv = kv;

  final KeyValueStorage _kv;

  static const _onboardingDone = 'onboarding_done';

  Future<bool> isOnboardingDone() async =>
      await _kv.getBool(_onboardingDone) ?? false;

  Future<void> setOnboardingDone(bool value) =>
      _kv.setBool(_onboardingDone, value);
}
```

Кто хранит конкретные данные (репозиторий и т.п.) — просит `SharedPrefsStorage` из контейнера, а не лезет в контракт и ключи сам. Сторедж не обязан быть один — большой можно разбить на несколько, каждый поверх того же `KeyValueStorage`.

## Регистрация

Один раз при сборке приложения в DI-контейнер кладутся два объекта: контракт `KeyValueStorage` — реализацией `KeyValueStorageImpl`, и фасад `SharedPrefsStorage` поверх контракта. Реализации нужен готовый `SharedPreferences` — он берётся асинхронно, получи его до регистрации. Как класть в контейнер — это часть про контейнер; дальше потребители берут `SharedPrefsStorage` оттуда, реализация в их коде не появляется.

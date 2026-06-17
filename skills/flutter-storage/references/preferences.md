# Простой KV — shared_preferences

Папка: `common/storage/preferences/`. Несекретные значения по ключу. Три объекта: примитивы, их реализация и общий сторедж с конкретным хранением.

## 1. Примитивы — что вообще умеем

`common/storage/preferences/key_value_storage.dart` — абстракция: какие операции доступны (положить/взять по ключу). Сюда смотришь, чтобы понять, *как* можно сохранить.

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

## 2. Реализация примитивов — поверх `shared_preferences`

`common/storage/preferences/key_value_storage_impl.dart`. Механизм, создаётся один раз при сборке; конкретный `shared_preferences` дальше нигде не виден.

```dart
import 'package:shared_preferences/shared_preferences.dart';

class KeyValueStorageImpl implements KeyValueStorage {
  KeyValueStorageImpl(SharedPreferences prefs) : _prefs = prefs;

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

`common/storage/preferences/shared_prefs_storage.dart`. Держит примитивы (`KeyValueStorage`) и пишет на них осмысленные методы под конкретные данные. **Это тот файл, где ты делаешь реализацию хранения**; какие операции доступны — смотри в `KeyValueStorage`.

```dart
class SharedPrefsStorage {
  SharedPrefsStorage(KeyValueStorage kv) : _kv = kv;

  final KeyValueStorage _kv;

  static const _onboardingDone = 'onboarding_done';

  Future<bool> isOnboardingDone() async =>
      await _kv.getBool(_onboardingDone) ?? false;

  Future<void> setOnboardingDone(bool value) =>
      _kv.setBool(_onboardingDone, value);
}
```

Кто хранит конкретные данные (репозиторий и т.п.) — просит `SharedPrefsStorage`, а не лезет в примитивы и ключи сам.

## Регистрация

При сборке зависимостей: примитивы — на готовом `SharedPreferences`, сторедж — поверх примитивов.

```dart
final prefs = await SharedPreferences.getInstance();
di.registerSingleton<KeyValueStorage>(KeyValueStorageImpl(prefs));
di.registerLazySingleton<SharedPrefsStorage>(
  (r) => SharedPrefsStorage(r.resolve<KeyValueStorage>()),
);
```

Дальше берёшь `SharedPrefsStorage` из контейнера.

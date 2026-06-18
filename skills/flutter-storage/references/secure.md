# Защищённый KV — flutter_secure_storage

Папка: `common/storage/secure/`. Секретные значения по ключу: токены, ключи, пароли. На устройстве — Keychain (iOS) / Keystore (Android). Контракт, его реализация и общий сторедж с конкретным хранением.

## 1. Контракт — что вообще умеем

`common/storage/secure/secure_storage.dart` — абстракция: какие операции доступны (положить/взять/удалить секрет по ключу).

```dart
abstract class SecureStorage {
  Future<String?> read(String key);
  Future<void> write(String key, String value);
  Future<void> delete(String key);
  Future<void> deleteAll();
}
```

Отдельный тип от `KeyValueStorage` — нарочно: секрет нельзя случайно положить в простой KV, нужно осознанно взять `SecureStorage`. Тип = намерение.

## 2. Реализация — поверх `flutter_secure_storage`

`common/storage/secure/secure_storage_impl.dart`. Конкретный `flutter_secure_storage` виден только здесь.

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorageImpl implements SecureStorage {
  SecureStorageImpl({required FlutterSecureStorage storage}) : _storage = storage;

  final FlutterSecureStorage _storage;

  @override
  Future<String?> read(String key) => _storage.read(key: key);

  @override
  Future<void> write(String key, String value) =>
      _storage.write(key: key, value: value);

  @override
  Future<void> delete(String key) => _storage.delete(key: key);

  @override
  Future<void> deleteAll() => _storage.deleteAll();
}
```

## 3. Общий сторедж — где живёт конкретное хранение

`common/storage/secure/secrets_storage.dart`. Держит контракт (`SecureStorage`) и пишет на нём осмысленные методы под конкретные секреты. **Это тот файл, где ты делаешь реализацию хранения**; какие операции доступны — смотри в контракте.

```dart
class SecretsStorage {
  SecretsStorage({required SecureStorage storage}) : _storage = storage;

  final SecureStorage _storage;

  static const _authToken = 'auth_token';

  Future<String?> authToken() => _storage.read(_authToken);

  Future<void> setAuthToken(String value) => _storage.write(_authToken, value);

  Future<void> clearAuthToken() => _storage.delete(_authToken);
}
```

Кто хранит конкретные секреты (репозиторий и т.п.) — просит `SecretsStorage` из контейнера, а не лезет в контракт и ключи сам.

## Регистрация

Один раз при сборке приложения в DI-контейнер кладут два объекта. `SecureStorage` регистрируют под типом контракта, а объектом дают реализацию `SecureStorageImpl` (ей нужен экземпляр `FlutterSecureStorage`). Фасад `SecretsStorage` контракта не имеет — идёт под своим типом, поверх контракта. Как класть в контейнер — это часть про контейнер; дальше потребители берут `SecretsStorage` оттуда.

## Платформенные заметки

- **Android** — данные в Keystore. Для шифрованного бэкенда ключ-значений включи `AndroidOptions(encryptedSharedPreferences: true)`.
- **iOS/macOS** — пишется в Keychain и **переживает переустановку** приложения. Если это нежелательно — чисти при первом запуске после установки.
- Операции асинхронные и могут бросить на залоченном устройстве — оборачивай вызов.

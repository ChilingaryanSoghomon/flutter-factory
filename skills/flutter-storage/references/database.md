# Локальная БД — sqflite

Папка: `common/storage/database/`. Структурированные записи, доступ запросом.

`AppDatabase` держит соединение (создаётся один раз при сборке). Данные достаёшь через DAO: интерфейс `NotesDao` + реализация `NotesDaoImpl` с SQL внутри. В контексте — интерфейс DAO.

## Открытие БД

`common/storage/database/app_database.dart` — одно соединение на приложение, открывается асинхронно со схемой и версией:

```dart
import 'package:path/path.dart' as p;
import 'package:sqflite/sqflite.dart';

class AppDatabase {
  AppDatabase._(this.db);

  final Database db;

  static Future<AppDatabase> open() async {
    final path = p.join(await getDatabasesPath(), 'app.db');
    final db = await openDatabase(
      path,
      version: 1,
      onCreate: (db, version) async {
        await db.execute('''
          CREATE TABLE notes(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            created_at INTEGER NOT NULL
          )
        ''');
      },
      // onUpgrade: миграции, когда поднимаешь version
    );
    return AppDatabase._(db);
  }
}
```

## Доступ — контракт + DAO

Контракт описывает операции в терминах приложения; SQL-запросы спрятаны в реализации.

`common/storage/database/notes_dao.dart`:

```dart
abstract class NotesDao {
  Future<List<NoteRow>> recent({int limit = 20});
  Future<int> insert(NoteRow row);
  Future<void> deleteById(int id);
}

class NotesDaoImpl implements NotesDao {
  NotesDaoImpl({required Database db}) : _db = db;

  final Database _db;

  @override
  Future<List<NoteRow>> recent({int limit = 20}) async {
    final rows =
        await _db.query('notes', orderBy: 'created_at DESC', limit: limit);
    return rows.map(NoteRow.fromMap).toList();
  }

  @override
  Future<int> insert(NoteRow row) => _db.insert('notes', row.toMap());

  @override
  Future<void> deleteById(int id) =>
      _db.delete('notes', where: 'id = ?', whereArgs: [id]);
}
```

`NoteRow` — пример строки таблицы с `fromMap`/`toMap`; замени на свои данные.

## Регистрация

Один раз при сборке приложения: `AppDatabase` открывается (асинхронно) и кладётся в контейнер; контракт `NotesDao` — реализацией `NotesDaoImpl` поверх соединения `AppDatabase`. Как класть в контейнер — это часть про контейнер; дальше потребители берут `NotesDao` оттуда. Схема и миграции (`version` + `onUpgrade`) — деталь этой папки; запрос живёт внутри DAO, наружу торчит только контракт `NotesDao`.

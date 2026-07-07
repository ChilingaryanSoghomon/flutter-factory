# Контракты наблюдаемости

Стартовые абстрактные классы для зоны `common/logging/`. Это контракты — потребитель зависит только от них и берёт из DI. Реализации и наблюдатель пишутся отдельно и связываются в `diSetup`; конкретный пакет/SDK остаётся в их импортах.

## Observability / ErrorReporter

```dart
/// Уровень важности сообщения.
enum LogLevel { debug, info, warning, error }

/// Контракт технической видимости. Реализация прячет Sentry/Crashlytics/logger-пакет.
abstract class ErrorReporter {
  /// Записать сообщение с уровнем; для ошибок передают error и stackTrace.
  void log(
    LogLevel level,
    String message, {
    Object? error,
    StackTrace? stackTrace,
  });
}
```

Здесь живут сообщения, предупреждения, ошибки, падения и пойманные исключения.

## Analytics — продуктовые события

```dart
/// Контракт аналитики. Реализация прячет конкретный SDK.
abstract class Analytics {
  /// Отправить именованное событие с необязательными параметрами.
  void track(String event, {Map<String, Object?>? parameters});
}
```

Здесь только продуктовые события. Без Firestore, SharedPreferences, debug storage и любых методов чтения/записи данных.

## Наблюдатель BLoC — мост состояний

Наблюдатель не контракт, а реализация-мост. Он не владеет SDK: получает `ErrorReporter` и, если нужно, `Analytics` через конструктор. `onError` всегда шлёт ошибку в общий reporter; debug-консоль остаётся деталью реализации reporter-а.

```dart
class AppBlocObserver extends BlocObserver {
  AppBlocObserver({
    required ErrorReporter errorReporter,
    required Analytics analytics,
  })
      : _errorReporter = errorReporter,
        _analytics = analytics;

  final ErrorReporter _errorReporter;
  final Analytics _analytics;

  // onError -> _errorReporter.log(...)
  // onChange / onTransition -> _analytics.track(...) по необходимости
}
```

Связывание — один раз в `diSetup`: зарегистрировать `ErrorReporter` и `Analytics` на их реализации, повесить `AppBlocObserver`.

# Контракты наблюдаемости

Стартовые абстрактные классы для зоны `common/logging/`. Это контракты — потребитель зависит только от них и берёт из DI. Реализации (`LoggerImpl`, `AnalyticsImpl`) и наблюдатель пишутся отдельно и связываются в `diSetup`; конкретный пакет/SDK остаётся в их импортах.

## Logger — диагностика для разработчика

```dart
/// Уровень важности сообщения.
enum LogLevel { debug, info, warning, error }

/// Контракт логирования. Реализация прячет конкретный вывод/SDK.
abstract class Logger {
  /// Записать сообщение с уровнем; для ошибок передают error и stackTrace.
  void log(
    LogLevel level,
    String message, {
    Object? error,
    StackTrace? stackTrace,
  });
}
```

## Analytics — продуктовые события

```dart
/// Контракт аналитики. Реализация прячет конкретный SDK.
abstract class Analytics {
  /// Отправить именованное событие с необязательными параметрами.
  void track(String event, {Map<String, Object?>? parameters});
}
```

## Наблюдатель BLoC — мост состояний

Наблюдатель не контракт, а реализация-мост: получает `Logger` и `Analytics` через конструктор и шлёт в них переходы состояний. Каркас (тела — на этап реализации):

```dart
class AppBlocObserver extends BlocObserver {
  AppBlocObserver({required Logger logger, required Analytics analytics})
      : _logger = logger,
        _analytics = analytics;

  final Logger _logger;
  final Analytics _analytics;

  // onChange / onTransition / onError -> _logger.log(...) и/или _analytics.track(...)
}
```

Связывание — один раз в `diSetup`: зарегистрировать `Logger` и `Analytics` на их `Impl`, повесить `AppBlocObserver`.

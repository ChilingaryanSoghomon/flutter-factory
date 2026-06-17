# Запуск — шаблон каркаса

Поток: `main` → ран-функция → `App` (настройки оболочки + `MaterialApp`) → splash собирает DI → поддерево приложения. `App` держит `SettingsBloc` (тема, локаль) вне DI, поэтому splash уже тематизирован. Состав фич — под приложение.

## Конфиг приложения

`AppConfig` — сквозная конфигурация: значения, заданные один раз на старте и доступные всему приложению (окружение, фиче-флаги, общие endpoints/лимиты). Создаётся в `main`, провайдится через `Provider<AppConfig>` для UI и инъектится в `diSetup` для не-UI. Разные сборки/окружения — это разные значения за одним интерфейсом; здесь только контракт (`common/core/`), конкретный `AppConfigImpl` — под приложение.

```dart
abstract class AppConfig {
  String get environment;   // окружение сборки: dev / staging / prod
  bool flag(String name);   // фиче-флаги
  // + прочие сквозные значения: endpoints, лимиты, ключи
}
```

## Точка входа

```dart
// main.dart — собирает конфиг и зовёт ран-функцию
Future<void> main() => bootstrap(AppConfigImpl());
```

Несколько окружений (если нужны) — это просто разные `main`, зовущие ту же ран-функцию со своим `AppConfig`.

## Ран-функция (до UI)

Инициализация, которая обязана быть до интерфейса, и перехват необработанных ошибок в release.

```dart
Future<void> bootstrap(AppConfig appConfig) async {
  WidgetsFlutterBinding.ensureInitialized();
  // инициализация внешних сервисов до старта (краш-репортинг, мониторинг)

  if (kDebugMode) {
    runApp(App(appConfig: appConfig));
  } else {
    await runZonedGuarded(
      () async {
        // init мониторинга ошибок
        runApp(App(appConfig: appConfig));
      },
      (error, stack) {
        // отправить ошибку в краш-репортинг
      },
    );
  }
}
```

## Оболочка

`App` провайдит `AppConfig` и `SettingsBloc`, затем строит `MaterialApp`: тему и локаль берёт из `SettingsBloc`, `home` — splash. Настройки — вне DI.

```dart
class App extends StatelessWidget {
  final AppConfig appConfig;
  const App({super.key, required this.appConfig});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        Provider<AppConfig>(create: (_) => appConfig),
        BlocProvider<SettingsBloc>(
          create: (_) => SettingsBloc()..add(SettingsInitEvent()),
        ),
      ],
      child: BlocBuilder<SettingsBloc, SettingsState>(
        builder: (context, settings) {
          return MaterialApp(
            debugShowCheckedModeBanner: false,
            locale: settings.locale,
            theme: ThemeData(brightness: settings.brightness),
            // localizationsDelegates/supportedLocales добавляют с локализацией
            home: SplashScreen(
              diSetup: () => di.setup(appConfig),
              child: const _AppRoot(), // поддерево приложения после DI
            ),
          );
        },
      ),
    );
  }
}
```

## Настройки оболочки

`SettingsBloc` держит локаль и тему оболочки. Стартовые значения — дефолты устройства: локаль `en`, яркость из `PlatformDispatcher`. `SettingsInitEvent` — заглушка-хук под будущее: когда добавят сохранение, он прочитает сохранённое и заэмитит. Пока почти всегда нужная структура (тема + локализация) уже на месте и не мешает.

```dart
sealed class SettingsEvent {}

final class SettingsInitEvent extends SettingsEvent {}

class SettingsState {
  final Locale locale;
  final Brightness brightness;
  const SettingsState({required this.locale, required this.brightness});
}

class SettingsBloc extends Bloc<SettingsEvent, SettingsState> {
  SettingsBloc()
      : super(
          SettingsState(
            locale: const Locale('en'),
            brightness: PlatformDispatcher.instance.platformBrightness,
          ),
        ) {
    on<SettingsInitEvent>((event, emit) {
      // заглушка: возвращаем дефолты (en + тема устройства).
      // позже здесь читается сохранённое и эмитится в state.
    });
  }
}
```

## Splash-гейт

Показывает минимальный UI, асинхронно собирает DI и, как `Resolver` готов, отдаёт его вниз через `Provider`. Конкретный UI splash-а — деталь приложения.

```dart
sealed class SplashEvent {}

final class SplashAddDependenciesEvent extends SplashEvent {}

sealed class SplashState {}

final class SplashInitialState extends SplashState {}

final class SplashAddedDependenciesState extends SplashState {
  final Resolver container;
  SplashAddedDependenciesState(this.container);
}

class SplashBloc extends Bloc<SplashEvent, SplashState> {
  final Future<Resolver> Function() diSetup;

  SplashBloc({required this.diSetup}) : super(SplashInitialState()) {
    on<SplashAddDependenciesEvent>((event, emit) async {
      emit(SplashAddedDependenciesState(await diSetup()));
    });
  }
}

class SplashScreen extends StatefulWidget {
  final Widget child;
  final Future<Resolver> Function() diSetup;

  const SplashScreen({super.key, required this.child, required this.diSetup});

  @override
  State<SplashScreen> createState() => _SplashScreenState();
}

class _SplashScreenState extends State<SplashScreen> {
  late final SplashBloc _bloc;

  @override
  void initState() {
    super.initState();
    _bloc = SplashBloc(diSetup: widget.diSetup)
      ..add(SplashAddDependenciesEvent());
  }

  @override
  void dispose() {
    _bloc.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<SplashBloc, SplashState>(
      bloc: _bloc,
      builder: (context, state) {
        return switch (state) {
          SplashInitialState() => const SplashWidget(), // деталь приложения
          SplashAddedDependenciesState() => Provider<Resolver>.value(
            value: state.container,
            child: widget.child,
          ),
        };
      },
    );
  }
}
```

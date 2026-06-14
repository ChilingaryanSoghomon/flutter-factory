# Запуск — шаблон каркаса

Поток: `main` → ран-функция → `App`/`MaterialApp` → splash собирает DI → поддерево приложения. Тема/настройки оболочки и состав фич — отдельно, под приложение.

## Точка входа

```dart
// main.dart — собирает конфиг и зовёт ран-функцию
Future<void> main() => bootstrap(AppConfigImpl());
```

Несколько окружений (если нужны) — это просто разные `main`, зовущие ту же ран-функцию со своим конфигом.

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

`App` провайдит `AppConfig` и строит `MaterialApp`; `home` — splash. Тему/локаль сюда добавляют отдельно (в каркас не входит).

```dart
class App extends StatelessWidget {
  final AppConfig appConfig;
  const App({super.key, required this.appConfig});

  @override
  Widget build(BuildContext context) {
    return Provider<AppConfig>(
      create: (_) => appConfig,
      child: MaterialApp(
        debugShowCheckedModeBanner: false,
        home: SplashScreen(
          diSetup: () => di.setup(appConfig),
          child: const _AppRoot(), // поддерево приложения после DI
        ),
      ),
    );
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

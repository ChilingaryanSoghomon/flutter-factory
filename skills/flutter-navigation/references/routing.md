# Настройка роутера — шаблон

Движок по умолчанию — `go_router`; он спрятан за `AppNavigator` и сменяем. `AppNavigator` завязан на дерево (навигатор-ключ → контекст), поэтому его раздают через `Provider` в поддереве, а **не** кладут в DI-контейнер. Маршруты и охранники собраны тут, в фичи не протекают.

## Роутер

Класс держит общий `navigatorKey` (его же получает `GoRouter`) и конфиг: наблюдатели (например, аналитика переходов), стартовую локацию, охранников через `redirect`, общий каркас экранов через `ShellRoute` (нижняя навигация и т.п.).

```dart
import 'package:go_router/go_router.dart';

class AppRouter {
  static final navigatorKey =
      GlobalKey<NavigatorState>(debugLabel: 'appNavigatorKey');

  GoRouter get router => _router;

  final GoRouter _router = GoRouter(
    navigatorKey: navigatorKey,
    initialLocation: '/home',
    debugLogDiagnostics: kDebugMode,
    observers: [/* напр. наблюдатель аналитики переходов */],
    redirect: (context, state) {
      // глобальный охранник: вернуть путь редиректа или null, чтобы пропустить
      return null;
    },
    routes: [
      ShellRoute(
        builder: (context, state, child) => AppNavigationShell(child: child),
        routes: [
          GoRoute(
            name: 'home',
            path: '/home',
            builder: (context, state) => const HomeScreen(),
          ),
          // ... остальные разделы; новый экран — новый GoRoute с уникальным name
        ],
      ),
    ],
  );
}
```

Охранник можно держать глобально (как здесь) или на конкретном `GoRoute` — что ближе к правилу.

## Адаптер под контракт

`AppNavigatorImpl` реализует `AppNavigator` через `navigatorKey.currentContext` (расширения go_router). `pop` защищён `canPop`.

```dart
class AppNavigatorImpl implements AppNavigator {
  final GlobalKey<NavigatorState> _navigatorKey;
  AppNavigatorImpl(this._navigatorKey);

  BuildContext get _context => _navigatorKey.currentContext!;

  @override
  void goNamed(String name, {Object? extra}) =>
      _context.goNamed(name, extra: extra);

  @override
  Future<T?> pushNamed<T extends Object?>(String name, {Object? extra}) =>
      _context.pushNamed<T>(name, extra: extra);

  @override
  void replaceNamed(String name, {Object? extra}) =>
      _context.replaceNamed(name, extra: extra);

  @override
  void pop<T extends Object?>([T? result]) {
    if (_context.canPop()) _context.pop<T>(result);
  }
}
```

## Подключение: Provider + Router

`MaterialApp` остаётся сверху (тема, локаль), а `GoRouter` монтируется ниже — голым `Router` в поддереве после сборки DI. `AppNavigator` раздаётся `Provider`-ом над ним; фичи берут его из дерева.

```dart
class AppRouterScope extends StatefulWidget {
  const AppRouterScope({super.key});

  @override
  State<AppRouterScope> createState() => _AppRouterScopeState();
}

class _AppRouterScopeState extends State<AppRouterScope> {
  late final AppRouter _appRouter;
  late final AppNavigator _navigator;

  @override
  void initState() {
    super.initState();
    _appRouter = AppRouter();
    _navigator = AppNavigatorImpl(AppRouter.navigatorKey);
  }

  @override
  Widget build(BuildContext context) {
    final router = _appRouter.router;
    return Provider<AppNavigator>(
      create: (_) => _navigator,
      child: Router(
        backButtonDispatcher: RootBackButtonDispatcher(),
        routeInformationProvider: router.routeInformationProvider,
        routeInformationParser: router.routeInformationParser,
        routerDelegate: router.routerDelegate,
      ),
    );
  }
}
```

Это и есть поддерево приложения после splash-гейта DI — то, что подставляют в точку `_AppRoot` каркаса.

## Структура

```text
common/navigation/
├── navigator/   # контракт AppNavigator + адаптер AppNavigatorImpl
├── router/      # AppRouter: navigatorKey, таблица маршрутов, охранники
├── widgets/     # навигационный UI: shell (нижняя навигация), экраны-обёртки
└── helpers/     # вспомогательное, по необходимости
```

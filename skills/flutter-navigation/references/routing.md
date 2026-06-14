# Настройка роутера — шаблон

Движок по умолчанию — `go_router`; он спрятан за `AppNavigator` и сменяем. `AppNavigator` завязан на дерево (навигатор-ключ → контекст), поэтому его раздают через `Provider` в поддереве, а **не** кладут в DI-контейнер. Маршруты и охранники собраны тут, в фичи не протекают.

## Имена маршрутов

Имена и пути экранов — в одной сущности `AppRoutes`, а не строками-литералами по коду. На неё ссылаются и роутер, и переходы (`goNamed(AppRoutes.home)`); фичам не нужен сам класс-роутер.

```dart
abstract final class AppRoutes {
  static const home = 'home';
  static const homePath = '/home';
  // новый экран — имя и путь сюда
}
```

## Роутер

`AppRouter` держит общий `navigatorKey` (его же получает `GoRouter`) и конфиг: наблюдатели (например, аналитика переходов), стартовую локацию, охранников через `redirect`, общий каркас экранов через `ShellRoute`.

```dart
import 'package:go_router/go_router.dart';

class AppRouter {
  static final navigatorKey =
      GlobalKey<NavigatorState>(debugLabel: 'appNavigatorKey');

  GoRouter get router => _router;

  final GoRouter _router = GoRouter(
    navigatorKey: navigatorKey,
    initialLocation: AppRoutes.homePath,
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
            name: AppRoutes.home,
            path: AppRoutes.homePath,
            builder: (context, state) => const HomeScreen(),
          ),
        ],
      ),
    ],
  );
}
```

`ShellRoute` нужен только когда у группы экранов общий каркас — обычно нижняя навигация (bottom navigation bar): он оборачивает дочерние экраны в постоянный shell. Нет общего каркаса — `GoRoute` кладут прямо в `routes`, без `ShellRoute`. Охранник можно держать глобально (как здесь) или на конкретном `GoRoute`.

## Добавить экран

```dart
// 1) имя и путь — в AppRoutes
static const details = 'details';
static const detailsPath = '/details';

// 2) GoRoute — в routes: внутри ShellRoute, если экран под общим каркасом, иначе сверху
GoRoute(
  name: AppRoutes.details,
  path: AppRoutes.detailsPath,
  builder: (context, state) => const DetailsScreen(),
),

// 3) переход из фичи
context.read<AppNavigator>().goNamed(AppRoutes.details, extra: id);
```

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
├── router/      # AppRouter, AppRoutes (имена/пути), охранники
├── widgets/     # навигационный UI: shell (нижняя навигация), экраны-обёртки
└── helpers/     # вспомогательное, по необходимости
```

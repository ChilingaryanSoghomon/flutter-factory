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

## Вкладки с сохранением состояния

Когда каждая вкладка нижней навигации держит свой стек и помнит позицию — вместо `ShellRoute` берут `StatefulShellRoute.indexedStack` (выбирают одно). Каждая вкладка — отдельный `StatefulShellBranch`; каркас принимает `StatefulNavigationShell` вместо `child` и переключает ветки.

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) =>
      AppNavigationShell(navigationShell: navigationShell),
  branches: [
    StatefulShellBranch(
      routes: [
        GoRoute(
          name: AppRoutes.home,
          path: AppRoutes.homePath,
          builder: (context, state) => const HomeScreen(),
        ),
      ],
    ),
    StatefulShellBranch(
      routes: [
        GoRoute(
          name: AppRoutes.settings,
          path: AppRoutes.settingsPath,
          builder: (context, state) => const SettingsScreen(),
        ),
      ],
    ),
  ],
)
```

Каркас переключает ветки через `goBranch`; повторный тап по активной вкладке возвращает её к начальной локации:

```dart
class AppNavigationShell extends StatelessWidget {
  const AppNavigationShell({required this.navigationShell, super.key});

  final StatefulNavigationShell navigationShell;

  void _goBranch(int index) => navigationShell.goBranch(
        index,
        initialLocation: index == navigationShell.currentIndex,
      );

  @override
  Widget build(BuildContext context) => Scaffold(
        body: navigationShell,
        bottomNavigationBar: NavigationBar(
          selectedIndex: navigationShell.currentIndex,
          onDestinationSelected: _goBranch,
          destinations: const [
            NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
            NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
          ],
        ),
      );
}
```

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

## Deep linking

Внешняя ссылка (`https://example.com/details/123`) открывает нужный экран: go_router разбирает путь по `routes`, платформам надо разрешить перехват. Пути берут из `AppRoutes`.

### Android

`AndroidManifest.xml`, внутри `<activity>` для `.MainActivity`:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="http" android:host="example.com" />
  <data android:scheme="https" />
</intent-filter>
```

`assetlinks.json` хостят по `https://example.com/.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.example.app",
    "sha256_cert_fingerprints": ["SHA256_FINGERPRINT"]
  }
}]
```

### iOS

`Info.plist` — включить штатный обработчик (при стороннем плагине вроде `app_links` ставят `NO`):

```xml
<key>FlutterDeepLinkingEnabled</key>
<true/>
```

`Runner.entitlements` — связанный домен:

```xml
<key>com.apple.developer.associated-domains</key>
<array>
  <string>applinks:example.com</string>
</array>
```

`apple-app-site-association` (без расширения) хостят по `https://example.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "apps": [],
    "details": [{
      "appIDs": ["TEAM_ID.com.example.app"],
      "paths": ["*"],
      "components": [{"/": "/*"}]
    }]
  }
}
```

### Проверка

```bash
# Android
adb shell 'am start -a android.intent.action.VIEW -c android.intent.category.BROWSABLE \
  -d "https://example.com/details/123"' com.example.app

# iOS (запущенный симулятор)
xcrun simctl openurl booted https://example.com/details/123
```

Для web `#` из URL убирает `usePathUrlStrategy()` в `main()` до `runApp`.

## Структура

```text
common/navigation/
├── navigator/   # контракт AppNavigator + адаптер AppNavigatorImpl
├── router/      # AppRouter, AppRoutes (имена/пути), охранники
├── widgets/     # навигационный UI: shell (нижняя навигация), экраны-обёртки
└── helpers/     # вспомогательное, по необходимости
```

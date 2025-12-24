# Fuse Wallet Codebase Instructions

## Project Overview
Fuse Wallet is a Flutter mobile app that provides a white-label ERC-20/ERC-721 wallet built on the Charge Wallet API. It supports features like token trading, NFT management, WalletConnect integration, and on-ramp payments (Ramp Network, Xanpool).

## Architecture Patterns

### Redux State Management
The app uses Redux for global state with three main state branches:
- `UserState` - authentication, locale, currency
- `CashWalletState` - tokens, transaction history, wallet balances
- `NftsState` - NFT metadata and collections
- `SwapState` - swap-related trading state

**Key files**: [lib/models/app_state.dart](lib/models/app_state.dart), [lib/redux/store.dart](lib/redux/store.dart)

**Data flow**: Actions → Thunks → API calls → Reducers → State → UI via `StoreConnector`

When dispatching async operations, use thunks (see [lib/redux/actions/user_actions.dart](lib/redux/actions/user_actions.dart#L200) for pattern). Redux state is persisted to secure storage automatically via `redux_persist`.

### Dependency Injection
`GetIt` + `@injectable` generates DI config. Dependencies are pre-resolved at app startup in `configureDependencies()` ([lib/common/di/di.dart](lib/common/di/di.dart)). Access injected services via `getIt<ServiceType>()`.

**Key modules**: Firebase services, Dio HTTP client, Router, Analytics, ChargeApi SDK, SharedPreferences.

### Routing
Uses `AutoRoute` with declarative route definitions ([lib/common/router/routes.dart](lib/common/router/routes.dart)). Routes are organized hierarchically with tab-based navigation. AuthGuard validates wallet creation before accessing MainPage.

## Critical Development Workflows

### Code Generation
This project heavily relies on generated code via `build_runner`:

```bash
flutter pub get
flutter pub run build_runner build --delete-conflicting-outputs
```

Run after changes to:
- `@freezed` classes (models with `.freezed.dart`)
- `@JsonSerializable` classes
- `@injectable` DI configurations
- `@MaterialAutoRouter` routes

**Generated files to never edit**: `*.freezed.dart`, `*.g.dart`, `routes.gr.dart`, `di.config.dart`

### Environment Setup
1. Copy `.env` from `environment/.env.example` - required for Ramp and Xanpool API keys
2. Android: Create `android/key.properties` with signing config
3. iOS: Firebase GoogleService-Info.plist auto-configured in Xcode

### Building & Running
```bash
flutter pub get
flutter run -t lib/main.dart
flutter build apk --release  # Android
flutter build ios --release   # iOS
```

## Project-Specific Conventions

### Widget Structure
- Extend `SfWidget` (Stateful) or `SlWidget` (Stateless) for alert/exception handling helpers
- Use `StoreConnector` with ViewModel pattern for Redux-connected widgets
- ViewModels use `Equatable` for `distinct: true` optimization ([lib/redux/viewsmodels/home.dart](lib/redux/viewsmodels/home.dart))

### Model Definition
Models use `freezed` with custom converters for Redux serialization. Example pattern:
```dart
@freezed
class MyState with _$MyState {
  @MyStateConverter()
  const factory MyState({required List<String> items}) = _MyState;
  factory MyState.initial() => MyState(items: []);
  factory MyState.fromJson(dynamic json) => _$MyStateFromJson(json);
}
```

### Error Handling
- Use `Alerts` service ([lib/utils/alerts/alerts.dart](lib/utils/alerts/alerts.dart)) for user-facing errors via `throwAlert()` in widgets
- Firebase Crashlytics captures unhandled exceptions automatically (disabled in debug mode)
- Log errors to `logger` instance for debugging

### Internationalization (i18n)
App supports 14+ languages via ARB files ([lib/l10n/](lib/l10n/)). Access strings via `I10n.of(context).stringKey`. Add new keys to all `.arb` files.

## Integration Points

### Charge Wallet SDK
Accessed via `chargeApi` service ([lib/services.dart](lib/services.dart)). Handles smart wallet contract interactions, transaction broadcasting.

### Firebase Integration
- **Analytics**: Track events using `Analytics.track()` with predefined events from [lib/constants/analytics_events.dart](lib/constants/analytics_events.dart)
- **Auth**: Phone-based Firebase Auth for user verification
- **Crashlytics**: Auto-configured exception reporting
- **Performance**: Monitoring enabled in release builds

### Network Services
Dio HTTP client with logging middleware (debug mode only). Base setup in DI, used by SDK and custom API calls.

## Common Patterns & Gotchas

- **Never mutate Redux state**: Use freezed's copyWith pattern
- **Action dispatch**: Always dispatch from thunks/middleware, not UI directly for async ops
- **Router navigation**: Use `context.navigateTo()` from `auto_route` extension
- **Secure storage**: Phone numbers and credentials stored in FlutterSecureStorage, not SharedPreferences
- **Feature organization**: Each feature (home, wallet, swap) has its own router and screens subdirectories
- **Analytics tracking**: Include both event name AND properties (userId, timestamp auto-added)

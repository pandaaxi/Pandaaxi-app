# XBoard API Integration Map

A quick map of how the app talks to the XBoard backend and where to look in the codebase.

## Configuration Inputs
- `assets/config/xboard.config.yaml`: local entry point; defines provider name, remote config URLs, HTTP settings (UA, cert, obfuscation) and logging. Loaded by `lib/xboard/config/utils/config_file_loader.dart`.
- Remote `config.json`: downloaded by `lib/xboard/config/xboard_config.dart` and unpacked into panel/proxy/ws/update/subscription entries.
- Domain racing: `lib/xboard/infrastructure/network/domain_racing_service.dart` selects the fastest panel (and optional proxy) using the URLs from `config.json`.

## Initialization Flow (startup happy path)
1. `lib/xboard/features/domain_status/providers/domain_status_provider.dart` calls `DomainStatusService.checkDomainStatus()` to race panel domains and mark one as ready.
2. `lib/xboard/features/initialization/providers/initialization_provider.dart` waits for a ready domain, updates UI state, and triggers SDK wiring.
3. `lib/xboard/adapter/initialization/sdk_provider.dart` builds `HttpConfig` (User-Agent + obfuscation prefix + cert pinning), then calls  
   `XBoardSDK.instance.initialize(fastestUrl, panelType, proxyUrl, httpConfig)`.
4. The SDK now exposes typed API clients under `XBoardSDK.instance.*`.

## HTTP Stack
- Client: `lib/xboard/infrastructure/http/xboard_http_client.dart` wraps Dio with logging and retry.
- User-Agent and TLS: `lib/xboard/infrastructure/http/user_agent_config.dart` + `ConfigFileLoaderHelper` read UA strings, obfuscation prefixes, and CA paths from `xboard.config.yaml`.
- SDK config: `HttpConfig` is constructed in `sdk_provider.dart` and passed to `XBoardSDK` so every module (auth/order/subscription/…) shares the same HTTP options.

## API Surface Used in the App
| Area | SDK calls | Entry points in repo |
| --- | --- | --- |
| Auth | `hasToken`, `loginWithCredentials`, `auth.register`, `auth.sendEmailVerifyCode`, `auth.forgotPassword`, `logout` | `lib/xboard/features/auth/providers/xboard_user_provider.dart`, auth pages |
| Subscription | `subscription.getSubscription()` | `lib/xboard/features/subscription/services/{subscription_downloader,encrypted_subscription_service,concurrent_subscription_service}.dart`, `xboard_user_provider.dart` |
| Config | `config.getConfig()` | `lib/xboard/features/auth/providers/config_provider.dart` |
| Orders & Payment | `order.getOrders()`, `order.createOrder(...)`, `order.cancelOrder(...)`, `order.checkoutOrder(...)` | `lib/xboard/features/payment/providers/xboard_payment_provider.dart`, payment pages/widgets |
| Invite & Commission | `invite.generateInviteCode()`, `invite.withdrawCommission(...)`, `invite.transferCommissionToBalance(...)`, plus list/detail fetchers in generated providers | `lib/xboard/features/invite/providers/invite_provider.dart` |
| Token reuse | `getToken()` | `lib/xboard/features/online_support/services/service_config.dart` (customer support endpoints need the same auth) |

## State & Persistence Around the APIs
- Persistent storage: `lib/xboard/services/storage/xboard_storage_service.dart` caches auth email, user profile, and subscription info for quick rehydrate.
- Auth/session state: `XBoardUserAuthNotifier` in `lib/xboard/features/auth/providers/xboard_user_provider.dart` orchestrates login/refresh/logout, then saves/reads data via the storage service.
- Domain models: `lib/xboard/domain/models/*.dart` hold the app-facing shapes; mappers live alongside providers (for example `_mapSubscription` in `xboard_user_provider.dart`).
- Config provider: `lib/xboard/config/xboard_config.dart` exposes computed URLs (`panel`, `proxy`, `ws`, `update`, `subscription`) and racing stats for downstream services.

## High-Level Data Flow
```
assets/config/xboard.config.yaml
      ↓
XBoardConfig (loads remote config.json, runs domain racing)
      ↓
initializationProvider → xboardSdkProvider → XBoardSDK.instance
      ↓
SDK modules (auth / subscription / order / invite / config)
      ↓
Feature notifiers & pages (Riverpod) → UI
      ↓
XBoardStorageService caches auth + subscription data for reuse
```

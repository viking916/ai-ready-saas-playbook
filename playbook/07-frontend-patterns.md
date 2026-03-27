# 07 - Frontend Patterns (Flutter)

## Why

Flutter with Riverpod provides strong type safety, reactive state management, and code sharing between web and mobile. Go Router handles declarative routing with auth guards. Dio with interceptors provides a clean API client pattern.

---

## Riverpod State Management (AsyncNotifier Pattern)

```dart
// app-client/lib/shared/providers/auth_provider.dart

/// Provider for FirebaseAuth instance. Override in tests with MockFirebaseAuth.
final firebaseAuthProvider = Provider<fb.FirebaseAuth>((ref) {
  return fb.FirebaseAuth.instance;
});

/// Holds the MFA resolver during multi-factor sign-in flow.
final mfaResolverProvider = StateProvider<fb.MultiFactorResolver?>((ref) => null);

class AuthState {
  final User? user;
  final bool isAuthenticated;
  final bool isLoading;
  final String? error;

  const AuthState({
    this.user,
    this.isAuthenticated = false,
    this.isLoading = false,
    this.error,
  });

  AuthState copyWith({
    User? user,
    bool? isAuthenticated,
    bool? isLoading,
    String? error,
  }) {
    return AuthState(
      user: user ?? this.user,
      isAuthenticated: isAuthenticated ?? this.isAuthenticated,
      isLoading: isLoading ?? this.isLoading,
      error: error,
    );
  }
}

class AuthNotifier extends AsyncNotifier<AuthState> {
  fb.FirebaseAuth get _firebaseAuth => ref.read(firebaseAuthProvider);

  @override
  Future<AuthState> build() async {
    final fbUser = _firebaseAuth.currentUser;

    if (fbUser != null) {
      if (fbUser.emailVerified) {
        try {
          // Force token refresh so the ID token contains email_verified: true
          await fbUser.getIdToken(true);
          final user = await _completeRegistration();
          PushNotificationService.instance.registerToken(ref.read(apiClientProvider));
          await _loadPendingInviteToken();
          return AuthState(user: user, isAuthenticated: true);
        } catch (e) {
          await _firebaseAuth.signOut();
          return const AuthState();
        }
      } else {
        return const AuthState(isAuthenticated: true);
      }
    }

    return const AuthState();
  }

  /// Invalidate all cached data providers to prevent data leakage between users
  void _invalidateAllCachedData() {
    ref.invalidate(callsProvider);
    ref.invalidate(recentCallsProvider);
    ref.invalidate(upcomingCallsProvider);
    ref.invalidate(updatesProvider);
    ref.invalidate(elderlyUsersProvider);
    ref.invalidate(schedulesProvider);
    ref.invalidate(subscriptionPlansProvider);
    ref.invalidate(selectedElderlyUserProvider);
  }

  Future<void> login({required String email, required String password}) async {
    state = const AsyncValue.loading();
    _invalidateAllCachedData();

    try {
      await _firebaseAuth.signInWithEmailAndPassword(email: email, password: password);
      final fbUser = _firebaseAuth.currentUser;

      if (fbUser != null && fbUser.emailVerified) {
        final user = await _completeRegistration();
        _invalidateAllCachedData();
        await _loadPendingInviteToken();
        state = AsyncValue.data(AuthState(user: user, isAuthenticated: true));

        try {
          PushNotificationService.instance.registerToken(ref.read(apiClientProvider));
        } catch (_) {}
      } else {
        state = const AsyncValue.data(AuthState(isAuthenticated: true));
      }
    } on fb.FirebaseAuthMultiFactorException catch (e) {
      ref.read(mfaResolverProvider.notifier).state = e.resolver;
      state = const AsyncValue.data(AuthState(error: 'MFA_REQUIRED'));
    } catch (e) {
      state = AsyncValue.data(AuthState(error: _parseError(e)));
      rethrow;
    }
  }
}

final authStateProvider = AsyncNotifierProvider<AuthNotifier, AuthState>(() {
  return AuthNotifier();
});
```

**Key patterns:**
- `build()` runs on initialization and can be re-triggered via `ref.invalidate()`
- Use `ref.read()` for one-time reads, `ref.watch()` for reactive dependencies
- Override `firebaseAuthProvider` in tests with `MockFirebaseAuth`
- Invalidate all data providers on logout to prevent cross-user data leakage

---

## API Client with Dio (Complete)

```dart
// app-client/lib/core/api/api_client.dart

import 'package:dio/dio.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';

import '../constants/api_endpoints.dart';
import '../security/secure_storage.dart';

final dioProvider = Provider<Dio>((ref) {
  final dio = Dio(
    BaseOptions(
      baseUrl: ApiEndpoints.baseUrl,
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ),
  );

  // SECURITY: Disable request/response body logging to prevent data leakage
  if (kDebugMode) {
    dio.interceptors.add(
      PrettyDioLogger(
        requestHeader: false,
        requestBody: false,
        responseBody: false,
        responseHeader: false,
        error: true,
        compact: true,
      ),
    );
  }

  dio.interceptors.add(FirebaseAuthInterceptor(dio: dio));
  dio.interceptors.add(SessionActivityInterceptor());

  return dio;
});

final apiClientProvider = Provider<ApiClient>((ref) {
  return ApiClient(ref.read(dioProvider));
});

class ApiClient {
  final Dio _dio;
  ApiClient(this._dio);

  Future<Response<T>> get<T>(String path, {Map<String, dynamic>? queryParameters, Options? options}) async {
    return _dio.get<T>(path, queryParameters: queryParameters, options: options);
  }

  Future<Response<T>> post<T>(String path, {dynamic data, Map<String, dynamic>? queryParameters, Options? options}) async {
    return _dio.post<T>(path, data: data, queryParameters: queryParameters, options: options);
  }

  Future<Response<T>> put<T>(String path, {dynamic data, Map<String, dynamic>? queryParameters, Options? options}) async {
    return _dio.put<T>(path, data: data, queryParameters: queryParameters, options: options);
  }

  Future<Response<T>> patch<T>(String path, {dynamic data, Map<String, dynamic>? queryParameters, Options? options}) async {
    return _dio.patch<T>(path, data: data, queryParameters: queryParameters, options: options);
  }

  Future<Response<T>> delete<T>(String path, {dynamic data, Map<String, dynamic>? queryParameters, Options? options}) async {
    return _dio.delete<T>(path, data: data, queryParameters: queryParameters, options: options);
  }
}

/// Interceptor that attaches Firebase ID tokens to every request.
/// Firebase SDK handles token refresh internally (tokens are 1 hour, auto-refreshed).
class FirebaseAuthInterceptor extends Interceptor {
  final Dio dio;
  bool _isRetrying = false;

  FirebaseAuthInterceptor({required this.dio});

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final user = FirebaseAuth.instance.currentUser;
    if (user != null) {
      try {
        final token = await user.getIdToken();
        options.headers['Authorization'] = 'Bearer $token';
      } catch (_) {
        // If token retrieval fails, proceed without token
      }
    }
    handler.next(options);
  }

  @override
  void onError(DioException err, ErrorInterceptorHandler handler) async {
    // SECURITY: Only retry 401 once to prevent infinite loop
    if (err.response?.statusCode == 401 && !_isRetrying) {
      final user = FirebaseAuth.instance.currentUser;
      if (user != null) {
        _isRetrying = true;
        try {
          final freshToken = await user.getIdToken(true);
          final options = err.requestOptions;
          options.headers['Authorization'] = 'Bearer $freshToken';
          final response = await dio.fetch(options);
          return handler.resolve(response);
        } catch (_) {
        } finally {
          _isRetrying = false;
        }
      }
    }
    handler.next(err);
  }
}

/// SECURITY: Tracks user activity timestamp on each successful API response.
class SessionActivityInterceptor extends Interceptor {
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    SecureStorage.updateLastActivity();
    handler.next(response);
  }
}
```

**Key pattern:** The `_isRetrying` flag prevents infinite loops. Without it, a suspended account would: get 401 -> refresh token (succeeds, since Firebase doesn't know about suspension) -> retry request -> get 401 again -> infinite loop.

---

## Secure Storage

```dart
// app-client/lib/core/security/secure_storage.dart

import 'package:flutter_secure_storage/flutter_secure_storage.dart';

class SecureStorage {
  static const _storage = FlutterSecureStorage(
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock),
  );

  // Test-only in-memory override
  static Map<String, String>? _mockStorage;

  static void setMockStorage([Map<String, String>? initial]) {
    _mockStorage = initial ?? {};
  }

  static void clearMockStorage() {
    _mockStorage = null;
  }

  static Future<void> _write(String key, String value) async {
    if (_mockStorage != null) { _mockStorage![key] = value; return; }
    await _storage.write(key: key, value: value);
  }

  static Future<String?> _read(String key) async {
    if (_mockStorage != null) return _mockStorage![key];
    return _storage.read(key: key);
  }

  static Future<void> _delete(String key) async {
    if (_mockStorage != null) { _mockStorage!.remove(key); return; }
    await _storage.delete(key: key);
  }

  // Keys
  static const _keyAuthToken = 'auth_token';
  static const _keyRefreshToken = 'refresh_token';
  static const _keyLastActivity = 'last_activity';
  static const _keyPendingRegistration = 'pending_registration';

  // Auth tokens
  static Future<void> saveAuthToken(String token) => _write(_keyAuthToken, token);
  static Future<String?> getAuthToken() => _read(_keyAuthToken);

  // Session activity tracking (for auto-logout)
  static Future<void> updateLastActivity() =>
      _write(_keyLastActivity, DateTime.now().toIso8601String());

  static Future<bool> isSessionExpired({Duration timeout = const Duration(minutes: 30)}) async {
    final lastActivity = await getLastActivity();
    if (lastActivity == null) return true;
    return DateTime.now().difference(lastActivity) > timeout;
  }

  // Pending registration data (encrypted, not in SharedPreferences)
  static Future<void> savePendingRegistration(String json) => _write(_keyPendingRegistration, json);
  static Future<String?> getPendingRegistration() => _read(_keyPendingRegistration);
  static Future<void> removePendingRegistration() => _delete(_keyPendingRegistration);

  // Clear all on logout
  static Future<void> clearAll() async {
    if (_mockStorage != null) { _mockStorage!.clear(); return; }
    await _storage.deleteAll();
  }
}
```

---

## Design System (Theme)

```dart
// app-client/lib/core/theme/app_theme.dart

class AppColors {
  // Primary -- Black for actions, buttons, interactive elements
  static const Color primary = Color(0xFF111827);        // Gray 900
  static const Color primaryLight = Color(0xFF374151);   // Gray 700
  static const Color primarySurface = Color(0xFFF3F4F6); // Gray 100

  // Brand -- Teal for logo/branding only
  static const Color brand = Color(0xFF0D9488);          // Teal 600

  // Warm accent for highlights
  static const Color accent = Color(0xFFF97316);         // Orange 500
  static const Color accentSurface = Color(0xFFFFF7ED);  // Orange 50

  // Surface & Background
  static const Color surface = Color(0xFFFFFFFF);
  static const Color background = Color(0xFFFAFAFA);
  static const Color cardBackground = Color(0xFFFFFFFF);

  // Text
  static const Color textPrimary = Color(0xFF1F2937);    // Gray 800
  static const Color textSecondary = Color(0xFF4B5563);  // Gray 600
  static const Color textTertiary = Color(0xFF6B7280);   // Gray 500 (WCAG AA)

  // Border
  static const Color border = Color(0xFFE5E7EB);         // Gray 200
  static const Color borderLight = Color(0xFFF3F4F6);    // Gray 100

  // Semantic
  static const Color error = Color(0xFFDC2626);
  static const Color warning = Color(0xFFD97706);
  static const Color success = Color(0xFF059669);
  static const Color info = Color(0xFF0284C7);

  // Mood/Sentiment
  static const Color moodPositive = Color(0xFF10B981);
  static const Color moodNeutral = Color(0xFFF59E0B);
  static const Color moodConcerning = Color(0xFFEF4444);
}

class AppSpacing {
  static const double space1 = 4.0;
  static const double space2 = 8.0;
  static const double space3 = 12.0;
  static const double space4 = 16.0;
  static const double space6 = 24.0;
  static const double space8 = 32.0;
  static const double space12 = 48.0;

  static const double xs = space1;
  static const double sm = space2;
  static const double md = space4;
  static const double lg = space6;
  static const double xl = space8;
}

class AppRadius {
  static const double sm = 4.0;
  static const double md = 8.0;
  static const double lg = 12.0;
  static const double xl = 16.0;
  static const double full = 999.0;
}

class AppFontSize {
  static const double xs = 12.0;
  static const double sm = 14.0;
  static const double base = 16.0;
  static const double lg = 18.0;
  static const double xl = 20.0;
  static const double xxl = 24.0;
  static const double display = 36.0;
}
```

The `AppTheme.lightTheme` getter configures Material 3 with this color system, setting button styles, input decorations, card themes, navigation bars, etc.

---

## Go Router with Shell Routes

```dart
routes: [
  // Public routes (no shell)
  GoRoute(path: '/login', builder: (_, __) => const LoginScreen()),
  GoRoute(path: '/register', builder: (_, __) => const RegisterScreen()),
  GoRoute(path: '/verify-email', builder: (_, __) => const EmailVerificationScreen()),

  // Authenticated routes with persistent navigation
  ShellRoute(
    navigatorKey: _shellNavigatorKey,
    builder: (_, __, child) => MainScaffold(child: child),
    routes: [
      GoRoute(path: '/home', builder: (_, __) => const HomeScreen()),
      GoRoute(path: '/settings', builder: (_, __) => const SettingsScreen()),
      GoRoute(path: '/notifications', builder: (_, __) => const NotificationsScreen()),
    ],
  ),

  // Separate shell for facility admin
  ShellRoute(
    navigatorKey: _facilityShellNavigatorKey,
    builder: (_, __, child) => FacilityMainScaffold(child: child),
    routes: [
      GoRoute(path: '/facility/dashboard', builder: (_, __) => const FacilityDashboardScreen()),
      GoRoute(path: '/facility/residents', builder: (_, __) => const FacilityResidentsScreen()),
    ],
  ),
],
```

---

## Stripe Integration on Client

```dart
// Use WidgetsBindingObserver to refresh subscription state when user returns from Stripe portal
class _MyScreenState extends ConsumerState<MyScreen> with WidgetsBindingObserver {
  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    if (state == AppLifecycleState.resumed) {
      ref.invalidate(subscriptionPlansProvider);
    }
  }
}
```

---

## Feature Module Structure

```
lib/
  features/
    auth/
      screens/
        login_screen.dart
        register_screen.dart
        email_verification_screen.dart
    home/
      screens/
        home_screen.dart
      widgets/
        dashboard_card.dart
    settings/
      screens/
        settings_screen.dart
        profile_settings_screen.dart
  shared/
    models/
      user.dart
    providers/
      auth_provider.dart
    widgets/
      main_scaffold.dart
  core/
    api/
      api_client.dart
    router/
      app_router.dart
    theme/
      app_theme.dart
    constants/
      api_endpoints.dart
    utils/
      error_utils.dart
    security/
      secure_storage.dart
```

---

## Gotchas

1. **Use relative imports in Dart**, not `package:myapp/` style.

2. **Firebase `getIdToken()` returns cached tokens.** Call `getIdToken(true)` to force a refresh after email verification.

3. **Go Router redirect evaluates on every navigation.** Keep it fast -- no async operations.

4. **`SharedPreferences` is not secure.** Use `flutter_secure_storage` for tokens and sensitive data.

5. **`didChangeAppLifecycleState` doesn't fire on web** for tab switches.

---

## Checklist

- [ ] Set up Riverpod with `ProviderScope` at app root
- [ ] Create `AsyncNotifier` pattern for auth state
- [ ] Implement Dio-based API client with Firebase auth interceptor
- [ ] Add token refresh retry (with `_isRetrying` flag)
- [ ] Centralize API endpoints in constants class
- [ ] Implement Go Router with `redirect` for auth guards
- [ ] Use `ShellRoute` for persistent navigation
- [ ] Add session activity tracking interceptor
- [ ] Implement `ErrorUtils` for user-friendly error messages
- [ ] Set up feature module directory structure
- [ ] Use `WidgetsBindingObserver` to refresh state on return from external flows

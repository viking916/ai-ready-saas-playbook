# 01 - Authentication (Firebase Auth)

## Why

Firebase Auth handles the hard parts of authentication (password hashing, token management, OAuth providers, MFA) while keeping your backend stateless. The backend never stores passwords -- Firebase owns the credential lifecycle, and the backend only verifies Firebase ID tokens.

**Key design decisions:**
- Firebase is the **primary** auth system; a legacy JWT fallback exists for migration
- Backend user creation is **deferred** until email verification completes (prevents orphan users from abandoned signups)
- Auto-linking by email lets you migrate existing users to Firebase without disrupting them
- Google Sign-In users skip email verification entirely (Google already verified the email)

---

## Backend Token Verification Middleware

The `authenticate` middleware tries Firebase first, then falls back to legacy JWT. This is the most critical piece of backend security.

```typescript
// engine/src/middleware/auth.ts

import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { env } from '../config/env.js';
import { getFirebaseAuth } from '../config/firebase.js';
import { UnauthorizedError } from './errorHandler.js';
import { prisma } from '../index.js';
import { tokenService } from '../services/token.service.js';
import { sessionService } from '../services/session.service.js';
import { breachDetectionService } from '../services/breach-detection.service.js';

export interface AuthUser {
  userId: string;
  email: string;
  firebaseUid?: string;
}

// Keep backwards-compatible alias
export type JwtPayload = AuthUser;

declare global {
  namespace Express {
    interface Request {
      user?: AuthUser;
    }
  }
}

/**
 * Try to verify a token as a Firebase ID token, then look up the PG user.
 * Returns AuthUser on success, null on failure.
 */
async function verifyFirebaseToken(token: string): Promise<AuthUser | null> {
  try {
    const decoded = await getFirebaseAuth().verifyIdToken(token);

    // Try lookup by firebaseUid first
    let user = await prisma.user.findUnique({
      where: { firebaseUid: decoded.uid },
    });

    // Migration path: if no user by UID, try email match and auto-link
    // SECURITY: Only auto-link if Firebase email is verified to prevent account takeover
    if (!user && decoded.email && decoded.email_verified) {
      user = await prisma.user.findUnique({
        where: { email: decoded.email },
      });
      if (user) {
        await prisma.user.update({
          where: { id: user.id },
          data: { firebaseUid: decoded.uid },
        });
      }
    }

    if (!user || user.deletedAt) return null;

    // SECURITY: Reject tokens issued before tokensRevokedAt (bulk revocation)
    if (user.tokensRevokedAt && decoded.iat) {
      const tokenIssuedAt = new Date(decoded.iat * 1000);
      if (tokenIssuedAt < user.tokensRevokedAt) return null;
    }

    return { userId: user.id, email: user.email, firebaseUid: decoded.uid };
  } catch {
    return null;
  }
}

/**
 * Try to verify a token as a legacy JWT, then confirm user still exists.
 * Returns AuthUser on success, null on failure.
 */
async function verifyLegacyJwt(token: string): Promise<AuthUser | null> {
  if (!env.JWT_SECRET) return null;
  try {
    const decoded = jwt.verify(token, env.JWT_SECRET) as { userId: string; email: string; iat?: number };

    // SECURITY: Verify user still exists and is not deleted
    const user = await prisma.user.findUnique({
      where: { id: decoded.userId },
    });
    if (!user || user.deletedAt) return null;

    // SECURITY: Reject tokens issued before tokensRevokedAt (bulk revocation)
    if (user.tokensRevokedAt && decoded.iat) {
      const tokenIssuedAt = new Date(decoded.iat * 1000);
      if (tokenIssuedAt < user.tokensRevokedAt) return null;
    }

    return { userId: decoded.userId, email: decoded.email };
  } catch {
    return null;
  }
}

export async function authenticate(req: Request, _res: Response, next: NextFunction): Promise<void> {
  // Idempotent: skip if already authenticated (e.g., from router-level middleware)
  if (req.user) return next();

  try {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedError('No token provided');
    }

    const token = authHeader.split(' ')[1];

    // Check if token has been revoked (SECURITY: session invalidation)
    const revoked = await tokenService.isRevoked(token);
    if (revoked) {
      throw new UnauthorizedError('Token has been revoked');
    }

    // Try Firebase first, then fall back to legacy JWT
    const firebaseUser = await verifyFirebaseToken(token);
    if (firebaseUser) {
      // SECURITY: Check session timeout (inactivity-based auto-logout)
      if (sessionService.isSessionExpired(firebaseUser.userId)) {
        sessionService.clearSession(firebaseUser.userId);
        throw new UnauthorizedError('Session expired due to inactivity');
      }
      sessionService.trackActivity(firebaseUser.userId);
      breachDetectionService.recordAccess(firebaseUser.userId, req.path);
      req.user = firebaseUser;
      return next();
    }

    const jwtUser = await verifyLegacyJwt(token);
    if (jwtUser) {
      // SECURITY: Check session timeout (inactivity-based auto-logout)
      if (sessionService.isSessionExpired(jwtUser.userId)) {
        sessionService.clearSession(jwtUser.userId);
        throw new UnauthorizedError('Session expired due to inactivity');
      }
      sessionService.trackActivity(jwtUser.userId);
      breachDetectionService.recordAccess(jwtUser.userId, req.path);
      req.user = jwtUser;
      return next();
    }

    // SECURITY: Track failed authentication for breach detection
    breachDetectionService.recordFailedAuth(req.ip || 'unknown');
    throw new UnauthorizedError('Invalid token');
  } catch (error) {
    if (error instanceof UnauthorizedError) {
      return next(error);
    }
    breachDetectionService.recordFailedAuth(req.ip || 'unknown');
    next(new UnauthorizedError('Invalid token'));
  }
}

export async function optionalAuth(req: Request, _res: Response, next: NextFunction): Promise<void> {
  try {
    const authHeader = req.headers.authorization;

    if (authHeader && authHeader.startsWith('Bearer ')) {
      const token = authHeader.split(' ')[1];

      // SECURITY: Check token revocation even for optional auth
      const revoked = await tokenService.isRevoked(token);
      if (revoked) return next(); // Continue as unauthenticated

      const firebaseUser = await verifyFirebaseToken(token);
      if (firebaseUser) {
        req.user = firebaseUser;
        return next();
      }

      const jwtUser = await verifyLegacyJwt(token);
      if (jwtUser) {
        req.user = jwtUser;
        return next();
      }
    }

    next();
  } catch {
    // Token invalid, continue without user
    next();
  }
}
```

**Key patterns:**
- `if (req.user) return next()` -- makes the middleware **idempotent** (safe to apply at both router and route level)
- Auto-linking only happens when `email_verified === true` -- prevents account takeover via unverified email
- `tokensRevokedAt` enables bulk revocation (password change, account suspension) without storing every individual token
- `sessionService` and `breachDetectionService` are in-memory (acceptable for Cloud Run single-instance)

---

## Token Revocation

Individual token revocation uses SHA-256 hashing -- never store raw tokens:

```typescript
// engine/src/services/token.service.ts

class TokenService {
  hashToken(token: string): string {
    return crypto.createHash('sha256').update(token).digest('hex');
  }

  async revokeToken(token: string, userId: string, reason: RevocationReason = 'LOGOUT'): Promise<void> {
    const tokenHash = this.hashToken(token);
    const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 days

    try {
      await prisma.revokedToken.create({
        data: { tokenHash, userId, reason, expiresAt },
      });
    } catch (error: any) {
      // Ignore unique constraint violation (token already revoked)
      if (error?.code !== 'P2002') throw error;
    }
  }

  async isRevoked(token: string): Promise<boolean> {
    const tokenHash = this.hashToken(token);
    const revoked = await prisma.revokedToken.findUnique({
      where: { tokenHash },
    });
    return !!revoked;
  }

  // Bulk revocation: sets tokensRevokedAt on User -- auth middleware rejects
  // any token with iat before this timestamp
  async revokeAllForUser(userId: string): Promise<void> {
    await prisma.user.update({
      where: { id: userId },
      data: { tokensRevokedAt: new Date() },
    });
  }
}
```

---

## Session Timeout (Security)

In-memory session tracking enforces 30-minute inactivity timeout:

```typescript
// engine/src/services/session.service.ts

class SessionService {
  private lastActivity = new Map<string, number>();
  private timeoutMs: number;

  constructor() {
    this.timeoutMs = (env.SESSION_TIMEOUT_MINUTES || 30) * 60 * 1000;
  }

  trackActivity(userId: string): void {
    this.lastActivity.set(userId, Date.now());
  }

  isSessionExpired(userId: string): boolean {
    const last = this.lastActivity.get(userId);
    if (!last) return false; // First request -- not expired
    return Date.now() - last > this.timeoutMs;
  }

  clearSession(userId: string): void {
    this.lastActivity.delete(userId);
  }

  // Periodic cleanup to prevent memory leak
  cleanup(): void {
    const staleThreshold = Date.now() - this.timeoutMs * 2;
    for (const [userId, timestamp] of this.lastActivity) {
      if (timestamp < staleThreshold) {
        this.lastActivity.delete(userId);
      }
    }
  }
}

// Cleanup stale sessions every 15 minutes
setInterval(() => sessionService.cleanup(), 15 * 60 * 1000).unref();
```

**Why in-memory is acceptable:** Cloud Run instance restart = all sessions expire (more secure, not less). No sensitive data is stored -- just userId to timestamp mapping.

---

## Backend Auth Routes

### POST /auth/sync/new -- Create new user after Firebase signup

```typescript
// engine/src/routes/auth.routes.ts

const syncNewSchema = z.object({
  name: z.string().trim().min(2, 'Name is required').max(100),
  relationship: z.string().trim().max(100).default('Family'),
  phone: z.string().trim().max(20).optional(),
});

authRouter.post('/sync/new', authRateLimiter, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedError('No token provided');
    }

    const token = authHeader.split(' ')[1];
    const decoded = await getFirebaseAuth().verifyIdToken(token);

    if (!decoded.email) {
      throw new BadRequestError('Firebase user must have an email');
    }

    // SECURITY: Require email verification before creating any account
    if (!decoded.email_verified) {
      throw new BadRequestError('Email must be verified before creating an account');
    }

    const body = syncNewSchema.parse(req.body);

    // Check if user already exists (race condition guard)
    const existing = await prisma.user.findUnique({
      where: { email: decoded.email },
    });
    if (existing) {
      // SECURITY: Only auto-link if Firebase email is verified
      if (!existing.firebaseUid && decoded.email_verified) {
        await prisma.user.update({
          where: { id: existing.id },
          data: { firebaseUid: decoded.uid },
        });
      }
      // Return existing user data (redirect to sync)
      // ... fetch and return full user data
    }

    // Create new user + family member + trial subscription in a transaction
    const now = new Date();
    const result = await prisma.$transaction(async (tx) => {
      const user = await tx.user.create({
        data: {
          firebaseUid: decoded.uid,
          email: decoded.email!,
          name: body.name,
          phone: body.phone,
        },
      });

      const familyMember = await tx.familyMember.create({
        data: {
          userId: user.id,
          relationship: body.relationship,
        },
      });

      const subscription = await tx.subscription.create({
        data: {
          familyMemberId: familyMember.id,
          tier: 'TRIAL',
          status: 'ACTIVE',
          trialStartDate: now,
          trialEndDate: new Date(now.getTime() + 14 * 24 * 60 * 60 * 1000),
          trialCallsUsed: 0,
          trialCallsLimit: 3,
        },
      });

      return { user, familyMember, subscription };
    });

    // Send welcome email (fire-and-forget)
    notificationService.notifyFamilyWelcome(result.user.email, result.user.name);

    res.status(201).json({
      id: result.user.id,
      email: result.user.email,
      name: result.user.name,
      phone: result.user.phone,
      isAdmin: checkIsAdmin(result.user.email),
      familyMember: {
        id: result.familyMember.id,
        relationship: result.familyMember.relationship,
        receiveNotifications: result.familyMember.receiveNotifications,
        receiveEmails: result.familyMember.receiveEmails,
        elderlyUsers: [],
        subscription: result.subscription,
      },
      adminFacility: null,
    });
  } catch (error) {
    next(error);
  }
});
```

### POST /auth/sync -- Sync existing user on login

```typescript
authRouter.post('/sync', authenticate, async (req: Request, res: Response, next: NextFunction) => {
  try {
    let user = await prisma.user.findUnique({
      where: { id: req.user!.userId },
      include: {
        familyMember: {
          include: {
            elderlyUsers: {
              where: { deletedAt: null },
              select: { id: true, preferredName: true, phoneNumber: true, status: true, lastCallAt: true },
            },
            subscription: true,
          },
        },
        adminFacility: true,
      },
    });

    if (!user || user.deletedAt) {
      throw new UnauthorizedError('User not found');
    }

    // Safety net: create trial subscription if family member has never had one
    if (user.familyMember && !user.familyMember.subscription) {
      const existingCount = await prisma.subscription.count({
        where: { familyMemberId: user.familyMember.id },
      });
      if (existingCount === 0) {
        const now = new Date();
        const subscription = await prisma.subscription.create({
          data: {
            familyMemberId: user.familyMember.id,
            tier: 'TRIAL',
            status: 'ACTIVE',
            trialStartDate: now,
            trialEndDate: new Date(now.getTime() + 14 * 24 * 60 * 60 * 1000),
            trialCallsUsed: 0,
            trialCallsLimit: 3,
          },
        });
        user.familyMember.subscription = subscription;
      }
    }

    // Return FLAT response -- NOT nested under body.user
    res.json({
      id: user.id,
      email: user.email,
      name: user.name,
      phone: user.phone,
      isAdmin: checkIsAdmin(user.email),
      familyMember: user.familyMember ? { /* ... */ } : null,
      adminFacility: user.adminFacility ? { /* ... */ } : null,
    });
  } catch (error) {
    next(error);
  }
});
```

---

## Deferred Registration (Frontend)

Backend user creation happens **only after email verification**, not during signup. This prevents orphan users from abandoned signups.

### Registration Flow

```dart
// app-client/lib/shared/providers/auth_provider.dart

Future<void> register({
  required String name,
  required String email,
  required String password,
}) async {
  state = const AsyncValue.loading();

  try {
    // Create Firebase user
    await _firebaseAuth.createUserWithEmailAndPassword(
      email: email,
      password: password,
    );

    try {
      // Set displayName as backup for account creation on first login
      await _firebaseAuth.currentUser?.updateDisplayName(name);

      // Send email verification
      await _firebaseAuth.currentUser?.sendEmailVerification();

      // Sign out -- user must verify email then log in
      await _firebaseAuth.signOut();

      // Return unauthenticated state so UI can navigate to login
      state = const AsyncValue.data(AuthState());
    } catch (e) {
      // Failed to send verification or sign out
      // Delete the Firebase user so user can retry cleanly
      await _firebaseAuth.currentUser?.delete();
      rethrow;
    }
  } catch (e) {
    state = AsyncValue.data(AuthState(error: _parseError(e)));
    rethrow;
  }
}
```

### Complete Registration After Verification

```dart
/// Complete registration after email verification.
/// Reads pending registration data from SecureStorage and calls the
/// appropriate backend endpoint. If no pending data exists, falls back to
/// a basic sync (handles pre-migration users or users who already completed).
Future<User> _completeRegistration() async {
  final pendingJson = await SecureStorage.getPendingRegistration();

  if (pendingJson != null) {
    final pending = jsonDecode(pendingJson) as Map<String, dynamic>;
    final type = pending['type'] as String;

    try {
      User user;
      if (type == 'facility') {
        final apiClient = ref.read(apiClientProvider);
        final response = await apiClient.post(
          ApiEndpoints.registerFacility,
          data: {
            'facilityName': pending['facilityName'],
            'type': pending['facilityType'],
            'city': pending['city'],
            'state': pending['state'],
            if (pending['address'] != null) 'address': pending['address'],
            if (pending['zipCode'] != null) 'zipCode': pending['zipCode'],
            if (pending['facilityPhone'] != null) 'facilityPhone': pending['facilityPhone'],
            if (pending['website'] != null) 'website': pending['website'],
            'adminName': pending['adminName'],
            'adminEmail': pending['adminEmail'],
            if (pending['adminPhone'] != null) 'adminPhone': pending['adminPhone'],
          },
        );
        final data = response.data as Map<String, dynamic>;
        user = User.fromJson(data['user'] as Map<String, dynamic>);
      } else {
        // Family member registration
        user = await _syncWithBackend(
          name: pending['name'] as String?,
          relationship: pending['relationship'] as String?,
          phone: pending['phone'] as String?,
        );
      }

      // Clear pending data after successful sync
      await SecureStorage.removePendingRegistration();
      return user;
    } catch (e) {
      // Don't clear pending data on failure -- allow retry
      rethrow;
    }
  }

  // No pending data -- existing user or pre-migration user
  try {
    return await _syncWithBackend();
  } on DioException catch (e) {
    if (e.response?.statusCode == 401) {
      // No DB user exists and pending data was lost (common on Flutter web
      // where SecureStorage can be unreliable). Fall back to creating a new
      // user using the Firebase displayName.
      final fbUser = _firebaseAuth.currentUser;
      final displayName = fbUser?.displayName;

      // Check if displayName contains JSON facility data
      if (displayName != null && displayName.startsWith('{')) {
        try {
          final data = jsonDecode(displayName) as Map<String, dynamic>;
          if (data['t'] == 'facility') {
            final apiClient = ref.read(apiClientProvider);
            final response = await apiClient.post(
              ApiEndpoints.registerFacility,
              data: {
                'facilityName': data['fn'],
                'type': data['ft'],
                'city': data['c'],
                'state': data['s'],
                'adminName': data['an'],
                'adminEmail': fbUser?.email,
              },
            );
            final responseData = response.data as Map<String, dynamic>;
            return User.fromJson(responseData['user'] as Map<String, dynamic>);
          }
        } catch (_) {
          // JSON parse failed -- fall through to family member fallback
        }
      }

      final fallbackName = displayName ??
          fbUser?.email?.split('@').first ??
          'User';
      return _syncWithBackend(name: fallbackName);
    }
    rethrow;
  }
}
```

**Why deferred?** Without this, every abandoned signup creates a backend user that needs cleanup. With deferred registration, the backend only knows about users who completed verification.

**Gotcha -- Flutter web SecureStorage unreliability:** On Flutter web, `SecureStorage` (which uses `window.localStorage`) can be cleared between sessions. The fallback reads the Firebase `displayName` (set during signup) to reconstruct registration data. For facility admins, the displayName contains compact JSON: `{t: "facility", fn: "Sunrise", ft: "ASSISTED_LIVING", c: "Austin", s: "Texas", an: "John"}`.

---

## Google Sign-In

Google users are auto-verified by Google, so they skip the email verification flow entirely:

```dart
Future<void> signInWithGoogle() async {
  state = const AsyncValue.loading();
  _invalidateAllCachedData();

  try {
    final provider = fb.GoogleAuthProvider();
    final userCredential = await _firebaseAuth.signInWithPopup(provider);
    final fbUser = userCredential.user;

    if (fbUser != null) {
      final user = await _completeRegistration();
      _invalidateAllCachedData();
      await _loadPendingInviteToken();

      state = AsyncValue.data(AuthState(
        user: user,
        isAuthenticated: true,
      ));

      try {
        PushNotificationService.instance.registerToken(ref.read(apiClientProvider));
      } catch (_) {}
    } else {
      state = const AsyncValue.data(AuthState());
    }
  } on fb.FirebaseAuthMultiFactorException catch (e) {
    ref.read(mfaResolverProvider.notifier).state = e.resolver;
    state = const AsyncValue.data(AuthState(error: 'MFA_REQUIRED'));
  } catch (e) {
    state = AsyncValue.data(AuthState(error: _parseError(e)));
    rethrow;
  }
}
```

---

## MFA/TOTP

TOTP 2FA is entirely client-side via Firebase Auth's `TotpMultiFactorGenerator` -- no backend changes needed:

- **Enrollment:** Settings screen generates a TOTP secret, shows QR code
- **Login challenge:** `FirebaseAuthMultiFactorException` caught in `auth_provider` -> redirects to `/mfa-verify`
- **State:** `mfaResolverProvider` holds the `MultiFactorResolver` during the MFA flow

```dart
// Provider to hold MFA resolver during sign-in
final mfaResolverProvider = StateProvider<fb.MultiFactorResolver?>((ref) => null);

/// Complete MFA sign-in with a TOTP code.
Future<void> resolveMultiFactor(String totpCode) async {
  state = const AsyncValue.loading();
  try {
    final resolver = ref.read(mfaResolverProvider);
    if (resolver == null) {
      throw StateError('No MFA resolver available');
    }

    // Find the TOTP hint
    final totpHint = resolver.hints
        .whereType<fb.TotpMultiFactorInfo>()
        .first;

    // Get assertion for sign-in
    final assertion = await fb.TotpMultiFactorGenerator.getAssertionForSignIn(
      totpHint.uid,
      totpCode,
    );

    // Resolve sign-in
    await resolver.resolveSignIn(assertion);

    // Clear resolver
    ref.read(mfaResolverProvider.notifier).state = null;

    // Complete registration / sync with backend
    final user = await _completeRegistration();
    _invalidateAllCachedData();
    state = AsyncValue.data(AuthState(
      user: user,
      isAuthenticated: true,
    ));

    // Register FCM token for push notifications (fire-and-forget)
    PushNotificationService.instance.registerToken(ref.read(apiClientProvider));
  } catch (e) {
    state = AsyncValue.data(AuthState(error: _parseMfaError(e)));
    rethrow;
  }
}
```

---

## Router Guards (Frontend)

Go Router's `redirect` function enforces auth gates. The router reads email verification status **directly from Firebase** to avoid provider chain staleness:

```dart
// app-client/lib/core/router/app_router.dart

/// Notifier that triggers router refresh when auth state changes
class AuthStateNotifier extends ChangeNotifier {
  AuthStateNotifier(this._ref) {
    _ref.listen(authStateProvider, (_, __) {
      notifyListeners();
    });
  }

  final Ref _ref;

  bool get isLoading => _ref.read(authStateProvider).isLoading;
  bool get isAuthenticated =>
      _ref.read(authStateProvider).valueOrNull?.isAuthenticated ?? false;

  /// Read email verification directly from Firebase to avoid provider chain staleness
  bool get isEmailVerified {
    final firebaseAuth = _ref.read(firebaseAuthProvider);
    final user = firebaseAuth.currentUser;
    if (user?.email == 'demo@example.com') return true;
    return user?.emailVerified ?? false;
  }

  /// Compute user role directly from auth state
  UserRole get userRole {
    final user = _ref.read(authStateProvider).valueOrNull?.user;
    if (user == null) return UserRole.none;
    if (user.hasDualRole) return UserRole.dualRole;
    if (user.isFacilityAdmin) return UserRole.facilityAdmin;
    if (user.isFamilyMember) return UserRole.familyMember;
    return UserRole.none;
  }
}

final routerProvider = Provider<GoRouter>((ref) {
  final authNotifier = ref.watch(_authStateNotifierProvider);

  return GoRouter(
    navigatorKey: _rootNavigatorKey,
    initialLocation: '/splash',
    debugLogDiagnostics: kDebugMode,
    refreshListenable: authNotifier,
    redirect: (context, state) {
      final isLoggedIn = authNotifier.isAuthenticated;
      final isLoading = authNotifier.isLoading;
      final isEmailVerified = authNotifier.isEmailVerified;
      final userRole = authNotifier.userRole;
      final path = state.uri.path;

      if (isLoading) return null;

      final publicRoutes = ['/splash', '/login', '/register', '/register-facility',
        '/verify-email', '/forgot-password', '/mfa-verify', '/registration-success', '/activate'];
      final isPublicRoute = publicRoutes.contains(path);

      // Check if MFA is required -- redirect to MFA verification
      final authError = ref.read(authStateProvider).valueOrNull?.error;
      if (authError == 'MFA_REQUIRED' && path != '/mfa-verify') {
        return '/mfa-verify';
      }

      if (!isLoggedIn && !isPublicRoute) {
        return '/login';
      }

      // Authenticated but email not verified -- gate to verification screen
      if (isLoggedIn && !isEmailVerified && path != '/verify-email') {
        return '/verify-email';
      }

      // Check for pending invite token (from facility invite flow)
      if (isLoggedIn && isEmailVerified) {
        final authNotifier = ref.read(authStateProvider.notifier);
        if (authNotifier.pendingInviteToken != null &&
            !path.startsWith('/invite-review')) {
          return '/invite-review?token=${authNotifier.pendingInviteToken}';
        }
      }

      if (isLoggedIn && isEmailVerified && isPublicRoute && path != '/splash') {
        // Route based on user role
        if (userRole == UserRole.facilityAdmin) return '/facility/dashboard';
        if (userRole == UserRole.none) return '/splash';
        return '/home';
      }

      // SECURITY: Redirect non-admins away from admin routes
      if (isLoggedIn && isEmailVerified && path.startsWith('/admin')) {
        final user = ref.read(authStateProvider).valueOrNull?.user;
        if (!(user?.isAdmin ?? false)) return '/home';
      }

      // Facility-only admin on family-only routes -- redirect
      if (isLoggedIn && isEmailVerified && userRole == UserRole.facilityAdmin &&
          !path.startsWith('/facility') && !isPublicRoute && !path.startsWith('/admin') &&
          !path.startsWith('/settings') && !path.startsWith('/notifications')) {
        return '/facility/dashboard';
      }

      return null;
    },
    // ... routes
  );
});
```

---

## How Auto-Linking Works Step by Step

1. User has an existing account in PostgreSQL (email: `alice@example.com`, no `firebaseUid`)
2. User signs up with Firebase Auth (creates Firebase user with `uid: abc123`)
3. User verifies email and logs in
4. Frontend sends Firebase ID token to backend
5. `verifyFirebaseToken` decodes the token, gets `uid: abc123` and `email: alice@example.com`
6. Looks up user by `firebaseUid: abc123` -- not found (no link yet)
7. Checks `decoded.email_verified === true` -- yes
8. Looks up user by `email: alice@example.com` -- found!
9. Updates user record: `SET firebaseUid = 'abc123'`
10. Future requests use `firebaseUid` lookup directly (fast path)

**Security gate:** Step 7 is critical. Without the `email_verified` check, an attacker could create a Firebase account with someone else's email (unverified) and hijack their account.

---

## Password Re-authentication

Before destructive actions (account deletion), require re-authentication:

- **Password users:** Re-enter password via `reauthenticateWithCredential`
- **Google users:** Re-auth via `reauthenticateWithPopup`

---

## Gotchas

1. **Firebase token caching:** After email verification, the persisted Firebase ID token still has `email_verified: false`. You must call `fbUser.getIdToken(true)` to force a refresh.

2. **`/auth/sync` response shape:** Returns FLAT response (`body.email`, `body.name`), NOT nested (`body.user.email`). This was a source of bugs during initial implementation.

3. **Firebase rate limits `sendEmailVerification`:** Always implement a cooldown timer (60 seconds) on the verification screen to prevent "too many requests" errors.

4. **Auto-linking race condition:** If two Firebase accounts use the same email (e.g., email/password + Google), the second one to verify will auto-link. The `email_verified` check prevents a malicious unverified account from hijacking.

5. **`authenticate` middleware is idempotent:** Safe to apply at both router and route level. The `if (req.user) return next()` guard at the top prevents double-verification overhead.

---

## Checklist

- [ ] Create Firebase project, enable Email/Password + Google Sign-In providers
- [ ] Install `firebase-admin` on backend, initialize with project ID
- [ ] Implement `authenticate` middleware with Firebase token verification
- [ ] Add `firebaseUid` column to User model (unique, nullable for migration)
- [ ] Add `tokensRevokedAt` column for bulk token revocation
- [ ] Create `RevokedToken` model with `tokenHash` (unique) and `expiresAt`
- [ ] Implement deferred registration: store pending data client-side, create backend user after verification
- [ ] Add email verification screen with cooldown timer
- [ ] Implement Go Router redirect guards for auth, email verification, MFA, role-based routing
- [ ] Add Google Sign-In (web: `signInWithPopup`, mobile: `signInWithProvider`)
- [ ] Add MFA enrollment screen with QR code generation
- [ ] Handle `FirebaseAuthMultiFactorException` in login flow
- [ ] Implement session timeout service (in-memory, configurable duration)
- [ ] Add password re-authentication before account deletion
- [ ] Invalidate all cached data providers on logout (prevent cross-user data leakage)

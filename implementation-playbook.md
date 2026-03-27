# Full-Stack Implementation Playbook

A reusable guide for building production-grade SaaS applications with Firebase Auth, Stripe, GCP Cloud Run, and Flutter.

Each module below is a standalone deep-dive with complete, working code examples. An AI agent reading these docs should be able to reproduce the same patterns in a new project.

---

## Modules

### [01 - Authentication](playbook/01-authentication.md)
Firebase Auth with deferred registration, dual-mode token verification (Firebase + legacy JWT), auto-linking by email, Google Sign-In, TOTP MFA, Go Router guards, session timeout, and token revocation. Covers the full lifecycle from signup through email verification to backend user creation.

### [02 - Email & Notifications](playbook/02-email-notifications.md)
Gmail API via domain-wide delegation with the IAM Credentials API signJwt workaround for Cloud Run. FCM push notifications, in-app notification persistence, HMAC-based one-click unsubscribe, dual notification preference flags, and scheduled email jobs via Cloud Scheduler.

### [03 - Security](playbook/03-security.md)
Compliance-ready security patterns: structured logging with PII auto-redaction (Pino + GCP Cloud Logging), generic error responses with custom error classes, client-side error mapping, Zod input validation, three-layer rate limiting with account lockout, audit logging, breach detection, webhook signature validation (Stripe + OIDC), and IDOR prevention.

### [04 - Payments (Stripe)](playbook/04-payments-stripe.md)
Stripe integration with graceful degradation, checkout session creation with automatic tax, plan changes via Customer Portal, webhook handling with idempotency, product/price ID management in cloudbuild.yaml, B2B seat-based facility billing, subscription lifecycle state machine, and DB/Stripe sync recovery.

### [05 - Cloud Infrastructure](playbook/05-cloud-infrastructure.md)
GCP Cloud Run deployment via Cloud Build, multi-stage Dockerfile, Cloud Scheduler job creation with OIDC auth, staging vs production environments, Secret Manager, CORS configuration, health check implementation, Express app setup with Helmet, and database migration strategy.

### [06 - Environment & Config](playbook/06-environment-config.md)
Zod-based environment validation with type transformations, Secret Manager placeholder handling, production-only HTTPS and SSL enforcement, graceful degradation pattern for optional services, and frontend base URL configuration via compile-time flags.

### [07 - Frontend Patterns (Flutter)](playbook/07-frontend-patterns.md)
Riverpod AsyncNotifier state management, Dio API client with Firebase auth interceptor and token refresh retry, Go Router with ShellRoute and auth guards, design system (colors, spacing, typography, shadows), secure storage, centralized API endpoints, error utilities, and feature module structure.

### [08 - Database (Prisma)](playbook/08-database-prisma.md)
Prisma schema design with soft deletes, CUID IDs, pgvector support, one-to-one extension pattern, RBAC setup with restricted app role, migration workflow for dev/staging/production, connection string configuration, and seeding.

### [09 - Testing](playbook/09-testing.md)
Jest test setup with deep Prisma mocks, Firebase Admin mocks, global mock registration, mock factory pattern, Supertest integration tests, auth chain mocking, Flutter test setup with Riverpod container overrides, mocktail, firebase_auth_mocks, test data fixtures, and security regression tests.

---

## Quick-Reference Flows

### Auth Flow (Email/Password)
```
1. User enters email/password on Register screen
2. Firebase createUserWithEmailAndPassword
3. Set displayName as fallback, store pending data in SecureStorage
4. Send email verification, sign out
5. User verifies email, logs in
6. Firebase signInWithEmailAndPassword
7. Force token refresh: fbUser.getIdToken(true)
8. _completeRegistration() reads SecureStorage -> POST /auth/sync/new
9. Backend creates User + UserProfile + Trial Subscription in transaction
10. Router redirects to /home
```

### Auth Flow (Google Sign-In)
```
1. User clicks "Sign in with Google"
2. Firebase signInWithPopup(GoogleAuthProvider)
3. Google users are auto-verified -- skip email verification
4. _completeRegistration() -> POST /auth/sync/new or /auth/sync
5. Router redirects to /home
```

### Stripe Checkout Flow
```
1. User selects plan on Pricing screen
2. POST /subscriptions/checkout -> Stripe Checkout Session
3. launchUrl(session.url) opens Stripe-hosted checkout
4. User completes payment
5. Stripe webhook: checkout.session.completed
6. Backend upserts Subscription with tier, status, period dates
7. User returns to app, WidgetsBindingObserver.didChangeAppLifecycleState refreshes state
```

### Stripe Plan Change Flow
```
1. User selects new plan
2. POST /subscriptions/change-plan -> Customer Portal URL with flow_data
3. launchUrl(portal.url) shows proration preview
4. User confirms change
5. Stripe webhook: customer.subscription.updated
6. Backend maps product ID -> tier, updates DB
7. If downgrade: truncate schedules, notify user
```

### Notification Flow
```
1. Business event triggers notification method (e.g., notifyAlert)
2. Look up user + userProfile (notification preferences)
3. Check receiveEmails flag -> send email via Gmail API (if enabled)
4. Check receiveNotifications flag + fcmToken -> send push via FCM (if enabled)
5. Always persist in-app notification (visible in notification center)
6. All steps are fire-and-forget -- errors logged, never thrown
```

### Deployment Flow
```
1. Apply database migrations: npx prisma migrate deploy (via Cloud SQL Auth Proxy)
2. Submit Cloud Build: gcloud builds submit --config=engine/cloudbuild.yaml
3. Cloud Build: docker build -> push to GCR -> deploy to Cloud Run
4. Health check: curl ${APP_URL}/health -> verify 200
5. If health check fails: gcloud run services update-traffic --to-latest
```

---

## File Structure Reference

```
project/
├── engine/                          # Backend
│   ├── src/
│   │   ├── config/
│   │   │   ├── env.ts               # Zod env validation
│   │   │   ├── firebase.ts          # Firebase Admin init
│   │   │   ├── logger.ts            # Pino + PII redaction
│   │   │   └── sentry.ts            # Error tracking
│   │   ├── middleware/
│   │   │   ├── auth.ts              # Firebase + JWT verification
│   │   │   ├── adminAuth.ts         # Admin role check
│   │   │   ├── oidcAuth.ts          # Cloud Scheduler auth
│   │   │   ├── rateLimiter.ts       # IP + user rate limits
│   │   │   ├── errorHandler.ts      # Custom error classes
│   │   │   └── auditMiddleware.ts   # Sensitive data access logging
│   │   ├── routes/
│   │   │   ├── index.ts             # Route registry
│   │   │   ├── auth.routes.ts       # Auth endpoints
│   │   │   ├── webhook.routes.ts    # Stripe webhooks
│   │   │   └── internal.routes.ts   # Cloud Scheduler endpoints
│   │   ├── services/
│   │   │   ├── stripe.service.ts    # Stripe with graceful degradation
│   │   │   ├── notification.service.ts # Email + Push + In-app
│   │   │   ├── audit.service.ts     # Audit log persistence
│   │   │   ├── token.service.ts     # Token revocation
│   │   │   ├── session.service.ts   # Inactivity timeout
│   │   │   └── breach-detection.service.ts
│   │   ├── utils/
│   │   │   └── retry.ts             # withRetry for external APIs
│   │   ├── app.ts                   # Express app factory
│   │   └── index.ts                 # Server entry + Prisma client
│   ├── prisma/
│   │   ├── schema.prisma            # Database schema
│   │   ├── rbac-setup.sql           # Restricted app role
│   │   ├── seed.ts                  # Demo data
│   │   └── migrations/
│   ├── __tests__/
│   │   ├── setup.ts                 # Global mocks
│   │   ├── mocks/                   # Mock factories
│   │   ├── fixtures/                # Test data factories
│   │   ├── integration/routes/      # Supertest route tests
│   │   ├── unit/services/           # Service unit tests
│   │   └── security/               # Security regression tests
│   ├── cloudbuild.yaml              # Production deploy
│   ├── cloudbuild-staging.yaml      # Staging deploy
│   ├── Dockerfile
│   └── .env.example
├── app-client/                      # Frontend
│   └── lib/
│       ├── core/
│       │   ├── api/api_client.dart   # Dio + interceptors
│       │   ├── router/app_router.dart # Go Router + guards
│       │   ├── theme/app_theme.dart
│       │   ├── constants/api_endpoints.dart
│       │   ├── utils/error_utils.dart
│       │   └── security/secure_storage.dart
│       ├── shared/
│       │   ├── models/user.dart
│       │   ├── providers/           # 12 Riverpod providers
│       │   └── widgets/
│       └── features/                # 13 feature modules
└── docs/
    ├── implementation-playbook.md   # This file (index)
    └── playbook/
        ├── 01-authentication.md
        ├── 02-email-notifications.md
        ├── 03-security.md
        ├── 04-payments-stripe.md
        ├── 05-cloud-infrastructure.md
        ├── 06-environment-config.md
        ├── 07-frontend-patterns.md
        ├── 08-database-prisma.md
        └── 09-testing.md
```

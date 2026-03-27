# 06 - Environment & Config

## Why

Zod-based env validation catches misconfiguration at startup rather than at runtime when a request hits a missing API key. Optional services with graceful degradation let you develop locally without every external service configured.

---

## Zod-Based Env Validation (Complete)

```typescript
// engine/src/config/env.ts

import { z } from 'zod';
import dotenv from 'dotenv';

dotenv.config();

const envSchema = z.object({
  // Server
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.string().default('3000').transform(Number),

  // Database
  // SECURITY: Use sslmode=require in production DATABASE_URL, e.g.:
  // postgresql://user:pass@host:5432/myapp?schema=public&sslmode=require
  DATABASE_URL: z.string().min(1, 'DATABASE_URL is required'),

  // JWT (optional -- only needed during dual-mode transition with legacy logins)
  JWT_SECRET: z.string().min(32).optional(),
  JWT_EXPIRES_IN: z.string().default('7d'),

  // Voice/telephony provider (optional until voice pipeline)
  TELNYX_API_KEY: z.string().optional(),
  TELNYX_CONNECTION_ID: z.string().optional(),
  TELNYX_PHONE_NUMBER: z.string().optional(),
  TELNYX_TEST_PHONE_NUMBER: z.string().optional(),
  TELNYX_WEBHOOK_PUBLIC_KEY: z.string().optional(),

  // Concurrency: max simultaneous outbound tasks
  MAX_CONCURRENT_TASKS: z.string().default('10').transform(Number),

  // Speech-to-text provider (optional until voice pipeline)
  DEEPGRAM_API_KEY: z.string().optional(),

  // AI provider (optional until voice pipeline)
  ANTHROPIC_API_KEY: z.string().optional(),

  // OpenAI (for embeddings in vector search)
  OPENAI_API_KEY: z.string().optional(),

  // Cohere (alternative for multilingual embeddings)
  COHERE_API_KEY: z.string().optional(),
  EMBEDDING_PROVIDER: z.enum(['openai', 'cohere']).optional().default('openai'),

  // Primary TTS provider (optional)
  ELEVENLABS_API_KEY: z.string().optional(),

  // Fallback TTS provider (optional)
  CARTESIA_API_KEY: z.string().optional(),

  // Firebase
  FIREBASE_PROJECT_ID: z.string().min(1, 'FIREBASE_PROJECT_ID is required'),
  FIREBASE_SERVICE_ACCOUNT_PATH: z.string().optional(),

  // Notifications -- Gmail API via service account domain-wide delegation
  GMAIL_SENDER_EMAIL: z.string().email().optional(),
  GMAIL_IMPERSONATE_EMAIL: z.string().email().optional(),
  ADMIN_EMAIL: z.string().email().optional(),

  // Stripe (optional until subscriptions)
  // Transform placeholder values from Secret Manager to undefined
  STRIPE_SECRET_KEY: z.string().optional().transform(v => v === 'placeholder' ? undefined : v),
  STRIPE_WEBHOOK_SECRET: z.string().optional().transform(v => v === 'placeholder' ? undefined : v),
  // Stable Stripe Product IDs
  STRIPE_PRODUCT_STARTER: z.string().optional(),
  STRIPE_PRODUCT_ESSENTIALS: z.string().optional(),
  STRIPE_PRODUCT_PLUS: z.string().optional(),
  STRIPE_PRODUCT_FAMILY: z.string().optional(),
  // Stable Stripe Price IDs
  STRIPE_PRICE_STARTER: z.string().optional(),
  STRIPE_PRICE_ESSENTIALS: z.string().optional(),
  STRIPE_PRICE_PLUS: z.string().optional(),
  STRIPE_PRICE_FAMILY: z.string().optional(),
  // Facility billing (B2B seat-based)
  STRIPE_FACILITY_PRODUCT: z.string().optional(),
  STRIPE_FACILITY_PRICE: z.string().optional(),
  // Portal configurations
  STRIPE_PORTAL_CONFIG_FAMILY: z.string().optional(),
  STRIPE_PORTAL_CONFIG_FACILITY: z.string().optional(),

  // Sentry (optional -- error tracking disabled if not set)
  SENTRY_DSN: z.string().optional().transform(v => v && v.startsWith('http') ? v : undefined),

  // Google Cloud Storage
  GCP_PROJECT_ID: z.string().optional(),
  GCS_BUCKET_NAME: z.string().optional(),
  GCP_SERVICE_ACCOUNT: z.string().optional(),

  // Admin
  ADMIN_EMAILS: z.string().optional(),

  // SECURITY: Session timeout (minutes of inactivity before auto-logout)
  SESSION_TIMEOUT_MINUTES: z.string().default('30').transform(Number),

  // Unsubscribe
  UNSUBSCRIBE_SECRET: z.string().optional(),

  // App
  APP_URL: z.string().default('http://localhost:3000'),
  CORS_ORIGINS: z.string().default('http://localhost:3000,http://localhost:5173'),

  // Version tracking (injected by Cloud Build)
  COMMIT_SHA: z.string().optional().default('dev'),
});

const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('Environment validation failed:');
  console.error(parsed.error.format());
  process.exit(1);
}

// SECURITY: Validate HTTPS in production
if (parsed.data.NODE_ENV === 'production') {
  if (!parsed.data.APP_URL.startsWith('https://')) {
    console.error('SECURITY: APP_URL must use HTTPS in production');
    process.exit(1);
  }
  // SECURITY: Enforce encrypted database connections in production
  // Cloud SQL Auth Proxy (Unix socket via /cloudsql/) encrypts automatically.
  // Direct TCP connections must use sslmode=require.
  const dbUrl = parsed.data.DATABASE_URL;
  if (!dbUrl.includes('/cloudsql/') && !dbUrl.includes('sslmode=')) {
    console.error('SECURITY: DATABASE_URL must include sslmode=require for direct TCP connections in production');
    process.exit(1);
  }
}

export const env = parsed.data;

export type Env = z.infer<typeof envSchema>;
```

---

## Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NODE_ENV` | No | `development` | `development`, `production`, or `test` |
| `PORT` | No | `3000` | Server port (Cloud Run uses 8080) |
| `DATABASE_URL` | **Yes** | -- | PostgreSQL connection string |
| `FIREBASE_PROJECT_ID` | **Yes** | -- | Firebase project ID |
| `JWT_SECRET` | No | -- | Legacy JWT signing key (min 32 chars) |
| `TELNYX_API_KEY` | No | -- | Voice/telephony provider API key |
| `DEEPGRAM_API_KEY` | No | -- | Speech-to-text provider API key |
| `ANTHROPIC_API_KEY` | No | -- | AI provider API key |
| `OPENAI_API_KEY` | No | -- | Embeddings provider API key |
| `ELEVENLABS_API_KEY` | No | -- | Primary TTS provider API key |
| `CARTESIA_API_KEY` | No | -- | Fallback TTS provider API key |
| `STRIPE_SECRET_KEY` | No | -- | Stripe API key (subscriptions disabled if unset) |
| `STRIPE_WEBHOOK_SECRET` | No | -- | Stripe webhook signature secret |
| `GMAIL_SENDER_EMAIL` | No | -- | Email notifications sender (disabled if unset) |
| `GMAIL_IMPERSONATE_EMAIL` | No | -- | Primary Workspace user to impersonate (required if sender is alias) |
| `SENTRY_DSN` | No | -- | Error tracking DSN |
| `GCS_BUCKET_NAME` | No | -- | Cloud storage bucket |
| `ADMIN_EMAILS` | No | -- | Comma-separated admin email addresses |
| `APP_URL` | No | `http://localhost:3000` | Public URL (must be HTTPS in production) |
| `CORS_ORIGINS` | No | `http://localhost:3000,http://localhost:5173` | Comma-separated allowed origins |
| `MAX_CONCURRENT_TASKS` | No | `10` | Max simultaneous outbound tasks |
| `SESSION_TIMEOUT_MINUTES` | No | `30` | Security inactivity timeout |
| `COMMIT_SHA` | No | `dev` | Git commit hash (injected by Cloud Build) |

---

## Key Patterns

### String-to-Number Transformation
```typescript
PORT: z.string().default('3000').transform(Number),
```
Environment variables are always strings. Use `.transform(Number)` to parse them.

### Secret Manager Placeholder Handling
```typescript
STRIPE_SECRET_KEY: z.string().optional().transform(v => v === 'placeholder' ? undefined : v),
```
When you create a Secret Manager secret without setting the actual value, it contains `"placeholder"`. Transform these to `undefined` to trigger graceful degradation.

### Sentry DSN Validation
```typescript
SENTRY_DSN: z.string().optional().transform(v => v && v.startsWith('http') ? v : undefined),
```
Only accept valid-looking DSN URLs.

### Production-Only Checks
```typescript
if (parsed.data.NODE_ENV === 'production') {
  if (!parsed.data.APP_URL.startsWith('https://')) {
    process.exit(1);
  }
  if (!dbUrl.includes('/cloudsql/') && !dbUrl.includes('sslmode=')) {
    process.exit(1);
  }
}
```

---

## Service Graceful Degradation Pattern

```typescript
class MyExternalService {
  private client: ExternalClient | null = null;

  constructor() {
    if (env.MY_API_KEY) {
      this.client = new ExternalClient(env.MY_API_KEY);
    }
  }

  isConfigured(): boolean {
    return !!this.client;
  }

  private getClient(): ExternalClient {
    if (!this.client) throw new Error('Service not configured');
    return this.client;
  }
}

// Route guard:
if (!myService.isConfigured()) {
  return res.status(503).json({ error: 'Service not available' });
}
```

---

## Frontend Base URL Configuration

```dart
class ApiEndpoints {
  static const String baseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:3000/api',
  );

  static const String authSync = '/auth/sync';
  static const String authSyncNew = '/auth/sync/new';
  // ...
}
```

```bash
# Local development (default)
flutter run -d chrome

# Staging
flutter run -d chrome --dart-define=API_BASE_URL=https://myapp-engine-staging-XXXXXXXXX.a.run.app/api

# Production build
flutter build web --release --dart-define=API_BASE_URL=https://myapp-engine-XXXXXXXXX.us-central1.run.app/api
```

---

## Gotchas

1. **Secret Manager placeholder values:** Transform `"placeholder"` to `undefined` in Zod.

2. **Cloud SQL Auth Proxy uses Unix sockets** (`/cloudsql/project:region:instance`), which don't need `sslmode`. Only direct TCP connections need `sslmode=require`.

3. **`CORS_ORIGINS` must include all frontend domains** -- both the Firebase Hosting URL and any custom domain.

4. **`PORT` must be `8080` on Cloud Run** (not the default `3000`). Set it in `cloudbuild.yaml` via `--port 8080`.

---

## Checklist

- [ ] Create Zod schema for all env vars with proper types and defaults
- [ ] Add `.transform()` for string-to-number conversions
- [ ] Handle Secret Manager placeholder values
- [ ] Add production-only HTTPS and SSL checks
- [ ] Implement `isConfigured()` pattern for all optional services
- [ ] Set up `.env.example` documenting all variables
- [ ] Configure frontend base URL via compile-time flag
- [ ] Create separate staging and production env configurations

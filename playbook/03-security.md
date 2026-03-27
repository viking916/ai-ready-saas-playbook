# 03 - Security

## Why

Security best practices require defense-in-depth: structured logging with PII redaction, generic error responses, input validation, rate limiting, audit logging, and ownership verification. These practices prevent common vulnerabilities (IDOR, injection, data leaks in logs).

---

## Structured Logging with PII Auto-Redaction

Never `console.log`. Always use `createLogger()` which auto-redacts PII:

```typescript
// engine/src/config/logger.ts

import pino from 'pino';
import { env } from './env.js';

// Phone number patterns: +1234567890, (123) 456-7890, 123-456-7890
const PHONE_REGEX = /(\+?\d[\d\s\-().]{7,}\d)/g;
// Email pattern
const EMAIL_REGEX = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;

/**
 * Redact PII from a string value.
 * Masks phone numbers and email addresses to prevent PII leaks in logs.
 */
function redactString(value: string): string {
  return value
    .replace(PHONE_REGEX, '[REDACTED_PHONE]')
    .replace(EMAIL_REGEX, '[REDACTED_EMAIL]');
}

/**
 * Deep-redact PII from an object for safe logging.
 * Recursively processes strings in objects/arrays.
 */
function redactValue(value: unknown): unknown {
  if (typeof value === 'string') {
    return redactString(value);
  }
  if (Array.isArray(value)) {
    return value.map(redactValue);
  }
  if (value !== null && typeof value === 'object') {
    const redacted: Record<string, unknown> = {};
    for (const [k, v] of Object.entries(value as Record<string, unknown>)) {
      // Fully redact known PII field names
      const lower = k.toLowerCase();
      if (
        lower === 'phonenumber' ||
        lower === 'phone' ||
        lower === 'email' ||
        lower === 'preferredname' ||
        lower === 'fullname' ||
        lower === 'name' && typeof v === 'string'
      ) {
        redacted[k] = '[REDACTED]';
      } else {
        redacted[k] = redactValue(v);
      }
    }
    return redacted;
  }
  return value;
}

const isProduction = env.NODE_ENV === 'production';
const isTest = env.NODE_ENV === 'test';

const logger = pino({
  level: isTest ? 'silent' : isProduction ? 'info' : 'debug',
  ...(isProduction
    ? {
        // GCP Cloud Logging compatible JSON format
        formatters: {
          level(label: string) {
            // Map pino levels to GCP Cloud Logging severity
            const severityMap: Record<string, string> = {
              trace: 'DEBUG',
              debug: 'DEBUG',
              info: 'INFO',
              warn: 'WARNING',
              error: 'ERROR',
              fatal: 'CRITICAL',
            };
            return {
              severity: severityMap[label] || 'DEFAULT',
              level: label,
            };
          },
        },
      }
    : {
        transport: {
          target: 'pino-pretty',
          options: {
            colorize: true,
            translateTime: 'HH:MM:ss',
            ignore: 'pid,hostname',
          },
        },
      }),
});

/**
 * Create a child logger with a service context.
 * All log entries will include the service name for filtering.
 */
export function createLogger(service: string) {
  const child = logger.child({ service });

  return {
    debug(msg: string, data?: Record<string, unknown>) {
      child.debug(data ? redactValue(data) as Record<string, unknown> : {}, msg);
    },
    info(msg: string, data?: Record<string, unknown>) {
      child.info(data ? redactValue(data) as Record<string, unknown> : {}, msg);
    },
    warn(msg: string, data?: Record<string, unknown>) {
      child.warn(data ? redactValue(data) as Record<string, unknown> : {}, msg);
    },
    error(msg: string, data?: Record<string, unknown> | Error) {
      if (data instanceof Error) {
        child.error({ err: data }, msg);
      } else {
        child.error(data ? redactValue(data) as Record<string, unknown> : {}, msg);
      }
    },
    fatal(msg: string, data?: Record<string, unknown> | Error) {
      if (data instanceof Error) {
        child.fatal({ err: data }, msg);
      } else {
        child.fatal(data ? redactValue(data) as Record<string, unknown> : {}, msg);
      }
    },
  };
}

export default logger;
```

**Key design:** Two-level redaction:
1. **Field-name-based:** `phone`, `email`, `name` fields are fully replaced with `[REDACTED]`
2. **Regex-based:** Phone numbers and emails embedded in any string value are replaced

**GCP integration:** In production, level names map to GCP Cloud Logging severity levels for proper filtering.

---

## Generic Error Responses

Never expose `error.message` to clients. Use custom error classes:

```typescript
// engine/src/middleware/errorHandler.ts

import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';
import { env } from '../config/env.js';
import { createLogger } from '../config/logger.js';
import { captureException } from '../config/sentry.js';

const log = createLogger('error-handler');

export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;

  constructor(message: string, statusCode: number = 500, isOperational: boolean = true) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = isOperational;

    Error.captureStackTrace(this, this.constructor);
  }
}

export class BadRequestError extends AppError {
  constructor(message: string = 'Bad Request') {
    super(message, 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Forbidden') {
    super(message, 403);
  }
}

export class NotFoundError extends AppError {
  constructor(message: string = 'Not Found') {
    super(message, 404);
  }
}

export class ConflictError extends AppError {
  constructor(message: string = 'Conflict') {
    super(message, 409);
  }
}

export function errorHandler(
  err: Error,
  _req: Request,
  res: Response,
  _next: NextFunction
): void {
  // Log error
  log.error(err.message, err);

  // Handle Zod validation errors
  if (err instanceof ZodError) {
    res.status(400).json({
      error: 'Validation Error',
      message: 'Invalid request data',
      details: err.errors.map(e => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    });
    return;
  }

  // Handle known operational errors
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: err.name,
      message: err.message,
    });
    return;
  }

  // Handle Prisma errors
  if (err.name === 'PrismaClientKnownRequestError') {
    const prismaError = err as any;
    if (prismaError.code === 'P2002') {
      res.status(409).json({
        error: 'Conflict',
        message: 'A record with this value already exists',
      });
      return;
    }
    if (prismaError.code === 'P2025') {
      res.status(404).json({
        error: 'Not Found',
        message: 'Record not found',
      });
      return;
    }
  }

  // Unknown errors -- report to Sentry
  captureException(err, {
    url: _req.originalUrl,
    method: _req.method,
  });

  res.status(500).json({
    error: 'Internal Server Error',
    message: env.NODE_ENV === 'development' ? err.message : 'An unexpected error occurred',
  });
}
```

---

## Client-Side Error Mapping

On the Flutter side, map technical errors to user-friendly messages. Only pass through backend messages for user-input errors (400, 409, 422):

```dart
// app-client/lib/core/utils/error_utils.dart

import 'package:dio/dio.dart';
import 'package:firebase_auth/firebase_auth.dart';

class ErrorUtils {
  static String getErrorMessage(dynamic error) {
    if (error is FirebaseAuthException) {
      return _getFirebaseAuthErrorMessage(error);
    }
    if (error is DioException) {
      return _getDioErrorMessage(error);
    }
    if (error is Exception) {
      return 'Something went wrong. Please try again.';
    }
    return 'An unexpected error occurred';
  }

  static String _getFirebaseAuthErrorMessage(FirebaseAuthException error) {
    switch (error.code) {
      case 'user-not-found':
      case 'wrong-password':
      case 'invalid-credential':
        return 'Invalid email or password. Please check your credentials and try again.';
      case 'email-already-in-use':
        return 'An account with this email already exists. Please sign in instead.';
      case 'weak-password':
        return 'Password is too weak. Please use at least 6 characters.';
      case 'invalid-email':
        return 'Please enter a valid email address.';
      case 'too-many-requests':
        return 'Too many attempts. Please wait a few minutes before trying again.';
      case 'network-request-failed':
        return 'Unable to connect. Please check your internet connection and try again.';
      case 'user-disabled':
        return 'This account has been disabled. Please contact support for help.';
      case 'requires-recent-login':
        return 'For security, please sign out and sign back in to make this change.';
      default:
        return 'Something went wrong. Please try again.';
    }
  }

  static String _getDioErrorMessage(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return 'Connection timed out. Please check your internet and try again.';
      case DioExceptionType.connectionError:
        return 'Unable to connect to the server. Please check your internet connection.';
      case DioExceptionType.cancel:
        return 'Request was cancelled.';
      case DioExceptionType.badResponse:
        return _getResponseErrorMessage(error.response);
      default:
        return 'Something went wrong. Please try again.';
    }
  }

  /// Only passes through backend messages for input validation errors
  /// (400, 409, 422) where the message is intended for end users.
  static String _getResponseErrorMessage(Response? response) {
    final statusCode = response?.statusCode;

    // For input-related errors, the backend message is user-facing
    if (statusCode == 400 || statusCode == 409 || statusCode == 422) {
      final backendMessage = _extractBackendMessage(response);
      if (backendMessage != null) return backendMessage;
    }

    return _getStatusCodeMessage(statusCode);
  }

  static String? _extractBackendMessage(Response? response) {
    if (response?.data == null) return null;
    final data = response!.data;
    if (data is Map<String, dynamic>) {
      final message = data['message'] ?? data['detail'];
      if (message is String && message.isNotEmpty) return message;
    }
    return null;
  }

  static String _getStatusCodeMessage(int? statusCode) {
    switch (statusCode) {
      case 400: return 'Please check your input and try again.';
      case 401: return 'Your session has expired. Please sign in again.';
      case 403: return 'You do not have permission to perform this action.';
      case 404: return 'The requested resource was not found.';
      case 409: return 'This action conflicts with existing data.';
      case 422: return 'Please check your input and try again.';
      case 429: return 'Too many requests. Please wait a moment and try again.';
      case 500: case 502: case 503:
        return 'Our servers are temporarily unavailable. Please try again later.';
      default: return 'Something went wrong. Please try again.';
    }
  }
}
```

**Why only 400/409/422?** These status codes represent user-input errors where the backend message (e.g., "Email already exists") is intended for the user. A 401 message like "Token has been revoked" is a technical detail that should never be shown.

---

## Rate Limiting

Three layers of rate limiting, plus account lockout:

```typescript
// engine/src/middleware/rateLimiter.ts

import rateLimit, { ipKeyGenerator } from 'express-rate-limit';
import { Request } from 'express';
import { env } from '../config/env.js';

// 1. Auth endpoints -- strict, IP-based (prevents brute force)
export const authRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: env.NODE_ENV === 'production' ? 5 : 100,
  message: {
    error: 'Too many authentication attempts',
    message: 'Please try again after 15 minutes',
  },
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: env.NODE_ENV === 'development',
});

// 2. Password reset (extra strict)
export const passwordResetRateLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: env.NODE_ENV === 'production' ? 3 : 100,
  message: {
    error: 'Too many password reset attempts',
    message: 'Please try again after 1 hour',
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// 3. Global API -- IP-based (first line of defense)
export const apiRateLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: env.NODE_ENV === 'production' ? 100 : 1000,
  message: {
    error: 'Too many requests',
    message: 'Please slow down and try again shortly',
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// 4. Authenticated routes -- user-based (prevents single user from overwhelming system)
export const authenticatedRateLimiter = rateLimit({
  windowMs: 1 * 60 * 1000, // 1 minute
  max: env.NODE_ENV === 'production' ? 60 : 1000,
  keyGenerator: (req: Request) => {
    return req.user?.userId || ipKeyGenerator(req.ip ?? 'unknown');
  },
  message: {
    error: 'Too many requests',
    message: 'Please slow down and try again shortly',
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// Per-account lockout after repeated failed login attempts
const LOCKOUT_MAX_ATTEMPTS = 10;
const LOCKOUT_WINDOW_MS = 60 * 60 * 1000; // 1 hour
const LOCKOUT_DURATION_MS = 30 * 60 * 1000; // 30 minutes

interface LoginAttemptRecord {
  attempts: number;
  firstAttemptAt: number;
  lockedUntil: number | null;
}

const loginAttempts = new Map<string, LoginAttemptRecord>();

// Clean up stale entries every 15 minutes
setInterval(() => {
  const now = Date.now();
  for (const [email, record] of loginAttempts.entries()) {
    const windowExpired = now - record.firstAttemptAt > LOCKOUT_WINDOW_MS;
    const lockExpired = record.lockedUntil && now > record.lockedUntil;
    if (windowExpired && (!record.lockedUntil || lockExpired)) {
      loginAttempts.delete(email);
    }
  }
}, 15 * 60 * 1000).unref();

export function isAccountLocked(email: string): boolean {
  if (env.NODE_ENV !== 'production') return false;
  const key = email.toLowerCase();
  const record = loginAttempts.get(key);
  if (!record?.lockedUntil) return false;
  if (Date.now() > record.lockedUntil) {
    loginAttempts.delete(key);
    return false;
  }
  return true;
}

export function recordFailedLogin(email: string): void {
  if (env.NODE_ENV !== 'production') return;
  const key = email.toLowerCase();
  const now = Date.now();
  const record = loginAttempts.get(key);

  if (!record || now - record.firstAttemptAt > LOCKOUT_WINDOW_MS) {
    loginAttempts.set(key, { attempts: 1, firstAttemptAt: now, lockedUntil: null });
    return;
  }

  record.attempts++;
  if (record.attempts >= LOCKOUT_MAX_ATTEMPTS) {
    record.lockedUntil = now + LOCKOUT_DURATION_MS;
  }
}

export function clearFailedLogins(email: string): void {
  loginAttempts.delete(email.toLowerCase());
}
```

---

## Audit Logging

```typescript
// engine/src/services/audit.service.ts

import { Request } from 'express';
import { withRetry } from '../utils/retry.js';
import { createLogger } from '../config/logger.js';

const log = createLogger('audit');

export type AuditAction = 'READ' | 'CREATE' | 'UPDATE' | 'DELETE' | 'LOGIN' |
  'LOGIN_FAILED' | 'LOGOUT' | 'EXPORT' | 'CONSENT_CHANGE' | 'TOKEN_REVOKE' |
  'WEBHOOK_PROCESSED' | 'WINBACK_EMAIL_SENT' | 'INACTIVE_TRIAL_NUDGE_SENT';

export type AuditOutcome = 'SUCCESS' | 'DENIED' | 'ERROR';

interface AuditEntry {
  userId?: string | null;
  action: AuditAction;
  resource: string;
  resourceId?: string | null;
  details?: Record<string, unknown> | null;
  ipAddress?: string | null;
  userAgent?: string | null;
  outcome?: AuditOutcome;
}

class AuditService {
  /**
   * Log an audit event. Fire-and-forget by default to avoid blocking requests.
   */
  async log(entry: AuditEntry): Promise<void> {
    const data = {
      userId: entry.userId ?? null,
      action: entry.action,
      resource: entry.resource,
      resourceId: entry.resourceId ?? null,
      details: entry.details ?? null,
      ipAddress: entry.ipAddress ?? null,
      userAgent: entry.userAgent ?? null,
      outcome: entry.outcome ?? 'SUCCESS',
    };

    try {
      await withRetry(
        () => {
          const prisma = getPrisma();
          return prisma.auditLog.create({ data });
        },
        {
          maxRetries: 2,
          baseDelayMs: 200,
          label: 'Audit log write',
        }
      );
    } catch (error) {
      log.error('Audit log lost after retries', error instanceof Error ? error : undefined);
    }
  }

  extractRequestContext(req: Request): { userId?: string; ipAddress: string; userAgent: string } {
    return {
      userId: req.user?.userId,
      ipAddress: req.ip || req.socket.remoteAddress || 'unknown',
      userAgent: req.headers['user-agent'] || 'unknown',
    };
  }

  async logAccess(req: Request, resource: string, resourceId?: string, details?: Record<string, unknown>): Promise<void> {
    const ctx = this.extractRequestContext(req);
    await this.log({
      userId: ctx.userId,
      action: 'READ',
      resource,
      resourceId,
      details,
      ipAddress: ctx.ipAddress,
      userAgent: ctx.userAgent,
    });
  }

  /**
   * Audit Log Retention Policy
   * Compliance policies typically require audit log retention (e.g., 6 years for HIPAA).
   */
  static readonly RETENTION_YEARS = 6;

  async enforceRetentionPolicy(): Promise<{ deleted: number; retainedBefore: Date }> {
    const retentionCutoff = new Date();
    retentionCutoff.setFullYear(retentionCutoff.getFullYear() - AuditService.RETENTION_YEARS);

    try {
      const prisma = getPrisma();
      const result = await prisma.auditLog.deleteMany({
        where: { createdAt: { lt: retentionCutoff } },
      });
      return { deleted: result.count, retainedBefore: retentionCutoff };
    } catch (error) {
      log.error('Retention policy enforcement failed', error instanceof Error ? error : undefined);
      return { deleted: 0, retainedBefore: retentionCutoff };
    }
  }
}

export const auditService = new AuditService();
```

**Key pattern:** Audit logging uses `res.on('finish', ...)` to log after the response is sent, so it never blocks the request. The service uses `withRetry` for resilience and fire-and-forget semantics.

---

## Input Validation with Zod

All request data validated with Zod. String fields must have `.trim().max()`. Pagination capped at 100:

```typescript
// Example route handler
const querySchema = z.object({
  page: z.string().optional().transform(v => parseInt(v || '1', 10)).pipe(z.number().min(1)),
  limit: z.string().optional().transform(v => parseInt(v || '20', 10)).pipe(z.number().min(1).max(100)),
  elderlyUserId: z.string().trim().max(100).optional(),
});

const bodySchema = z.object({
  name: z.string().trim().min(1).max(200),
  phone: z.string().trim().max(50).optional(),
  notes: z.string().trim().max(5000).optional(),
});

router.post('/', authenticate, async (req, res, next) => {
  try {
    const body = bodySchema.parse(req.body);
    // ... handler logic
  } catch (error) {
    next(error); // Zod errors caught by errorHandler
  }
});
```

---

## Webhook Signature Validation

**Stripe webhooks:** Use `stripe.webhooks.constructEvent()` with the raw request body:

```typescript
// Preserve raw body in Express middleware (required for Stripe signature verification)
app.use(express.json({
  limit: '10mb',
  verify: (req: any, _res, buf) => {
    req.rawBody = buf.toString();
  },
}));

// In webhook route:
const rawBody = (req as any).rawBody;
const event = stripeService.constructEvent(rawBody, signature);
```

**OIDC token verification** for Cloud Scheduler internal routes:

```typescript
export async function oidcAuth(req: Request, res: Response, next: NextFunction): Promise<void> {
  if (env.NODE_ENV === 'development') return next(); // Bypass for local testing

  const token = authHeader.slice(7);
  try {
    const ticket = await oauthClient.verifyIdToken({
      idToken: token,
      audience: env.APP_URL,  // Must match Cloud Scheduler's target audience
    });

    const payload = ticket.getPayload();
    if (env.GCP_SERVICE_ACCOUNT && payload.email !== env.GCP_SERVICE_ACCOUNT) {
      res.status(401).json({ error: 'Unauthorized service account' });
      return;
    }
    next();
  } catch (error) {
    res.status(401).json({ error: 'Invalid OIDC token' });
  }
}
```

---

## Ownership Verification (IDOR Prevention)

Every route that accesses a resource must verify the requesting user owns it:

```typescript
// Example: GET /api/calls/:id
router.get('/:id', authenticate, async (req, res, next) => {
  const call = await prisma.call.findUnique({
    where: { id: req.params.id },
    include: { elderlyUser: { include: { familyMember: true } } },
  });

  if (!call) throw new NotFoundError('Call not found');

  // SECURITY: Verify requesting user owns this call's elderly user
  if (call.elderlyUser?.familyMember?.userId !== req.user!.userId) {
    throw new ForbiddenError('Access denied');
  }

  res.json(call);
});
```

---

## Gotchas

1. **`trust proxy` must be set in production** for correct client IP in rate limiting. Cloud Run uses one proxy level: `app.set('trust proxy', 1)`.

2. **Pino's argument order is reversed:** `child.info(data, msg)` not `child.info(msg, data)`. The wrapper in `createLogger` normalizes this.

3. **Audit middleware must be applied BEFORE route handlers** (`app.use('/api', auditMiddleware)` before `app.use('/api', router)`).

4. **`.unref()` on `setInterval` timers** prevents them from keeping Node.js alive during shutdown. Essential for cleanup intervals.

5. **Prisma `P2002` (unique constraint)** and `P2025` (record not found) errors should map to 409 and 404 respectively, not 500.

---

## Checklist

- [ ] Implement `createLogger()` with PII auto-redaction (phone, email, name fields)
- [ ] Map pino levels to GCP Cloud Logging severity in production
- [ ] Create custom error classes (`BadRequestError`, `NotFoundError`, etc.)
- [ ] Error handler: expose `error.message` only for `AppError` and `ZodError`, never for unknown errors
- [ ] Implement client-side `ErrorUtils` that only passes through 400/409/422 backend messages
- [ ] Add Zod validation on all route inputs with `.trim().max()` on strings
- [ ] Cap pagination `limit` to 100 via `.pipe(z.number().min(1).max(100))`
- [ ] Set up three-layer rate limiting (auth/global/user-based)
- [ ] Implement account lockout (10 failures -> 30-min lock)
- [ ] Add audit middleware for sensitive data routes
- [ ] Implement breach detection service
- [ ] Preserve raw body for webhook signature validation
- [ ] Add OIDC auth for internal/scheduled routes
- [ ] Verify ownership on every resource access
- [ ] Set `trust proxy` in production Express config

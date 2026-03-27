# 09 - Testing

## Why

Tests with Prisma mocks provide fast, isolated unit tests without a database. Supertest enables integration testing of Express routes without starting a server. On the Flutter side, Riverpod container overrides let you inject mock dependencies cleanly.

---

## Backend Test Setup (Jest + Prisma Mocks)

```typescript
// engine/src/__tests__/setup.ts

/**
 * Jest Global Test Setup
 * This file runs before all tests (setupFilesAfterEnv).
 *
 * IMPORTANT: jest.mock calls here apply GLOBALLY to all test files.
 * This ensures mocks are registered before any module imports.
 */

import { mockDeep, mockReset, DeepMockProxy } from 'jest-mock-extended';
import { PrismaClient } from '@prisma/client';

// Set test environment
process.env.NODE_ENV = 'test';
process.env.JWT_SECRET = 'test-jwt-secret-key-minimum-32-characters-long';
process.env.DATABASE_URL = 'postgresql://test:test@localhost:5432/myapp_test';
process.env.PORT = '3001';
process.env.FIREBASE_PROJECT_ID = 'myapp-test';
process.env.APP_URL = 'http://localhost:3001';
process.env.CORS_ORIGINS = 'http://localhost:3000,http://localhost:5173';
process.env.TELNYX_TEST_PHONE_NUMBER = '+15555550199';

// ============================================
// GLOBAL MOCKS -- must be here so they apply
// before any test module imports route/middleware code
// ============================================

// 1. Prisma mock -- intercepts `import { prisma } from '../index.js'`
const prismaMock = mockDeep<PrismaClient>();
(global as any).__prismaMock__ = prismaMock;

jest.mock('../index', () => ({
  __esModule: true,
  prisma: (global as any).__prismaMock__,
}));

// 2. Firebase Admin mock -- intercepts firebase-admin imports
const mockVerifyIdToken = jest.fn();
const mockGetAuth = jest.fn().mockReturnValue({
  verifyIdToken: mockVerifyIdToken,
});
(global as any).__mockVerifyIdToken__ = mockVerifyIdToken;
(global as any).__mockGetAuth__ = mockGetAuth;

jest.mock('firebase-admin/auth', () => ({
  getAuth: (global as any).__mockGetAuth__,
}));

jest.mock('firebase-admin/app', () => ({
  initializeApp: jest.fn(),
  getApps: jest.fn().mockReturnValue([{ name: 'test-app' }]),
  cert: jest.fn(),
}));

// 3. Cloud Scheduler mock
const mockCloudSchedulerService = {
  createJob: jest.fn().mockResolvedValue('projects/myapp-test/locations/us-central1/jobs/test'),
  updateJob: jest.fn().mockResolvedValue(undefined),
  deleteJob: jest.fn().mockResolvedValue(undefined),
  pauseJob: jest.fn().mockResolvedValue(undefined),
  resumeJob: jest.fn().mockResolvedValue(undefined),
  buildCronExpression: jest.fn().mockReturnValue('0 10 * * 1,2,3,4,5'),
};
(global as any).__mockCloudSchedulerService__ = mockCloudSchedulerService;

jest.mock('../services/cloud-scheduler.service', () => ({
  __esModule: true,
  cloudSchedulerService: (global as any).__mockCloudSchedulerService__,
}));

// 4. Aggregate service mock
const mockAggregateService = {
  getFacilityDashboard: jest.fn().mockResolvedValue({}),
  getResidentsWithAlerts: jest.fn().mockResolvedValue({ highLoneliness: [], healthConcerns: [] }),
};
(global as any).__mockAggregateService__ = mockAggregateService;

jest.mock('../services/aggregate.service', () => ({
  __esModule: true,
  aggregateService: (global as any).__mockAggregateService__,
}));

// ============================================
// RESET MOCKS BEFORE EACH TEST
// ============================================

beforeEach(() => {
  mockReset(prismaMock);
  mockVerifyIdToken.mockReset();
  Object.values(mockCloudSchedulerService).forEach(fn => (fn as jest.Mock).mockClear());
  mockCloudSchedulerService.createJob.mockResolvedValue('projects/myapp-test/locations/us-central1/jobs/test');
  Object.values(mockAggregateService).forEach(fn => (fn as jest.Mock).mockReset());
  mockAggregateService.getFacilityDashboard.mockResolvedValue({});
  mockAggregateService.getResidentsWithAlerts.mockResolvedValue({ highLoneliness: [], healthConcerns: [] });
});

// ============================================
// OTHER TEST CONFIG
// ============================================

// Mock process.exit to prevent test runner from exiting
const mockExit = jest.spyOn(process, 'exit').mockImplementation((code?: number | string | null) => {
  return undefined as never;
});

jest.setTimeout(10000);

// Suppress console output during tests
global.console = {
  ...console,
  log: jest.fn(),
  debug: jest.fn(),
  info: jest.fn(),
  warn: jest.fn(),
  error: jest.fn(),
};

afterAll(async () => {
  mockExit.mockRestore();
});
```

**Key patterns:**
- Mocks are registered **globally** in `setup.ts` via `jest.mock()` -- they apply before any test module imports
- `mockDeep<PrismaClient>()` creates a deeply-mocked Prisma client that auto-generates mock functions for every model method
- `(global as any).__prismaMock__` shares the mock between setup and test files
- `mockReset()` in `beforeEach` clears all mock state between tests

---

## Mock Factories

```typescript
// engine/src/__tests__/mocks/prisma.mock.ts

import { DeepMockProxy } from 'jest-mock-extended';
import { PrismaClient } from '@prisma/client';

export const prismaMock = (global as any).__prismaMock__ as DeepMockProxy<PrismaClient>;

export const createMockUser = (overrides: Record<string, any> = {}) => ({
  id: 'user-123',
  email: 'test@example.com',
  name: 'Test User',
  phone: null,
  firebaseUid: null,
  passwordHash: null,
  createdAt: new Date(),
  updatedAt: new Date(),
  deletedAt: null,
  tokensRevokedAt: null,
  ...overrides,
});

export const createMockFamilyMember = (overrides: Record<string, any> = {}) => ({
  id: 'fm-123',
  userId: 'user-123',
  relationship: 'Son',
  receiveNotifications: true,
  receiveEmails: true,
  fcmToken: null,
  createdAt: new Date(),
  updatedAt: new Date(),
  ...overrides,
});

export const createMockElderlyUser = (overrides: Record<string, any> = {}) => ({
  id: 'eu-123',
  familyMemberId: 'fm-123',
  preferredName: 'Grandma',
  phoneNumber: '+1987654321',
  fullName: null,
  timezone: 'America/New_York',
  interests: ['gardening', 'cooking'],
  status: 'ACTIVE',
  deletedAt: null,
  lastCallAt: null,
  totalCalls: 0,
  createdAt: new Date(),
  updatedAt: new Date(),
  ...overrides,
});

export const createMockSubscription = (overrides: Record<string, any> = {}) => ({
  id: 'sub-123',
  familyMemberId: 'fm-123',
  tier: 'ESSENTIALS',
  status: 'ACTIVE',
  stripeCustomerId: 'cus_test',
  stripeSubscriptionId: 'sub_test',
  currentPeriodStart: new Date(),
  currentPeriodEnd: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000),
  cancelAtPeriodEnd: false,
  trialStartDate: null,
  trialEndDate: null,
  trialCallsUsed: null,
  trialCallsLimit: null,
  createdAt: new Date(),
  updatedAt: new Date(),
  ...overrides,
});
```

---

## Test Data Fixtures

```typescript
// engine/src/__tests__/fixtures/test-data.ts

export const createElderlyUserBody = (overrides = {}) => ({
  preferredName: 'Grandma',
  phoneNumber: '+1234567890',
  interests: ['gardening', 'cooking', 'reading'],
  communicationStyle: 'WARM',
  ...overrides,
});

export const createUpdateBody = (overrides = {}) => ({
  content: 'Had a great day at the park!',
  category: 'DAILY_LIFE',
  priority: 'NORMAL',
  elderlyUserIds: ['eu-123'],
  ...overrides,
});

export const createScheduleBody = (overrides = {}) => ({
  elderlyUserId: 'eu-123',
  dayOfWeek: 1, // Monday
  preferredTime: '14:00',
  duration: 30,
  enabled: true,
  ...overrides,
});

export const createTelnyxWebhookEvent = (eventType: string, overrides = {}) => ({
  data: {
    event_type: eventType,
    id: 'event-123',
    occurred_at: new Date().toISOString(),
    payload: {
      call_control_id: 'call-ctrl-123',
      call_leg_id: 'leg-123',
      call_session_id: 'session-123',
      from: '+1234567890',
      to: '+0987654321',
      direction: 'outgoing',
      client_state: Buffer.from(JSON.stringify({
        elderlyUserId: 'eu-123',
        elderlyUserName: 'Grandma',
      })).toString('base64'),
      ...overrides,
    },
    record_type: 'event',
  },
  meta: {
    attempt: 1,
    delivered_to: 'https://example.com/webhook',
  },
});

export const webhookEvents = {
  CALL_INITIATED: 'call.initiated',
  CALL_ANSWERED: 'call.answered',
  CALL_HANGUP: 'call.hangup',
  SPEAK_STARTED: 'call.speak.started',
  SPEAK_ENDED: 'call.speak.ended',
  STREAMING_STARTED: 'call.streaming.started',
  STREAMING_STOPPED: 'call.streaming.stopped',
};
```

**Pattern:** Factory functions with spread overrides let tests customize only the fields they care about.

---

## Integration Test Pattern (Supertest)

```typescript
// engine/src/__tests__/integration/routes/auth.routes.test.ts

import request from 'supertest';
import jwt from 'jsonwebtoken';
import { createApp } from '../../../app';
import { prismaMock, createMockUser, createMockFamilyMember } from '../../mocks/prisma.mock';

const app = createApp();
const JWT_SECRET = 'test-jwt-secret-key-minimum-32-characters-long';

describe('Auth Routes', () => {
  describe('GET /api/auth/me', () => {
    it('should return user profile with valid token', async () => {
      const mockUser = createMockUser();
      const mockFamilyMember = createMockFamilyMember({ userId: mockUser.id });

      // Generate a test token
      const token = jwt.sign(
        { userId: mockUser.id, email: mockUser.email },
        JWT_SECRET,
        { expiresIn: '1h' },
      );

      // Mock the chain: token revocation check -> auth lookup -> route handler
      prismaMock.revokedToken.findUnique.mockResolvedValue(null);
      prismaMock.user.findUnique.mockResolvedValueOnce(mockUser as any);
      prismaMock.user.findUnique.mockResolvedValueOnce({
        ...mockUser,
        familyMember: { ...mockFamilyMember, elderlyUsers: [], subscription: null },
        adminFacility: null,
      } as any);

      const response = await request(app)
        .get('/api/auth/me')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(response.body.id).toBe(mockUser.id);
      expect(response.body.email).toBe(mockUser.email);
    });

    it('should reject request without token', async () => {
      await request(app)
        .get('/api/auth/me')
        .expect(401);
    });

    it('should reject revoked tokens', async () => {
      const token = jwt.sign(
        { userId: 'user-123', email: 'test@example.com' },
        JWT_SECRET,
        { expiresIn: '1h' },
      );

      prismaMock.revokedToken.findUnique.mockResolvedValue({ id: '1', tokenHash: 'hash' } as any);

      await request(app)
        .get('/api/auth/me')
        .set('Authorization', `Bearer ${token}`)
        .expect(401);
    });
  });
});
```

**Key pattern:** The `authenticate` middleware makes **multiple Prisma calls** in sequence:
1. `revokedToken.findUnique` -- token revocation check
2. `user.findUnique` -- auth middleware user lookup
3. `user.findUnique` -- route handler user lookup (with includes)

You must mock each call with `mockResolvedValueOnce` in order. Using `mockResolvedValue` (without `Once`) would return the same value for all calls.

---

## Testing Webhook Signature Validation

```typescript
it('should reject invalid Stripe signature', async () => {
  await request(app)
    .post('/api/webhooks/stripe')
    .set('stripe-signature', 'invalid-sig')
    .send({ type: 'checkout.session.completed' })
    .expect(400);
});
```

---

## Flutter Test Setup (Riverpod Overrides)

```dart
// app-client/test/unit/providers/auth_provider_test.dart

import 'package:firebase_auth_mocks/firebase_auth_mocks.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockApiClient extends Mock implements ApiClient {}

/// Creates a ProviderContainer with mock dependencies
ProviderContainer createTestContainer({
  required MockFirebaseAuth mockAuth,
  required MockApiClient mockApiClient,
}) {
  return ProviderContainer(
    overrides: [
      firebaseAuthProvider.overrideWithValue(mockAuth),
      apiClientProvider.overrideWith((ref) => mockApiClient),
    ],
  );
}

/// Standard backend user JSON for tests
Map<String, dynamic> testUserJson({
  String id = 'user-123',
  String email = 'test@example.com',
  String name = 'Test User',
}) {
  return {
    'id': id,
    'email': email,
    'name': name,
    'familyMember': {
      'id': 'fm-123',
      'relationship': 'Son',
      'receiveNotifications': true,
      'receiveEmails': true,
      'elderlyUsers': <dynamic>[],
      'subscription': null,
    },
  };
}

/// Sets up MockApiClient to return success for /auth/sync
void setupSyncSuccess(MockApiClient mockApiClient, {Map<String, dynamic>? userJson}) {
  when(() => mockApiClient.post(
    '/auth/sync',
    data: any(named: 'data'),
    queryParameters: any(named: 'queryParameters'),
    options: any(named: 'options'),
  )).thenAnswer((_) async => Response(
    requestOptions: RequestOptions(path: '/auth/sync'),
    statusCode: 200,
    data: userJson ?? testUserJson(),
  ));
}
```

**Key patterns:**
- `ProviderContainer` with `overrides` replaces real providers with mocks
- `firebaseAuthProvider.overrideWithValue()` injects `MockFirebaseAuth`
- `apiClientProvider.overrideWith()` injects `MockApiClient`
- Use `when(() => mock.method(...)).thenAnswer(...)` from `mocktail` (not `mockito`)
- `firebase_auth_mocks` package provides a configurable `MockFirebaseAuth` with `MockUser`

---

## Security Regression Tests

```typescript
describe('Security Regression Tests', () => {
  it('should reject requests without auth token on protected routes', async () => {
    const protectedRoutes = [
      { method: 'get', path: '/api/calls' },
      { method: 'get', path: '/api/elderly-users' },
      { method: 'post', path: '/api/schedules' },
      { method: 'get', path: '/api/subscriptions/current' },
      { method: 'get', path: '/api/updates' },
      { method: 'get', path: '/api/preparations' },
    ];

    for (const route of protectedRoutes) {
      const response = await request(app)[route.method](route.path);
      expect(response.status).toBe(401);
    }
  });

  it('should return generic error messages for 500 errors in production', async () => {
    // Verify no error.message leakage
  });

  it('should enforce pagination limits', async () => {
    // Test that limit > 100 is rejected
  });
});
```

---

## Jest Configuration

```json
// engine/package.json (jest section)
{
  "jest": {
    "preset": "ts-jest",
    "testEnvironment": "node",
    "roots": ["<rootDir>/src"],
    "setupFilesAfterFramework": [],
    "setupFilesAfterEnv": ["<rootDir>/src/__tests__/setup.ts"],
    "moduleNameMapper": {
      "^(\\.\\.?\\/.+)\\.js$": "$1"
    },
    "transform": {
      "^.+\\.tsx?$": ["ts-jest", {
        "useESM": false,
        "tsconfig": "tsconfig.json"
      }]
    },
    "testPathIgnorePatterns": [
      "/node_modules/",
      "/dist/",
      "/__tests__/setup\\.ts$",
      "/__tests__/mocks/",
      "/__tests__/fixtures/"
    ]
  }
}
```

---

## Gotchas

1. **`jest.mock()` calls must be in `setup.ts`** (not in individual test files) for mocks that need to apply before module imports. Jest hoists `jest.mock()` calls to the top of each file, but cross-module mocks need to be global.

2. **`mockResolvedValueOnce` vs `mockResolvedValue`:** Use `Once` variants when the same method is called multiple times in a test (e.g., `user.findUnique` in auth middleware then in route handler).

3. **`process.env.ADMIN_EMAILS` must be set before importing modules** that use `env.ts`, because Zod parses env vars at import time.

4. **Flutter test package matches your `pubspec.yaml` name**, so test imports use `package:yourapp/...` even though source code uses relative imports.

5. **`console.log` suppression in tests** prevents noisy output. But if a test fails, you lose debug info. Consider selective suppression.

6. **Mock cleanup with `.unref()` timers:** `setInterval(...).unref()` prevents hanging test processes from timer callbacks that reference mocked modules.

7. **`SecureStorage.setMockStorage()`** must be called in Flutter test setUp to avoid platform channel errors from `flutter_secure_storage`.

---

## Checklist

- [ ] Set up Jest with `jest-mock-extended` for Prisma mocking
- [ ] Create `setup.ts` with global mocks (Prisma, Firebase Admin, external services)
- [ ] Create mock factories for all models (`createMockUser`, `createMockFamilyMember`, etc.)
- [ ] Create test data fixtures with override patterns
- [ ] Write integration tests using `supertest` against `createApp()`
- [ ] Mock the full auth chain (revocation check -> user lookup -> handler)
- [ ] Set up Flutter tests with `ProviderContainer` overrides
- [ ] Use `firebase_auth_mocks` for Firebase Auth testing
- [ ] Use `mocktail` for Flutter API client mocking
- [ ] Create security regression test suite
- [ ] Suppress console output in tests
- [ ] Call `SecureStorage.setMockStorage()` in Flutter test setUp

# 08 - Database (Prisma)

## Why

Prisma provides type-safe database access, automatic migration management, and a clean schema definition language. Combined with PostgreSQL, it handles relational data, full-text search, and vector embeddings (pgvector).

---

## Schema Design Patterns

```prisma
// engine/prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [vector]
}

// ============================================
// USER MODELS
// ============================================

// Soft delete pattern: deletedAt is null for active records
model User {
  id            String    @id @default(cuid())
  firebaseUid   String?   @unique  // Nullable for migration
  email         String    @unique
  passwordHash  String?
  name          String
  phone         String?
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  deletedAt         DateTime?  // Soft delete
  tokensRevokedAt   DateTime?  // All tokens issued before this timestamp are invalid

  // Relations
  familyMember  FamilyMember?
  adminFacility PartnerFacility?  // If this user is a facility admin
  feedbacks     Feedback[]

  @@map("users")  // Lowercase table name
}

// One-to-one extension pattern: separate concerns from User
model FamilyMember {
  id                    String           @id @default(cuid())
  userId                String           @unique
  relationship          String           // e.g., "son", "daughter", "spouse"
  receiveNotifications  Boolean          @default(true)
  receiveEmails         Boolean          @default(true)
  fcmToken              String?          // Firebase Cloud Messaging token
  createdAt             DateTime         @default(now())
  updatedAt             DateTime         @updatedAt

  // Relations
  user                  User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  elderlyUsers          ElderlyUser[]
  updates               FamilyUpdate[]
  subscription          Subscription?
  callPreparations      CallPreparation[]

  @@map("family_members")
}

model ElderlyUser {
  id                    String           @id @default(cuid())
  phoneNumber           String           @unique
  preferredName         String
  fullName              String?
  timezone              String           @default("America/New_York")
  interests             String[]         @default([])
  familyContext         String?          // Background info for AI context
  communicationNotes    String?          // Special instructions for AI
  status                UserStatus       @default(ONBOARDING)
  voiceId               String?          // ElevenLabs voice ID for TTS
  photoUrl              String?          // Photo URL (Firebase Storage)

  // Consent fields
  consentGivenAt        DateTime?
  directConsentStatus   DirectConsentStatus  @default(NOT_ASKED)
  directConsentDate     DateTime?
  consentAttempts       Int                  @default(0)

  // Metrics
  lastCallAt            DateTime?
  totalCalls            Int              @default(0)

  createdAt             DateTime         @default(now())
  updatedAt             DateTime         @updatedAt
  deletedAt             DateTime?

  // B2B Channel Partner -- Optional facility link
  facilityId            String?
  facility              PartnerFacility?  @relation(fields: [facilityId], references: [id], onDelete: SetNull)

  // Relations
  familyMemberId        String
  familyMember          FamilyMember     @relation(fields: [familyMemberId], references: [id], onDelete: Cascade)
  calls                 Call[]
  schedules             CallSchedule[]
  learnedFacts          LearnedFact[]
  memoryEmbeddings      MemoryEmbedding[]

  @@map("elderly_users")
}

// ============================================
// CALL MODELS
// ============================================

// COMPLIANCE: onDelete SetNull preserves records when elderly user is deleted
model Call {
  id                String        @id @default(cuid())
  twilioCallSid     String?       @unique
  direction         CallDirection
  status            CallStatus    @default(INITIATED)
  failureReason     CallFailureReason?

  // Timing
  scheduledAt       DateTime?
  startedAt         DateTime?
  endedAt           DateTime?
  durationSeconds   Int?

  // AI Analysis
  summary           String?
  topics            String[]      @default([])
  sentimentScore    Float?

  elderlyUserId     String?       // Nullable for compliance retention
  elderlyUser       ElderlyUser?  @relation(fields: [elderlyUserId], references: [id], onDelete: SetNull)

  messages          CallMessage[]
  learnedFacts      LearnedFact[]

  @@map("calls")
}

// ============================================
// SUBSCRIPTION MODELS
// ============================================

model Subscription {
  id                    String             @id @default(cuid())
  familyMemberId        String             @unique
  tier                  SubscriptionTier
  status                SubscriptionStatus @default(ACTIVE)

  // Stripe
  stripeCustomerId      String?
  stripeSubscriptionId  String?            @unique
  currentPeriodStart    DateTime?
  currentPeriodEnd      DateTime?
  cancelAtPeriodEnd     Boolean            @default(false)

  // Trial
  trialStartDate        DateTime?
  trialEndDate          DateTime?
  trialCallsUsed        Int?
  trialCallsLimit       Int?

  createdAt             DateTime           @default(now())
  updatedAt             DateTime           @updatedAt

  familyMember          FamilyMember       @relation(fields: [familyMemberId], references: [id], onDelete: Cascade)

  @@map("subscriptions")
}
```

**Key patterns:**
- **`@default(cuid())`:** CUIDs are URL-safe, sortable, and don't leak creation order like sequential IDs
- **`@@map("table_name")`:** Lowercase table names match PostgreSQL conventions
- **Soft deletes:** `deletedAt` field instead of `DELETE` -- Compliance may require data retention
- **`onDelete: SetNull`** on Call -> ElderlyUser: preserves call records when the elderly user is deleted
- **`onDelete: Cascade`** on FamilyMember -> User: deleting a user deletes all their data
- **Separate notification flags:** `receiveNotifications` and `receiveEmails` are independent booleans
- **`@unique`** on `stripeSubscriptionId`: prevents duplicate Stripe subscription records

---

## Migration Workflow

### Development

```bash
# 1. Edit prisma/schema.prisma
# 2. Generate migration
cd engine
npx prisma migrate dev --name describe_the_change

# 3. Verify the generated SQL in prisma/migrations/
# 4. Update affected routes/services
# 5. Update Flutter models if API response changed
# 6. Update api_endpoints.dart if new routes added
```

### Production

```bash
# 1. Start Cloud SQL Auth Proxy
cloud_sql_proxy -instances=my-gcp-project:us-central1:myapp-db=tcp:5432 &

# 2. Apply migrations
DATABASE_URL="postgresql://postgres:PASSWORD@localhost:5432/myapp?sslmode=require" npx prisma migrate deploy

# 3. Deploy code
gcloud builds submit --config=engine/cloudbuild.yaml --project=my-gcp-project \
  --substitutions=COMMIT_SHA=$(git rev-parse --short HEAD)
```

### Staging

```bash
# Staging uses a separate database on the same instance
DATABASE_URL="postgresql://postgres:PASSWORD@localhost:5432/myapp_staging?sslmode=require" npx prisma migrate deploy
```

---

## Connection String Format

```
postgresql://user:password@host:5432/database?schema=public&connection_limit=3&sslmode=require
```

- **`connection_limit=3`:** Each Cloud Run instance gets a small pool. With 5 max instances, that's 15 connections total.
- **`sslmode=require`:** Required for direct TCP connections in production (security best practice).
- **Cloud SQL Auth Proxy uses Unix sockets:** `/cloudsql/project:region:instance` -- no `sslmode` needed.

---

## Database RBAC (Complete)

```sql
-- engine/prisma/rbac-setup.sql

-- Database RBAC Setup
-- Creates a restricted application role with minimum required privileges.
-- Run this script as the database owner AFTER running Prisma migrations.

-- 1. Create the restricted application role (if not exists)
DO $$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'myapp_role') THEN
    CREATE ROLE myapp_role WITH LOGIN PASSWORD 'CHANGE_ME_BEFORE_RUNNING';
    RAISE NOTICE 'Created role myapp_role -- REMEMBER TO CHANGE THE PASSWORD';
  ELSE
    RAISE NOTICE 'Role myapp_role already exists';
  END IF;
END $$;

-- 2. Grant connect to the database
GRANT CONNECT ON DATABASE myapp TO myapp_role;

-- 3. Grant usage on the public schema
GRANT USAGE ON SCHEMA public TO myapp_role;

-- 4. Grant DML privileges on all existing tables (SELECT, INSERT, UPDATE, DELETE)
-- NO: CREATE, DROP, ALTER, TRUNCATE, REFERENCES, TRIGGER
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp_role;

-- 5. Grant usage on all sequences (needed for auto-increment/serial columns)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO myapp_role;

-- 6. Set default privileges for future tables created by the owner
-- This ensures new tables from Prisma migrations automatically get DML grants
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO myapp_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO myapp_role;

-- 7. Revoke dangerous privileges explicitly
REVOKE CREATE ON SCHEMA public FROM myapp_role;

-- Verification
DO $$
BEGIN
  RAISE NOTICE 'RBAC setup complete. The myapp_role role has:';
  RAISE NOTICE '  - SELECT, INSERT, UPDATE, DELETE on all tables';
  RAISE NOTICE '  - USAGE on all sequences';
  RAISE NOTICE '  - NO schema modification privileges';
  RAISE NOTICE '';
  RAISE NOTICE 'Update your app DATABASE_URL to use myapp_role.';
  RAISE NOTICE 'Keep using the admin role for Prisma migrations.';
END $$;
```

**Two roles:**
- **Admin role** (e.g., `postgres`): Used only for running `prisma migrate deploy`
- **App role** (e.g., `myapp_role`): Used in `DATABASE_URL` for the application. Cannot `CREATE`, `DROP`, `ALTER`, or `TRUNCATE` tables.

---

## Seeding

```typescript
// engine/prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  const demoUser = await prisma.user.upsert({
    where: { email: 'demo@example.com' },
    update: {},
    create: {
      email: 'demo@example.com',
      name: 'Demo User',
      firebaseUid: 'firebase-demo-uid',
    },
  });

  await prisma.familyMember.upsert({
    where: { userId: demoUser.id },
    update: {},
    create: {
      userId: demoUser.id,
      relationship: 'Son',
    },
  });
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

---

## Query Patterns

### Soft Delete Filtering

```typescript
// Always filter by deletedAt: null
const users = await prisma.elderlyUser.findMany({
  where: { familyMemberId, deletedAt: null },
});
```

### Pagination with Cap

```typescript
const querySchema = z.object({
  page: z.string().optional().transform(v => parseInt(v || '1', 10)).pipe(z.number().min(1)),
  limit: z.string().optional().transform(v => parseInt(v || '20', 10)).pipe(z.number().min(1).max(100)),
});

const { page, limit } = querySchema.parse(req.query);
const skip = (page - 1) * limit;

const [items, total] = await Promise.all([
  prisma.call.findMany({ where: { ... }, skip, take: limit, orderBy: { createdAt: 'desc' } }),
  prisma.call.count({ where: { ... } }),
]);

res.json({ items, pagination: { page, limit, total, totalPages: Math.ceil(total / limit) } });
```

### Transaction Pattern

```typescript
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({ data: { ... } });
  const familyMember = await tx.familyMember.create({ data: { userId: user.id, ... } });
  const subscription = await tx.subscription.create({ data: { familyMemberId: familyMember.id, ... } });
  return { user, familyMember, subscription };
});
```

---

## Gotchas

1. **Prisma `P2002` unique constraint errors** should be caught in the error handler and mapped to 409, not 500.

2. **`@updatedAt` auto-updates** on every Prisma `update()` call. No need to set it manually.

3. **Soft deletes require filtering** in every query: `where: { deletedAt: null }`.

4. **`cuid()` generates ~25 character IDs.** If you need shorter IDs for URLs, consider `nanoid`.

5. **Vector extension** (`extensions = [vector]`) requires PostgreSQL with the `pgvector` extension installed.

---

## Checklist

- [ ] Set up Prisma schema with `cuid()` IDs and `@@map()` for table names
- [ ] Add soft delete pattern (`deletedAt` field) on models requiring data retention
- [ ] Set appropriate `onDelete` behavior (Cascade vs SetNull)
- [ ] Configure connection string with `connection_limit` and `sslmode`
- [ ] Create RBAC setup script with restricted app role
- [ ] Set up seed script for demo data
- [ ] Implement migration workflow (dev, staging, production)
- [ ] Add `tokensRevokedAt` to User model for bulk token revocation
- [ ] Always filter by `deletedAt: null` in queries
- [ ] Cap pagination `limit` to 100

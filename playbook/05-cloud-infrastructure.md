# 05 - Cloud Infrastructure (GCP)

## Why

Cloud Run provides serverless containers with automatic scaling, HTTPS, and zero infrastructure management. Cloud Build enables CI/CD without a separate CI system. Cloud SQL manages PostgreSQL with automatic backups and the Cloud SQL Auth Proxy handles encrypted connections.

---

## Cloud Build Pipeline (Production)

```yaml
# engine/cloudbuild.yaml

steps:
  # Step 1: Build the Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'gcr.io/$PROJECT_ID/myapp-engine:$COMMIT_SHA'
      - '-t'
      - 'gcr.io/$PROJECT_ID/myapp-engine:latest'
      - './engine'
    id: 'build-image'

  # Step 2: Push the Docker image to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/myapp-engine:$COMMIT_SHA']
    id: 'push-image'
    waitFor: ['build-image']

  # Step 3: Push latest tag
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/myapp-engine:latest']
    id: 'push-latest'
    waitFor: ['build-image']

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'myapp-engine'
      - '--image'
      - 'gcr.io/$PROJECT_ID/myapp-engine:$COMMIT_SHA'
      - '--region'
      - '${_REGION}'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'
      - '--port'
      - '8080'
      - '--memory'
      - '512Mi'
      - '--cpu'
      - '1'
      - '--min-instances'
      - '0'
      - '--max-instances'
      - '5'
      - '--timeout'
      - '300'
      - '--concurrency'
      - '80'
      - '--cpu-boost'
      # Cloud SQL connection
      - '--add-cloudsql-instances'
      - '${_CLOUDSQL_CONNECTION}'
      # Secrets from Secret Manager
      - '--set-secrets'
      - 'DATABASE_URL=DATABASE_URL:latest,JWT_SECRET=JWT_SECRET:latest,...'
      # Environment variables (use ^ separator since CORS_ORIGINS contains commas)
      - '--set-env-vars'
      - '^##^NODE_ENV=production##APP_URL=${_APP_URL}##CORS_ORIGINS=${_CORS_ORIGINS}##FIREBASE_PROJECT_ID=my-gcp-project##...'
      # Service account
      - '--service-account'
      - '${_SERVICE_ACCOUNT}'
    id: 'deploy-cloud-run'
    waitFor: ['push-image']

  # Step 5: Verify deployment health
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: bash
    args:
      - '-c'
      - |
        echo "Waiting 15s for revision to stabilize..."
        sleep 15
        HTTP_CODE=$$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 ${_APP_URL}/health)
        if [ "$$HTTP_CODE" = "200" ]; then
          echo "Health check PASSED"
        else
          echo "Health check FAILED"
          exit 1
        fi
    id: 'verify-deploy'
    waitFor: ['deploy-cloud-run']

substitutions:
  _REGION: 'us-central1'
  _CLOUDSQL_CONNECTION: 'my-gcp-project:us-central1:myapp-db'
  _APP_URL: 'https://myapp-engine-XXXXXXXXX.us-central1.run.app'
  _CORS_ORIGINS: 'https://my-gcp-project.web.app,https://example.com,https://app.example.com,https://myapp.web.app'
  _SERVICE_ACCOUNT: 'myapp-backend@my-gcp-project.iam.gserviceaccount.com'

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'

images:
  - 'gcr.io/$PROJECT_ID/myapp-engine:$COMMIT_SHA'
  - 'gcr.io/$PROJECT_ID/myapp-engine:latest'

timeout: '900s'
```

**Key patterns:**
- **`^##^` separator:** When env var values contain commas (like CORS_ORIGINS), use a custom separator
- **`--set-secrets`:** Reads from Secret Manager at runtime (not baked into image)
- **`--cpu-boost`:** Temporary CPU boost during startup for faster cold starts
- **Health check step:** Verifies the new revision is serving traffic

---

## Multi-Stage Dockerfile

```dockerfile
# engine/Dockerfile

# Stage 1: Dependencies
FROM node:20-slim AS deps
WORKDIR /app

# Install dependencies needed for native modules (bcrypt) and Prisma (openssl)
RUN apt-get update && apt-get install -y python3 make g++ openssl && \
    rm -rf /var/lib/apt/lists/*

COPY package*.json ./
COPY prisma ./prisma/

RUN npm ci
RUN npx prisma generate

# Stage 2: Builder
FROM node:20-slim AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/package*.json ./

COPY tsconfig.json ./
COPY src ./src
COPY prisma ./prisma

RUN npm run build

# Stage 3: Production
FROM node:20-slim AS production
WORKDIR /app

RUN apt-get update && apt-get install -y python3 make g++ openssl wget && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user for security
RUN groupadd -g 1001 nodejs && \
    useradd -u 1001 -g nodejs -m appuser

COPY package*.json ./
COPY prisma ./prisma/

RUN npm ci --only=production && \
    npx prisma generate && \
    apt-get purge -y python3 make g++ && \
    apt-get autoremove -y && \
    rm -rf /root/.npm /tmp/*

COPY --from=builder /app/dist ./dist
COPY entrypoint.sh ./entrypoint.sh

RUN chown -R appuser:nodejs /app

USER appuser

ENV NODE_ENV=production
ENV PORT=8080

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

CMD ["./entrypoint.sh"]
```

### Entrypoint Script

```bash
#!/bin/sh
# engine/entrypoint.sh
set -e

echo "Running database migrations..."
npx prisma migrate deploy 2>&1 || {
  echo "WARNING: Prisma migrate deploy failed -- starting server anyway"
  echo "The server may fail if schema is out of sync"
}

echo "Starting server..."
exec node dist/index.js
```

---

## NEVER Use `--set-env-vars` with `gcloud run services update`

This is the single most dangerous GCP gotcha. `--set-env-vars` REPLACES all env vars. Use `--update-env-vars` to add/change without destroying existing ones.

```bash
# WRONG: This destroys all existing env vars
gcloud run services update myapp-engine --set-env-vars KEY=value

# RIGHT: This adds/updates just this one var
gcloud run services update myapp-engine --update-env-vars KEY=value
```

In `cloudbuild.yaml`, `--set-env-vars` is acceptable because it's a full deploy that sets ALL vars every time.

---

## Staging vs Production

Staging uses a **separate database** on the same Cloud SQL instance:

```
Production: myapp (database) -> myapp-engine (Cloud Run service)
Staging:    myapp_staging (database) -> myapp-engine-staging (Cloud Run service)
```

Key differences in `cloudbuild-staging.yaml`:
- `DATABASE_URL=DATABASE_URL_STAGING:latest` (separate secret)
- `STRIPE_SECRET_KEY=STRIPE_SECRET_KEY_STAGING:latest` (test mode keys)
- `--memory 256Mi` (smaller for staging)
- `--max-instances 3` (fewer instances)
- Different `_APP_URL`, `_CORS_ORIGINS` (localhost allowed for staging)
- Different Stripe Product/Price IDs (test mode)

---

## Cloud Scheduler Jobs

Cloud Scheduler jobs call internal endpoints with OIDC authentication:

```bash
# Per-schedule task triggers (created dynamically when schedules are created)
gcloud scheduler jobs create http task-schedule-{SCHEDULE_ID} \
  --schedule="0 10 * * 1,3,5" \
  --uri="https://myapp-engine.run.app/api/internal/trigger-task" \
  --http-method=POST \
  --headers="Content-Type=application/json" \
  --message-body='{"scheduleId":"SCHEDULE_ID","entityId":"ENTITY_ID"}' \
  --oidc-service-account-email=myapp-backend@my-gcp-project.iam.gserviceaccount.com \
  --oidc-token-audience=https://myapp-engine-XXXXXXXXX.us-central1.run.app \
  --time-zone=America/New_York

# Recurring maintenance jobs
gcloud scheduler jobs create http trial-expiring-check \
  --schedule="0 9 * * *" \
  --uri="https://myapp-engine.run.app/api/internal/trial-expiring-check" \
  --http-method=POST \
  --oidc-service-account-email=myapp-backend@my-gcp-project.iam.gserviceaccount.com \
  --oidc-token-audience=https://myapp-engine-XXXXXXXXX.us-central1.run.app

gcloud scheduler jobs create http data-cleanup \
  --schedule="0 3 * * 0" \
  --uri="https://myapp-engine.run.app/api/internal/data-cleanup/data-purge" \
  --http-method=POST \
  --oidc-service-account-email=myapp-backend@my-gcp-project.iam.gserviceaccount.com \
  --oidc-token-audience=https://myapp-engine-XXXXXXXXX.us-central1.run.app
```

---

## Health Check Endpoint

```typescript
// engine/src/app.ts
app.get('/health', (_req: Request, res: Response) => {
  res.json({
    status: 'ok',
    version: '1.0.0',
    commit: env.COMMIT_SHA,
    timestamp: new Date().toISOString(),
    environment: env.NODE_ENV,
  });
});
```

---

## Express App Setup

```typescript
export function createApp(): Express {
  const app = express();

  // Trust one proxy level (Cloud Run load balancer)
  if (env.NODE_ENV === 'production') {
    app.set('trust proxy', 1);
  }

  // Security headers
  app.use(helmet({
    contentSecurityPolicy: { directives: { /* ... */ } },
    hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  }));

  // Unique request ID for audit trail
  app.use((req, res, next) => {
    const requestId = (req.headers['x-request-id'] as string) || crypto.randomUUID();
    req.headers['x-request-id'] = requestId;
    res.setHeader('x-request-id', requestId);
    next();
  });

  // CORS with dynamic origin checking
  app.use(cors({
    origin: (origin, callback) => {
      if (!origin) return callback(null, env.NODE_ENV === 'development');
      if (env.NODE_ENV === 'development' && origin.match(/^http:\/\/localhost(:\d+)?$/)) {
        return callback(null, true);
      }
      if (corsOrigins.includes(origin)) return callback(null, true);
      callback(new Error('Not allowed by CORS'));
    },
    credentials: true,
    maxAge: 86400,
  }));

  // Request parsing -- preserve raw body for webhook signatures
  app.use(express.json({
    limit: '10mb',
    verify: (req: any, _res, buf) => { req.rawBody = buf.toString(); },
  }));

  // Rate limiting
  app.use('/api', apiRateLimiter);

  // Audit logging
  app.use('/api', auditMiddleware);

  // Routes
  app.use('/api', router);

  // Error handler (must be last)
  app.use(errorHandler);

  return app;
}
```

---

## Firebase Hosting Multi-Site Config

```json
// firebase.json
{
  "hosting": [
    {
      "target": "website",
      "public": "website",
      "ignore": ["firebase.json", "**/.*"],
      "headers": [{ "source": "**", "headers": [{"key": "Cache-Control", "value": "max-age=3600"}] }]
    },
    {
      "target": "app",
      "public": "app-client/build/web",
      "ignore": ["firebase.json", "**/.*"],
      "rewrites": [{ "source": "**", "destination": "/index.html" }]
    }
  ]
}
```

Deploy:
```bash
cd website && firebase deploy --only hosting:website
cd app-client && flutter build web --release --dart-define=API_BASE_URL=https://myapp-engine-XXXXXXXXX.us-central1.run.app/api && firebase deploy --only hosting:app
```

---

## Gotchas

1. **Cloud Run cold starts:** `--cpu-boost` and `--min-instances=0` trade cost for speed. For production with consistent traffic, consider `--min-instances=1`.

2. **Database connections:** Set `connection_limit=3` in `DATABASE_URL` for Cloud Run (each instance gets a pool). With `max-instances=5`, that's 15 connections max against Cloud SQL's default 50.

3. **CORS with commas:** `CORS_ORIGINS` is comma-separated, but Cloud Build's `--set-env-vars` also uses commas. Use `^##^` as an alternative separator.

4. **Cloud Run deploy creates a new revision** but may not route traffic if the container fails the health check. Use `gcloud run services update-traffic --to-latest` to manually route.

5. **Dockerfile should NOT run `prisma migrate deploy`** -- a failed migration would prevent the container from starting, and you can't easily rollback.

---

## Checklist

- [ ] Create Cloud SQL instance and database
- [ ] Store secrets in Secret Manager
- [ ] Create service account with minimal permissions
- [ ] Write `cloudbuild.yaml` with build, push, deploy, health check steps
- [ ] Set up staging environment (separate database, separate Cloud Run service)
- [ ] Create Cloud Scheduler jobs for recurring tasks with OIDC auth
- [ ] Implement OIDC auth middleware for internal routes
- [ ] Implement health check endpoint
- [ ] Configure CORS with comma-aware separator in cloudbuild.yaml
- [ ] Set `trust proxy` for production
- [ ] Add `--cpu-boost` for faster cold starts
- [ ] Create multi-stage Dockerfile with non-root user
- [ ] Set up Firebase Hosting multi-site for website + app

# Full-Stack SaaS Implementation Playbook

A battle-tested, AI-friendly playbook for building production-grade SaaS applications. Contains complete, working code patterns for authentication, payments, notifications, infrastructure, and more.

## Purpose

This playbook is designed to **accelerate new project creation when working with AI coding assistants**. Each module is a standalone deep-dive with complete, working code that an AI agent can read and reproduce in a new project without additional context.

**Built for:** Firebase Auth + Express/Node.js + Prisma/PostgreSQL + Stripe + GCP Cloud Run + Flutter

## Quick Start

1. Read [`implementation-playbook.md`](implementation-playbook.md) for the full overview and quick-reference flows
2. Pick the modules relevant to your project
3. Feed the relevant module(s) to your AI assistant as context when building features
4. Adapt the code patterns to your domain models and business logic

## Modules

| # | Module | What It Covers |
|---|--------|----------------|
| 01 | [Authentication](playbook/01-authentication.md) | Firebase Auth, deferred registration, dual-mode token verification, Google Sign-In, TOTP MFA, router guards, session timeout, token revocation |
| 02 | [Email & Notifications](playbook/02-email-notifications.md) | Gmail API via domain-wide delegation, FCM push, in-app notifications, HMAC unsubscribe tokens, notification preferences |
| 03 | [Security](playbook/03-security.md) | Structured logging with PII redaction, generic error responses, Zod validation, rate limiting, audit logging, IDOR prevention |
| 04 | [Payments (Stripe)](playbook/04-payments-stripe.md) | Checkout sessions, Customer Portal plan changes, webhook handling with idempotency, graceful degradation, DB/Stripe sync recovery |
| 05 | [Cloud Infrastructure](playbook/05-cloud-infrastructure.md) | Cloud Run deployment, Cloud Build CI/CD, multi-stage Dockerfile, Cloud Scheduler with OIDC, staging/production environments |
| 06 | [Environment & Config](playbook/06-environment-config.md) | Zod-based env validation, Secret Manager integration, graceful degradation for optional services, frontend compile-time config |
| 07 | [Frontend Patterns (Flutter)](playbook/07-frontend-patterns.md) | Riverpod state management, Dio API client with auth interceptor, Go Router guards, design system, secure storage |
| 08 | [Database (Prisma)](playbook/08-database-prisma.md) | Schema design with soft deletes, CUID IDs, RBAC setup, migration workflow, connection management, seeding |
| 09 | [Testing](playbook/09-testing.md) | Jest + Prisma mocks, Supertest integration tests, Flutter Riverpod test overrides, mock factories, security regression tests |

## How to Use with AI Assistants

### Starting a new project
```
"Read the implementation playbook and set up the project structure
following the patterns in modules 01 (auth), 03 (security), 05 (infra),
06 (config), and 08 (database)."
```

### Adding a specific feature
```
"Read playbook module 04 (payments) and implement Stripe checkout
with webhook handling for our subscription plans."
```

### Reference during development
```
"Following the patterns in module 03 (security), add rate limiting
and audit logging to the new API endpoints."
```

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                  Flutter App                      │
│  (Riverpod + Go Router + Dio + Firebase Auth)     │
└──────────────────┬───────────────────────────────┘
                   │ HTTPS
┌──────────────────▼───────────────────────────────┐
│              Cloud Run (Express)                  │
│  Auth Middleware → Rate Limiter → Routes          │
│  Error Handler → Audit Logger                     │
├───────────┬──────────┬──────────┬────────────────┤
│  Prisma   │  Stripe  │  Gmail   │  FCM           │
│  (PgSQL)  │  API     │  API     │  (Push)        │
└───────────┴──────────┴──────────┴────────────────┘
```

## Key Design Principles

- **Graceful degradation** — Optional services (Stripe, Gmail, Sentry) are no-ops when unconfigured
- **Defense in depth** — PII redaction in logs, generic error responses, rate limiting, audit trails
- **Deferred registration** — Backend users created only after email verification (no orphan records)
- **Fire-and-forget notifications** — Notification failures never break business flows
- **Idempotent middleware** — Safe to apply at both router and route level
- **Environment parity** — Same code runs locally, staging, and production with config-driven behavior

## Customizing for Your Project

1. **Replace placeholder identifiers** — Search for `[YourApp]`, `myapp`, `my-gcp-project`, `example.com` and replace with your actual project values
2. **Adapt domain models** — The code examples use sample models (User, FamilyMember, etc.). Replace with your domain entities
3. **Pick your compliance level** — Security patterns are labeled with `SECURITY:` comments. Add or remove based on your requirements
4. **Choose your services** — Environment config supports optional services. Only configure what you need

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Flutter (Web + Mobile) |
| State Management | Riverpod (AsyncNotifier) |
| Routing | Go Router |
| Backend | Express.js (TypeScript) |
| Database | PostgreSQL + Prisma ORM |
| Auth | Firebase Auth |
| Payments | Stripe (Checkout + Customer Portal) |
| Email | Gmail API (Domain-Wide Delegation) |
| Push Notifications | Firebase Cloud Messaging |
| Hosting | GCP Cloud Run |
| CI/CD | Cloud Build |
| Secrets | GCP Secret Manager |
| Scheduled Jobs | Cloud Scheduler (OIDC) |
| Error Tracking | Sentry |
| Logging | Pino (GCP Cloud Logging compatible) |

## License

This project is licensed under the [MIT License](LICENSE).

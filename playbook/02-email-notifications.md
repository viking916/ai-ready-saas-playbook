# 02 - Email & Notifications

## Why

Gmail API via domain-wide delegation is more reliable than SMTP (no rate limits, no spam folder issues with Google Workspace domains) and cheaper than transactional email services. Domain-wide delegation lets a service account send email as any user in the Google Workspace domain.

Push notifications via FCM provide real-time alerts without polling. In-app notification persistence ensures users see notifications even if they missed the push.

**Key design decision:** All notification methods are **fire-and-forget** -- errors are logged but never thrown. A failed notification should never break a business flow.

---

## Gmail API Setup (Domain-Wide Delegation)

The tricky part: on Cloud Run, the metadata server cannot sign JWTs with a `sub` (subject) claim, which is required for domain-wide delegation. The solution is to use the IAM Credentials API to sign the JWT.

### GCP Setup Required

1. Enable the **Gmail API** and **IAM Credentials API** in GCP Console
2. Grant the service account the `iam.serviceAccounts.signJwt` permission (via `roles/iam.serviceAccountTokenCreator`)
3. In **Google Workspace Admin Console** > Security > API Controls > Domain-wide Delegation, authorize the service account with scope `https://www.googleapis.com/auth/gmail.send`

### The signJwt Workaround for Cloud Run

```typescript
// engine/src/services/notification.service.ts

import crypto from 'crypto';
import { google, gmail_v1 } from 'googleapis';
import { GoogleAuth } from 'google-auth-library';
import { getMessaging } from 'firebase-admin/messaging';
import { prisma } from '../index.js';
import { env } from '../config/env.js';
import { createLogger } from '../config/logger.js';

const log = createLogger('notification');

const SENDER_NAME = '[YourApp]';

/**
 * Escape HTML special characters to prevent injection in email templates.
 */
function escapeHtml(str: string): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');
}

class NotificationService {
  private senderAddress: string = '';
  private gmailEnabled: boolean = false;

  constructor() {
    if (env.GMAIL_SENDER_EMAIL) {
      this.senderAddress = env.GMAIL_SENDER_EMAIL;
      this.gmailEnabled = true;
      log.info('Gmail API email notifications enabled', { sender: env.GMAIL_SENDER_EMAIL });
    } else {
      log.info('GMAIL_SENDER_EMAIL not set -- email notifications disabled');
    }
  }

  /**
   * Get a Gmail client with domain-wide delegation.
   * On Cloud Run, the metadata server can't sign JWTs with a `sub` claim,
   * so we use the IAM Credentials API (signJwt) to create a delegated token.
   */
  private async getGmailClient(): Promise<gmail_v1.Gmail> {
    const impersonateAs = env.GMAIL_IMPERSONATE_EMAIL || env.GMAIL_SENDER_EMAIL!;

    // Get default credentials (metadata server on Cloud Run, ADC locally)
    const defaultAuth = new GoogleAuth();
    const defaultClient = await defaultAuth.getClient();

    // Get the service account email from the credentials
    const credentials = await defaultAuth.getCredentials();
    const serviceAccountEmail = credentials.client_email
      || 'myapp-backend@my-gcp-project.iam.gserviceaccount.com';

    // Create a JWT claim set with `sub` for domain-wide delegation
    const now = Math.floor(Date.now() / 1000);
    const jwtPayload = JSON.stringify({
      iss: serviceAccountEmail,
      sub: impersonateAs,
      scope: 'https://www.googleapis.com/auth/gmail.send',
      aud: 'https://oauth2.googleapis.com/token',
      iat: now,
      exp: now + 3600,
    });

    // Sign the JWT using IAM Credentials API
    const signRes = await defaultClient.request<{ signedJwt: string }>({
      url: `https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/${serviceAccountEmail}:signJwt`,
      method: 'POST',
      data: { payload: jwtPayload },
    });

    // Exchange signed JWT for a delegated access token
    const tokenRes = await fetch('https://oauth2.googleapis.com/token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: `grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&assertion=${signRes.data.signedJwt}`,
    });
    const tokenData = await tokenRes.json() as { access_token: string; error?: string };

    if (tokenData.error) {
      throw new Error(`Token exchange failed: ${tokenData.error}`);
    }

    // Create OAuth2 client with the delegated access token
    const oauth2Client = new google.auth.OAuth2();
    oauth2Client.setCredentials({ access_token: tokenData.access_token });

    return google.gmail({ version: 'v1', auth: oauth2Client });
  }
```

---

## Sending Email

The `sendEmail` method constructs raw RFC 2822 email with proper headers:

```typescript
  /**
   * Send an email via Gmail API (domain-wide delegation).
   * Gracefully degrades if Gmail is not configured.
   * When unsubscribeUrl is provided, List-Unsubscribe headers are added for native
   * email client unsubscribe buttons (e.g. Gmail).
   */
  async sendEmail(options: { to: string; subject: string; html: string; unsubscribeUrl?: string }): Promise<void> {
    if (!this.gmailEnabled) {
      log.debug('Email skipped (Gmail not configured)', { subject: options.subject });
      return;
    }

    try {
      const gmail = await this.getGmailClient();

      const headers = [
        `From: ${SENDER_NAME} <${this.senderAddress}>`,
        `To: ${options.to}`,
        `Subject: ${options.subject}`,
        'MIME-Version: 1.0',
        'Content-Type: text/html; charset=UTF-8',
      ];

      // RFC 8058: List-Unsubscribe headers for native email client unsubscribe buttons
      if (options.unsubscribeUrl) {
        headers.push(`List-Unsubscribe: <${options.unsubscribeUrl}>`);
        headers.push('List-Unsubscribe-Post: List-Unsubscribe=One-Click');
      }

      const rawEmail = [...headers, '', options.html].join('\r\n');
      const encodedMessage = Buffer.from(rawEmail).toString('base64url');

      await gmail.users.messages.send({
        userId: 'me',
        requestBody: { raw: encodedMessage },
      });
      log.info('Email sent', { subject: options.subject });
    } catch (error) {
      log.error('Failed to send email', error instanceof Error ? error : undefined);
      // Fire-and-forget: never throw
    }
  }
```

---

## One-Click Unsubscribe (HMAC Tokens)

Unsubscribe tokens use HMAC-SHA256 to prevent forgery without requiring database lookups:

```typescript
  /**
   * Get the secret used for HMAC-signing unsubscribe tokens.
   */
  private getUnsubscribeSecret(): string {
    return env.UNSUBSCRIBE_SECRET || env.JWT_SECRET || 'myapp-dev-unsubscribe-secret-do-not-use-in-prod';
  }

  /**
   * Generate an HMAC-signed unsubscribe token for a given userId.
   * Token format: base64url(userId):base64url(hmac)
   */
  generateUnsubscribeToken(userId: string): string {
    const secret = this.getUnsubscribeSecret();
    const userIdEncoded = Buffer.from(userId).toString('base64url');
    const hmac = crypto.createHmac('sha256', secret).update(userId).digest('base64url');
    return `${userIdEncoded}:${hmac}`;
  }

  /**
   * Verify an unsubscribe token and return the userId if valid, null otherwise.
   */
  verifyUnsubscribeToken(token: string): string | null {
    try {
      const parts = token.split(':');
      if (parts.length !== 2) return null;

      const [userIdEncoded, hmac] = parts;
      const userId = Buffer.from(userIdEncoded, 'base64url').toString('utf8');
      if (!userId) return null;

      const secret = this.getUnsubscribeSecret();
      const expectedHmac = crypto.createHmac('sha256', secret).update(userId).digest('base64url');

      // Use timingSafeEqual to prevent timing attacks
      if (hmac.length !== expectedHmac.length) return null;
      const isValid = crypto.timingSafeEqual(Buffer.from(hmac), Buffer.from(expectedHmac));

      return isValid ? userId : null;
    } catch {
      return null;
    }
  }

  /**
   * Build an unsubscribe URL for a given userId.
   */
  private buildUnsubscribeUrl(userId: string): string {
    const token = this.generateUnsubscribeToken(userId);
    return `${env.APP_URL}/api/users/unsubscribe?token=${encodeURIComponent(token)}`;
  }
```

---

## Email Footer Template

Consistent footer across all emails, with optional unsubscribe link:

```typescript
  /**
   * Consistent email footer with support contact for all templates.
   * When userId is provided, includes an unsubscribe link for non-essential emails.
   */
  private emailFooter(userId?: string): string {
    const unsubscribeHtml = userId
      ? `
      <p style="margin:8px 0 0;color:#999;font-size:12px;">
        <a href="${this.buildUnsubscribeUrl(userId)}" style="color:#999;text-decoration:underline;">Unsubscribe from non-essential emails</a>
      </p>`
      : '';

    return `
      <p style="margin:24px 0 0;color:#999;font-size:13px;">
        The [YourApp] Team
      </p>
      <p style="margin:8px 0 0;color:#999;font-size:12px;">
        Questions? Contact us at <a href="mailto:support@example.com" style="color:#6366F1;">support@example.com</a>
      </p>${unsubscribeHtml}
    `;
  }
```

---

## Push Notifications (FCM)

```typescript
  /**
   * Send a push notification via Firebase Cloud Messaging.
   * Gracefully degrades if the token is missing or FCM fails.
   */
  async sendPushNotification(options: {
    fcmToken: string;
    title: string;
    body: string;
    data?: Record<string, string>;
  }): Promise<void> {
    try {
      await getMessaging().send({
        token: options.fcmToken,
        notification: {
          title: options.title,
          body: options.body,
        },
        data: options.data,
      });
      log.info('Push notification sent', { title: options.title });
    } catch (error) {
      log.error('Failed to send push notification', error instanceof Error ? error : undefined);
    }
  }
```

Client-side FCM token registration is also fire-and-forget:

```dart
// After successful login:
PushNotificationService.instance.registerToken(ref.read(apiClientProvider));
```

---

## In-App Notification Persistence

```typescript
  /**
   * Persist an in-app notification record. Fire-and-forget -- never throws.
   */
  private async persistNotification(options: {
    userId: string;
    type: string;
    title: string;
    body: string;
    data?: Record<string, string | number>;
  }): Promise<void> {
    try {
      await prisma.notification.create({
        data: {
          userId: options.userId,
          type: options.type,
          title: options.title,
          body: options.body,
          data: options.data ? JSON.parse(JSON.stringify(options.data)) : undefined,
        },
      });
    } catch (error) {
      log.error('Failed to persist notification', error instanceof Error ? error : undefined);
    }
  }
```

---

## Full Email Template Example (Welcome Email)

```typescript
  async notifyFamilyWelcome(email: string, name: string): Promise<void> {
    try {
      await this.sendEmail({
        to: email,
        subject: 'Welcome to [YourApp]!',
        html: `
          <div style="font-family:sans-serif;max-width:560px;margin:0 auto;padding:24px;">
            <h2 style="color:#1a1a1a;margin:0 0 16px;">Welcome to [YourApp]</h2>
            <p style="margin:0 0 16px;color:#333;">
              Hi ${escapeHtml(name)},
            </p>
            <p style="margin:0 0 16px;color:#333;">
              Thank you for joining [YourApp]! We're excited to have you on board.
            </p>
            <p style="margin:0 0 16px;color:#333;">
              Here's what you can do:
            </p>
            <ul style="margin:0 0 16px;color:#333;line-height:1.7;">
              <li><strong>Set up your profile</strong> and configure your preferences</li>
              <li><strong>Explore the dashboard</strong> to get an overview of your activity</li>
              <li><strong>Customize notifications</strong> to stay informed on what matters</li>
              <li><strong>Get alerts</strong> for important events and updates</li>
            </ul>
            <p style="margin:0 0 16px;color:#333;">
              Get started by exploring the app.
            </p>
            <p style="margin:0 0 16px;">
              <a href="https://app.example.com" style="display:inline-block;background:#111827;color:#fff;padding:12px 24px;border-radius:8px;text-decoration:none;font-weight:600;">
                Open [YourApp]
              </a>
            </p>
            ${this.emailFooter()}
          </div>
        `,
      });
    } catch (error) {
      log.error('Failed to send family welcome email', error instanceof Error ? error : undefined);
    }
  }
```

---

## Notification Preference Flags and Gating Logic

Two independent flags control email and push notifications:

```typescript
  async notifyHighLoneliness(
    familyMemberUserId: string,
    elderlyUserName: string,
    score: number
  ): Promise<void> {
    try {
      const user = await prisma.user.findUnique({
        where: { id: familyMemberUserId },
        include: { familyMember: true },
      });

      if (!user) return;

      // Skip ALL notifications only when BOTH flags are false
      if (user.familyMember && !user.familyMember.receiveNotifications && !user.familyMember.receiveEmails) {
        log.debug('User has all notifications disabled, skipping');
        return;
      }

      // Send email if user hasn't disabled it
      if (user.familyMember?.receiveEmails !== false) {
        const unsubscribeUrl = this.buildUnsubscribeUrl(familyMemberUserId);
        await this.sendEmail({
          to: user.email,
          subject: `Wellbeing Alert: ${escapeHtml(elderlyUserName)}`,
          html: `...email HTML...`,
          unsubscribeUrl,
        });
      }

      // Send push if user hasn't disabled it and has FCM token
      if (user.familyMember?.receiveNotifications !== false && user.familyMember?.fcmToken) {
        await this.sendPushNotification({
          fcmToken: user.familyMember.fcmToken,
          title: 'Wellbeing Alert',
          body: `${elderlyUserName} may be feeling lonely.`,
          data: { type: 'high_loneliness', elderlyUserName },
        });
      }

      // Always persist in-app notification (visible in notification center)
      await this.persistNotification({
        userId: familyMemberUserId,
        type: 'high_loneliness',
        title: 'Wellbeing Alert',
        body: `${elderlyUserName} may be feeling lonely.`,
        data: { elderlyUserName, score },
      });
    } catch (error) {
      log.error('Failed to notify high loneliness', error instanceof Error ? error : undefined);
    }
  }
```

---

## Essential vs Non-Essential Notifications

| Category | Examples | Gated by preferences? |
|----------|----------|----------------------|
| Essential | Consent declined, admin alerts, welcome emails, invite | No -- always sent |
| Non-essential | Alerts (e.g., anomaly detection), action failed, trial expiring, schedule downgraded, delink | Yes -- both flags checked |

**6 user-facing methods gated:** `notifyAlert`, `notifyCallFailed`, `notifyTrialExpiring`, `notifySubscriptionLapsed`, `notifyScheduleDowngraded`, `notifyDelink`

**NOT gated (always sent):** `notifyConsentDeclined` (critical), `notifyFacilityWelcome` (facility), `notifyAdminFacilitySignup` (admin), `notifyFacilityTrialExpiring` (facility admin), `notifyInvite` (invite), `notifyWelcome` (new account)

---

## Scheduled Email Patterns

Cloud Scheduler triggers internal endpoints for recurring notifications:

```typescript
// engine/src/routes/internal.routes.ts

// All internal routes require OIDC auth (Cloud Scheduler tokens)
internalRouter.use(oidcAuth);

internalRouter.post('/trial-expiring-check', async (req, res) => {
  // Find users whose trial expires in 3 or 1 days
  const expiringUsers = await prisma.subscription.findMany({
    where: { tier: 'TRIAL', currentPeriodEnd: { gte: now, lte: threeDaysFromNow } },
  });
  // Send notification to each
  for (const sub of expiringUsers) {
    await notificationService.notifyTrialExpiring(sub.familyMember.userId, daysRemaining);
  }
  res.json({ processed: expiringUsers.length });
});
```

---

## Gotchas

1. **Cloud Run metadata server cannot sign JWTs with `sub` claim.** You must use the IAM Credentials API `signJwt` endpoint. This took significant debugging to discover.

2. **Gmail aliases:** If `GMAIL_SENDER_EMAIL` is an alias (e.g., `notifications@yourdomain.com`) rather than a primary Workspace user, you must set `GMAIL_IMPERSONATE_EMAIL` to the primary user account.

3. **`List-Unsubscribe-Post` header is required** by RFC 8058 for one-click unsubscribe in Gmail. Without it, Gmail won't show the native unsubscribe button.

4. **`timingSafeEqual` requires same-length buffers.** Check length before calling, or it throws.

5. **Always use `escapeHtml` on user-provided content** in email templates. A user's name could contain `<script>` tags.

---

## Checklist

- [ ] Enable Gmail API and IAM Credentials API in GCP
- [ ] Grant service account `roles/iam.serviceAccountTokenCreator`
- [ ] Set up domain-wide delegation in Google Workspace Admin Console
- [ ] Implement `sendEmail` with graceful degradation (no-op if not configured)
- [ ] Add `List-Unsubscribe` and `List-Unsubscribe-Post` headers
- [ ] Implement HMAC-based unsubscribe tokens with `timingSafeEqual`
- [ ] Add `receiveEmails` and `receiveNotifications` flags to user model
- [ ] Implement FCM token registration endpoint
- [ ] Create in-app notification persistence (database model)
- [ ] Categorize notifications as essential vs non-essential
- [ ] Set up Cloud Scheduler jobs for recurring notification tasks
- [ ] Always `escapeHtml()` user-provided content in email templates

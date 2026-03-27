# 04 - Payments (Stripe)

## Why

Stripe handles PCI compliance, tax calculation, and subscription lifecycle. The key design decision is **graceful degradation**: when `STRIPE_SECRET_KEY` is not set, the service reports `isConfigured() === false` and routes return 503. This lets you develop and test without Stripe credentials.

Plan changes use **Stripe Customer Portal** instead of a custom UI. This eliminates the need to build proration preview, tax calculation, and payment method update screens.

---

## Stripe Service (Complete)

```typescript
// engine/src/services/stripe.service.ts

import Stripe from 'stripe';
import { env } from '../config/env.js';
import { createLogger } from '../config/logger.js';

const log = createLogger('stripe');

export interface PlanDefinition {
  tier: 'STARTER' | 'ESSENTIALS' | 'PLUS' | 'FAMILY';
  name: string;
  monthlyPriceUsd: number;
  features: string[];
  maxEntities: number;
  actionsPerWeek: number;
  stripeProductId?: string;
  stripePriceId?: string;
}

export const FACILITY_PLAN = {
  name: 'Facility Seat',
  seatPriceUsd: 2500, // $25/seat/month in cents
  minSeats: 10,
  actionsPerWeekPerSeat: 3,
  stripeProductId: env.STRIPE_FACILITY_PRODUCT,
  stripePriceId: env.STRIPE_FACILITY_PRICE,
};

export const PLANS: PlanDefinition[] = [
  {
    tier: 'STARTER',
    name: 'Starter',
    monthlyPriceUsd: 1500,
    features: ['1 entity', '1 action per week', 'Activity summaries', 'Data archiving', 'Anomaly alerts', 'Team updates'],
    maxEntities: 1,
    actionsPerWeek: 1,
    stripeProductId: env.STRIPE_PRODUCT_STARTER,
    stripePriceId: env.STRIPE_PRICE_STARTER,
  },
  {
    tier: 'ESSENTIALS',
    name: 'Essentials',
    monthlyPriceUsd: 3000,
    features: ['1 entity', '3 actions per week', 'Activity summaries', 'Data archiving', 'Anomaly alerts', 'Team updates'],
    maxEntities: 1,
    actionsPerWeek: 3,
    stripeProductId: env.STRIPE_PRODUCT_ESSENTIALS,
    stripePriceId: env.STRIPE_PRICE_ESSENTIALS,
  },
  {
    tier: 'PLUS',
    name: 'Plus',
    monthlyPriceUsd: 5000,
    features: ['2 entities', '3 actions per week each', 'Activity summaries', 'Data archiving', 'Anomaly alerts', 'Team updates'],
    maxEntities: 2,
    actionsPerWeek: 3,
    stripeProductId: env.STRIPE_PRODUCT_PLUS,
    stripePriceId: env.STRIPE_PRICE_PLUS,
  },
  {
    tier: 'FAMILY',
    name: 'Family',
    monthlyPriceUsd: 7000,
    features: ['Up to 4 entities', '5 actions per week each', 'Activity summaries', 'Data archiving', 'Anomaly alerts', 'Team updates'],
    maxEntities: 4,
    actionsPerWeek: 5,
    stripeProductId: env.STRIPE_PRODUCT_FAMILY,
    stripePriceId: env.STRIPE_PRICE_FAMILY,
  },
];

class StripeService {
  private stripe: Stripe | null = null;

  constructor() {
    if (env.STRIPE_SECRET_KEY) {
      this.stripe = new Stripe(env.STRIPE_SECRET_KEY);
    }
  }

  isConfigured(): boolean {
    return !!this.stripe;
  }

  private getClient(): Stripe {
    if (!this.stripe) {
      throw new Error('Stripe is not configured. Set STRIPE_SECRET_KEY.');
    }
    return this.stripe;
  }

  async getOrCreateCustomer(userId: string, email: string, name?: string): Promise<string> {
    const stripe = this.getClient();
    const existing = await stripe.customers.search({
      query: `metadata['app_user_id']:'${userId}'`,
    });
    if (existing.data.length > 0) return existing.data[0].id;

    const customer = await stripe.customers.create({
      email,
      name: name || undefined,
      metadata: { app_user_id: userId },
    });
    log.info('Created Stripe customer', { customerId: customer.id, userId });
    return customer.id;
  }

  async createCheckoutSession(params: {
    customerId: string;
    tier: string;
    successUrl: string;
    cancelUrl: string;
    memberId: string;
  }): Promise<{ sessionId: string; url: string }> {
    const stripe = this.getClient();
    const plan = PLANS.find(p => p.tier === params.tier);
    if (!plan) throw new Error(`Unknown plan tier: ${params.tier}`);

    const lineItem = plan.stripePriceId
      ? { price: plan.stripePriceId, quantity: 1 }
      : {
          price_data: {
            currency: 'usd',
            ...(plan.stripeProductId
              ? { product: plan.stripeProductId }
              : { product_data: { name: `${plan.name} Plan`, description: plan.features.join(', ') } }
            ),
            unit_amount: plan.monthlyPriceUsd,
            recurring: { interval: 'month' as const },
          },
          quantity: 1,
        };

    const session = await stripe.checkout.sessions.create({
      customer: params.customerId,
      mode: 'subscription',
      payment_method_types: ['card'],
      line_items: [lineItem],
      success_url: params.successUrl,
      cancel_url: params.cancelUrl,
      automatic_tax: { enabled: true },
      customer_update: { address: 'auto' },
      metadata: {
        app_tier: params.tier,
        app_member_id: params.memberId,
      },
      subscription_data: {
        metadata: {
          app_tier: params.tier,
          app_member_id: params.memberId,
        },
      },
    });

    return { sessionId: session.id, url: session.url! };
  }

  async createPlanChangeSession(params: {
    customerId: string;
    stripeSubscriptionId: string;
    newTier: string;
    returnUrl: string;
  }): Promise<{ url: string }> {
    const stripe = this.getClient();
    const plan = PLANS.find(p => p.tier === params.newTier);
    if (!plan) throw new Error(`Unknown plan tier: ${params.newTier}`);
    if (!plan.stripePriceId) throw new Error(`Stripe Price ID not configured for ${plan.tier}`);

    const subscription = await stripe.subscriptions.retrieve(params.stripeSubscriptionId);
    const itemId = subscription.items.data[0]?.id;
    if (!itemId) throw new Error('No subscription item found');

    const session = await stripe.billingPortal.sessions.create({
      customer: params.customerId,
      return_url: params.returnUrl,
      ...(env.STRIPE_PORTAL_CONFIG_FAMILY ? { configuration: env.STRIPE_PORTAL_CONFIG_FAMILY } : {}),
      flow_data: {
        type: 'subscription_update_confirm',
        subscription_update_confirm: {
          subscription: params.stripeSubscriptionId,
          items: [{ id: itemId, price: plan.stripePriceId }],
        },
      },
    });

    return { url: session.url };
  }

  async getSubscriptionTier(stripeSubscriptionId: string): Promise<string | null> {
    const stripe = this.getClient();
    const subscription = await stripe.subscriptions.retrieve(stripeSubscriptionId);
    const productId = subscription.items?.data?.[0]?.price?.product;
    if (!productId) return null;
    const matchedPlan = PLANS.find(p => p.stripeProductId === String(productId));
    return matchedPlan?.tier ?? null;
  }

  async cancelSubscription(stripeSubscriptionId: string): Promise<void> {
    const stripe = this.getClient();
    await stripe.subscriptions.update(stripeSubscriptionId, { cancel_at_period_end: true });
  }

  async cancelSubscriptionImmediately(stripeSubscriptionId: string): Promise<void> {
    const stripe = this.getClient();
    await stripe.subscriptions.cancel(stripeSubscriptionId);
  }

  async reactivateSubscription(stripeSubscriptionId: string): Promise<void> {
    const stripe = this.getClient();
    await stripe.subscriptions.update(stripeSubscriptionId, { cancel_at_period_end: false });
  }

  constructEvent(rawBody: string, signature: string): Stripe.Event {
    const stripe = this.getClient();
    if (!env.STRIPE_WEBHOOK_SECRET) throw new Error('STRIPE_WEBHOOK_SECRET is not configured');
    return stripe.webhooks.constructEvent(rawBody, signature, env.STRIPE_WEBHOOK_SECRET);
  }

  // ============================================
  // FACILITY BILLING (B2B)
  // ============================================

  async createFacilityCheckoutSession(params: {
    customerId: string;
    facilityId: string;
    seats: number;
    successUrl: string;
    cancelUrl: string;
  }): Promise<{ sessionId: string; url: string }> {
    const stripe = this.getClient();

    const lineItem = FACILITY_PLAN.stripePriceId
      ? { price: FACILITY_PLAN.stripePriceId, quantity: params.seats }
      : {
          price_data: {
            currency: 'usd' as const,
            ...(FACILITY_PLAN.stripeProductId
              ? { product: FACILITY_PLAN.stripeProductId }
              : { product_data: { name: `${FACILITY_PLAN.name}` } }
            ),
            unit_amount: FACILITY_PLAN.seatPriceUsd,
            recurring: { interval: 'month' as const },
          },
          quantity: params.seats,
        };

    const session = await stripe.checkout.sessions.create({
      customer: params.customerId,
      mode: 'subscription',
      payment_method_types: ['card'],
      line_items: [{
        ...lineItem,
        adjustable_quantity: {
          enabled: true,
          minimum: FACILITY_PLAN.minSeats,
        },
      }],
      success_url: params.successUrl,
      cancel_url: params.cancelUrl,
      automatic_tax: { enabled: true },
      customer_update: { address: 'auto' },
      metadata: {
        app_type: 'facility',
        app_entity_id: params.facilityId,
        app_seats: String(params.seats),
      },
      subscription_data: {
        metadata: {
          app_type: 'facility',
          app_entity_id: params.facilityId,
          app_seats: String(params.seats),
        },
      },
    });

    return { sessionId: session.id, url: session.url! };
  }

  async updateSubscriptionQuantity(stripeSubscriptionId: string, newQuantity: number): Promise<void> {
    const stripe = this.getClient();
    const subscription = await stripe.subscriptions.retrieve(stripeSubscriptionId);
    const firstItem = subscription.items.data[0];
    if (!firstItem) throw new Error('No subscription item found');

    await stripe.subscriptionItems.update(firstItem.id, { quantity: newQuantity });
    await stripe.subscriptions.update(stripeSubscriptionId, {
      metadata: { app_seats: String(newQuantity) },
    });
  }
}

export const stripeService = new StripeService();
```

---

## Webhook Handler (Complete)

```typescript
// engine/src/routes/webhook.routes.ts

webhookRouter.post('/stripe', async (req: Request, res: Response, next: NextFunction) => {
  try {
    if (!stripeService.isConfigured()) {
      return res.status(503).json({ error: 'Stripe is not configured' });
    }

    const signature = req.headers['stripe-signature'];
    if (!signature || typeof signature !== 'string') {
      return res.status(400).json({ error: 'Missing Stripe signature' });
    }

    const rawBody = (req as any).rawBody;
    if (!rawBody) {
      return res.status(400).json({ error: 'Missing request body' });
    }

    let event;
    try {
      event = stripeService.constructEvent(rawBody, signature);
    } catch (err: any) {
      log.warn('Webhook signature verification failed', { error: err.message });
      return res.status(400).json({ error: 'Invalid signature' });
    }

    // Idempotency check: skip if this event was already processed
    const existingLog = await prisma.auditLog.findFirst({
      where: {
        action: 'WEBHOOK_PROCESSED',
        details: { path: ['stripeEventId'], equals: event.id },
      },
    });

    if (existingLog) {
      return res.json({ received: true });
    }

    switch (event.type) {
      case 'checkout.session.completed': {
        const session = event.data.object as any;
        if (session.metadata?.app_type === 'facility') {
          await handleFacilityCheckoutCompleted(session);
        } else {
          await handleCheckoutCompleted(session);
        }
        break;
      }
      case 'customer.subscription.updated': {
        const subscription = event.data.object as any;
        if (subscription.metadata?.app_type === 'facility') {
          await handleFacilitySubscriptionUpdated(subscription);
        } else {
          await handleSubscriptionUpdated(subscription);
        }
        break;
      }
      case 'customer.subscription.deleted': {
        const subscription = event.data.object as any;
        if (subscription.metadata?.app_type === 'facility') {
          await handleFacilitySubscriptionDeleted(subscription);
        } else {
          await handleSubscriptionDeleted(subscription);
        }
        break;
      }
      case 'invoice.payment_failed': {
        const invoice = event.data.object as any;
        if (invoice.subscription) {
          const sub = await stripeService.getSubscription(invoice.subscription);
          if (sub.metadata?.app_type === 'facility') {
            await handleFacilityPaymentFailed(invoice);
            break;
          }
        }
        await handlePaymentFailed(invoice);
        break;
      }
      default:
        log.info('Unhandled webhook event type', { type: event.type });
    }

    // Log successful processing for idempotency
    auditService.log({
      action: 'WEBHOOK_PROCESSED',
      resource: 'Subscription',
      details: { stripeEventId: event.id, type: event.type },
      outcome: 'SUCCESS' as any,
    });

    // Always return 200 to acknowledge receipt
    res.json({ received: true });
  } catch (error) {
    log.error('Webhook processing error', { error });
    // Return 500 so Stripe retries on transient failures
    res.status(500).json({ error: 'Processing error' });
  }
});
```

### Subscription Status Mapping

```typescript
function mapStripeStatus(stripeStatus: string): SubscriptionStatus {
  switch (stripeStatus) {
    case 'active':
    case 'trialing':
      return SubscriptionStatus.ACTIVE;
    case 'past_due':
      return SubscriptionStatus.PAST_DUE;
    case 'canceled':
    case 'unpaid':
      return SubscriptionStatus.CANCELLED;
    case 'paused':
      return SubscriptionStatus.PAUSED;
    case 'incomplete_expired':
      return SubscriptionStatus.EXPIRED;
    default:
      return SubscriptionStatus.ACTIVE;
  }
}
```

### Product ID to Tier Fallback

For portal-initiated plan changes, metadata may not be updated. Fall back to mapping the product ID:

```typescript
// In handleSubscriptionUpdated:
let newTier = subscription.metadata?.app_tier;

const productId = subscription.items?.data?.[0]?.price?.product;
if (productId) {
  const matchedPlan = PLANS.find(p => p.stripeProductId === String(productId));
  if (matchedPlan) {
    const productTier = matchedPlan.tier;
    if (!newTier || newTier !== productTier) {
      newTier = productTier;
      // Sync metadata so future webhook calls have the correct tier
      await stripeService.updateSubscriptionMetadata(subscription.id, { app_tier: newTier });
    }
  }
}
```

---

## DB/Stripe Sync Recovery

The `change-plan` endpoint detects when DB and Stripe are out of sync:

```typescript
// In POST /subscriptions/change-plan:
try {
  const { url } = await stripeService.createPlanChangeSession({ ... });
  return res.json({ url });
} catch (stripeError: any) {
  // Handle DB/Stripe tier mismatch: Stripe already has this plan
  if (stripeError?.type === 'StripeInvalidRequestError' &&
      stripeError?.message?.includes('no changes to confirm')) {
    // Sync DB tier from Stripe's actual product
    const actualTier = await stripeService.getSubscriptionTier(subscription.stripeSubscriptionId);
    if (actualTier) {
      await prisma.subscription.update({
        where: { id: subscription.id },
        data: { tier: actualTier as any },
      });
    }
    return res.status(409).json({ error: 'Your plan was already updated. Refreshing your subscription.' });
  }
  throw stripeError;
}
```

On the Flutter side, 409 triggers auto-refresh:

```dart
try {
  final response = await apiClient.post(ApiEndpoints.subscriptionChangePlan, data: {'tier': tier});
  await launchUrl(Uri.parse(response.data['url']));
} on DioException catch (e) {
  if (e.response?.statusCode == 409) {
    ref.invalidate(subscriptionPlansProvider); // Auto-refresh
  }
}
```

---

## Schedule Downgrade on Plan Change

When a plan is downgraded, schedules must be truncated to the new action limit:

```typescript
async function handleScheduleDowngrade(
  memberId: string,
  oldActionsPerWeek: number,
  newActionsPerWeek: number
): Promise<void> {
  const entities = await prisma.entity.findMany({
    where: { memberId, deletedAt: null },
    include: { schedules: true },
  });

  for (const entity of entities) {
    const schedule = entity.schedules[0];
    if (!schedule || schedule.daysOfWeek.length <= newActionsPerWeek) continue;

    // Keep the first N days (sorted ascending)
    const sortedDays = [...schedule.daysOfWeek].sort((a, b) => a - b);
    const keptDays = sortedDays.slice(0, newActionsPerWeek);

    await prisma.taskSchedule.update({
      where: { id: schedule.id },
      data: { daysOfWeek: keptDays },
    });

    // Update Cloud Scheduler cron expression
    await cloudSchedulerService.updateJob(schedule.id, entity.id, schedule.preferredTime, schedule.timezone, keptDays);

    // Notify member
    notificationService.notifyScheduleDowngraded(member.userId, entity.name, oldActionsPerWeek, newActionsPerWeek, keptDayNames);
  }
}
```

---

## Product/Price ID Management

Store Stripe IDs in `cloudbuild.yaml` substitutions, not manually-set env vars:

```yaml
substitutions:
  _STRIPE_PRODUCT_STARTER: 'prod_XXXXXXXXXXXXXXX'
  _STRIPE_PRODUCT_ESSENTIALS: 'prod_XXXXXXXXXXXXXXX'
  _STRIPE_PRICE_STARTER: 'price_XXXXXXXXXXXXXXXXXXXXXXXXX'
  _STRIPE_PRICE_ESSENTIALS: 'price_XXXXXXXXXXXXXXXXXXXXXXXXX'
```

**Why not just env vars?** Cloud Build `--set-env-vars` REPLACES all env vars on deploy. If you manually set an env var on Cloud Run, the next deploy wipes it.

---

## Gotchas

1. **`rawBody` must be preserved** before `express.json()` parses the request. Use the `verify` callback.

2. **Portal-initiated plan changes** don't update subscription metadata. Map Product ID to tier as fallback.

3. **DB/Stripe sync can diverge.** Catch "no changes to confirm" errors, sync DB from Stripe, return 409.

4. **`automatic_tax: { enabled: true }` requires** either `customer_update: { address: 'auto' }` or an existing customer address. Without an address, tax defaults to 0.

5. **Webhook signature verification requires the raw body as a string**, not the parsed JSON.

6. **Staging and production use separate Stripe API keys** (test mode vs live mode). Maintain separate `cloudbuild.yaml` files with different Product/Price IDs.

7. **Facility checkout uses `adjustable_quantity`** -- the webhook must read the actual seat count from the Stripe subscription item, not session metadata.

---

## Checklist

- [ ] Implement `isConfigured()` pattern for graceful degradation
- [ ] Create checkout sessions with `automatic_tax` and `customer_update`
- [ ] Set metadata on both session and `subscription_data`
- [ ] Use Customer Portal for plan changes (not custom UI)
- [ ] Implement webhook handler with idempotency (audit log check)
- [ ] Map Product IDs to tiers as webhook fallback
- [ ] Preserve `rawBody` in Express for signature verification
- [ ] Store Product/Price IDs in `cloudbuild.yaml` substitutions
- [ ] Set up separate Stripe test/live keys for staging/production
- [ ] Register tax IDs for applicable jurisdictions
- [ ] Implement DB/Stripe sync recovery (409 on "no changes" error)
- [ ] Handle schedule downgrades on plan changes

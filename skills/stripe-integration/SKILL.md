---
name: stripe-integration
description: Enforces Stripe integration best practices for webhook handling, idempotency, signature verification, and test vs live mode safety. Auto-invoke on any file touching Stripe webhooks, subscriptions, payments, checkout, or when user mentions Stripe, webhooks, webhook secret, idempotency, or subscription lifecycle.
model: sonnet
effort: medium
tools: Read, Edit, Grep, Bash
color: purple
---
# Stripe Integration

This skill enforces the non-negotiables for Stripe integration in Black Bear Studio projects (SkillsFrame, Plato, future billing).

## Webhook signature verification — REQUIRED

Every webhook handler MUST verify the Stripe signature before processing. Unverified webhooks are a critical security hole.

```typescript
import Stripe from 'stripe';
import { headers } from 'next/headers';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: Request) {
  const body = await req.text();
  const sig = headers().get('stripe-signature');

  if (!sig) return new Response('Missing signature', { status: 400 });

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(
      body,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    );
  } catch (err) {
    return new Response(`Webhook signature verification failed`, { status: 400 });
  }

  // Now event is verified — safe to process
}
```

NEVER parse the body with `req.json()` — you need the raw text for signature verification.

## Idempotency — REQUIRED for mutations

All mutation endpoints (create subscription, charge, refund) MUST use idempotency keys:

```typescript
await stripe.subscriptions.create(
  { customer: customerId, items: [{ price: priceId }] },
  { idempotencyKey: `sub-${userId}-${Date.now()}` }
);
```

Without idempotency keys, a retried request creates duplicate charges.

## Webhook event handling

Stripe delivers webhooks at-least-once. Your handler must be idempotent:
1. Check if the event has already been processed (store `event.id` in DB)
2. If processed, return 200 immediately — do not reprocess
3. If new, process and record `event.id` in the same transaction as the state change

## Test vs live mode

- Test mode keys start with `sk_test_` / `pk_test_`
- Live mode keys start with `sk_live_` / `pk_live_`
- NEVER commit live keys. Even once. Use `.env.local` only.
- Pre-launch check: confirm prod env has `sk_live_` and webhook endpoint is registered in live mode
- The `deploy-checklist` skill verifies this

## Subscription lifecycle events to handle

At minimum, handle:
- `checkout.session.completed` → grant access
- `customer.subscription.updated` → update plan/status
- `customer.subscription.deleted` → revoke access
- `invoice.payment_failed` → notify user, optionally downgrade
- `invoice.payment_succeeded` → confirm renewal

Do NOT rely on `checkout.session.completed` alone — users can downgrade, cancel, or fail payment later.

## Checklist for any Stripe PR

- [ ] Webhook signature verified before parsing body
- [ ] Raw body used (req.text, not req.json)
- [ ] Idempotency key on every mutation
- [ ] Event ID dedup in webhook handler
- [ ] No live keys in code, .env.example, or test fixtures
- [ ] All relevant subscription lifecycle events handled
- [ ] Error paths return 4xx / 5xx appropriately (Stripe will retry on 5xx)
- [ ] Webhook endpoint registered in Stripe dashboard for the target mode

Report any missing items before approving the change.

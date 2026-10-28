# Manage Subscription

Use when the agent must read the subscription state, change or cancel the plan, undo a pending change, or investigate open invoices and saved payment methods.

## Objective

Report the real plan and payment state, and change it only with explicit user approval, because plan changes can charge the account immediately.

## Preconditions

- A private key (`sk-aiv-acc`). Public keys cannot call these routes.
- The account is not `Custom`: its plan is managed by the AIVAX team and plan changes are rejected.
- Prices, fees, and renewal rules come from the live responses and from [Pricing](https://docs.aivax.net/docs/pricing) and [Plans and limits](https://docs.aivax.net/docs/limits), never from this skill.

## Decision Tree

1. Need the current state? `GET /api/v1/subscriptions`. Need only public list prices, without authentication? `GET /api/v1/information/plans.json` returns `plan` and `monthlyPrice` (USD) for `Free`, `Pro`, and `Max`; it excludes negotiated prices, so use `GET /api/v1/subscriptions` for the account's price.
2. Upgrade or first subscription? `POST /api/v1/subscriptions/plan` with `{"plan": "Pro" | "Max"}`. It is charged now with a pro-rata discount and applied only after payment is confirmed.
3. Downgrade? Same call; it is scheduled for the next renewal (`scheduled: true`) and nothing is charged now.
4. Stop renewing? `POST /api/v1/subscriptions/cancel`. The paid plan stays active until the end of the paid cycle, then the account returns to `Free`.
5. Changed your mind about a pending cancellation or downgrade? `POST /api/v1/subscriptions/resubscribe`.
6. Cards? `GET /api/v1/subscriptions/payment-methods` lists saved cards; `POST /api/v1/subscriptions/payment-methods/portal` returns a short-lived `url` for the user to add, remove, or default a card.
7. Open or paid invoices? Read `invoices` from `GET /api/v1/information/usage`.

## Reading The Responses

Success responses use the standard `{ message, data }` envelope.

- `GET /api/v1/subscriptions`: `plan`, `monthlyPrice`, `isActive` (current cycle is paid), `cycleEnd`, `pendingPlanModification`, `pendingPlan`, `isCancellationPending`, `planStartAt`, `pendingInvoiceId`, `pendingPaymentLocation`, `supportsSavedPaymentMethods`, and `plans[]` with `plan`, `monthlyPrice`, `dueToday`, `isDowngrade`. Use `plans[].dueToday` to show the user what the change costs before asking for approval.
- `POST .../plan`: `applied`, `scheduled`, `invoiceId`, `paymentLocation`. `applied: false` with a `paymentLocation` means the user must open that link to pay; the plan changes only after confirmation. `paymentLocation: null` means a saved card was charged or nothing was due.
- `GET .../payment-methods`: array of `id`, `brand`, `last4`, `expirationMonth`, `expirationYear`, `isDefault`.
- `invoices[]` in the usage response: `invoiceId`, `createdAt`, `paidAt`, `expiresAt`, `amount`, `description`, `isExpired`, `type` (`CreditAddition`, `SubscriptionPayment`, or `SubscriptionRenewal`), `paymentLocation` (only while open), `receiptLocation` (only when paid).

## Rules

- Do not call `POST .../plan`, `cancel`, or `resubscribe` without explicit approval; show the target plan, `dueToday`, and whether a saved card may be charged.
- Do not treat `Free` or `Custom` as cancellable (`400`).
- A pending payment already exists for the same plan: reuse `pendingPaymentLocation` instead of repeating the call.
- Plan-change and portal calls are rate-limited; on `429`, load `references/resilience/situations/rate-limit-429.md` and do not loop.
- Subscription invoices do not add prepaid balance; only `CreditAddition` invoices do. A subscription is separate from balance and from service allowances.
- Do not echo card details, payment links, or receipt links in final reports; give the invoice ID and state.
- Unpaid renewals are retried during a grace period and then the account returns to `Free`. A `SubscriptionRenewalFailureNotification` warns before that; see `references/notifications/`.

## Validation

- `GET /api/v1/subscriptions` shows the expected `plan` (immediate changes) or `pendingPlan` / `isCancellationPending` (scheduled changes).
- `GET /api/v1/information/balance` reports the same `plan`.
- The invoice for an upgrade moves from open to paid (`paidAt` set) before reporting success.

## Escalation

- Payment link expired or card failed: send the user to the account console; do not retry the charge in a loop.
- Custom terms, negotiated price, or billing start questions: the authorized administrator.
- Unexpected charge or balance drop: load `references/cost-monitoring/`.

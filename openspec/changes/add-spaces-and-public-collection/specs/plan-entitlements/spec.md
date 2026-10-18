## ADDED Requirements

### Requirement: Plan definitions
The system SHALL store the Free and Paid plans in an app-owned `plans` table with key, name, price, currency, billing interval, maximum spaces, maximum testimonials per space, Stripe price ID and active flag. Free SHALL allow 3 spaces and 100 testimonials per space; Paid SHALL allow 25 spaces and 1,000 testimonials per space. Seeding SHALL be idempotent.

#### Scenario: Plans are seeded
- **WHEN** the plan seeder runs on an empty database
- **THEN** a `free` plan (3 / 100) and a `paid` plan (25 / 1,000) exist

#### Scenario: Seeding twice does not duplicate
- **WHEN** the plan seeder runs twice
- **THEN** exactly two plan rows exist

#### Scenario: Paid Stripe price comes from environment
- **WHEN** the paid price ID env value is set and the seeder runs
- **THEN** the paid plan's `stripe_price_id` equals that value

### Requirement: Plan resolution from Cashier
The system SHALL determine a user's current plan from their existing Cashier subscription without altering Cashier's tables. A `default` subscription whose Stripe price matches the paid plan's `stripe_price_id` and that Cashier considers valid resolves to Paid, unless its status is excluded; every other case resolves to Free.

#### Scenario: User without subscription is Free
- **WHEN** a user has no subscription records
- **THEN** their plan resolves to Free

#### Scenario: Active paid subscription is Paid
- **WHEN** a user has a valid subscription for the paid price
- **THEN** their plan resolves to Paid

#### Scenario: Cancelled or expired subscription is Free
- **WHEN** a user's subscription has ended
- **THEN** their plan resolves to Free

### Requirement: Explicit subscription status check
The resolver SHALL check the subscription status explicitly: `incomplete`, `incomplete_expired`, `unpaid` and `paused` SHALL never resolve to Paid. The explicit check is required because Cashier's validity also passes through trial and grace-period dates that ignore the status.

#### Scenario: Excluded statuses are Free even with future dates
- **WHEN** a subscription has status `incomplete`, `incomplete_expired` or `unpaid` and a future `trial_ends_at` or `ends_at`
- **THEN** their plan resolves to Free

#### Scenario: Paused is Free
- **WHEN** a subscription for the paid price has status `paused`
- **THEN** their plan resolves to Free

### Requirement: Past-due subscriptions stay Paid
The system SHALL register `Cashier::keepPastDueSubscriptionsActive()` so that a `past_due` subscription remains Paid during Stripe's retries.

#### Scenario: past_due stays Paid
- **WHEN** a user's `default` subscription for the paid price has status `past_due`
- **THEN** their plan resolves to Paid

#### Scenario: Past-due setting is registered
- **WHEN** the application has booted
- **THEN** `Cashier::$deactivatePastDue` is false

### Requirement: Free resolution needs no subscription row
The system SHALL NOT require a Free subscription row. A user without a qualifying paid subscription SHALL resolve to the Free plan. A cancelled subscription SHALL remain Paid while its `ends_at` is in the future and SHALL resolve to Free afterwards. The Phase 5 `plan_id` and billing-period columns on `subscriptions`, webhook-driven activation and the payment-failed notice are out of scope for this change.

#### Scenario: Cancelled subscription stays Paid through its end date
- **WHEN** a user's paid subscription is cancelled with `ends_at` in the future
- **THEN** their plan resolves to Paid until `ends_at` passes and to Free afterwards

### Requirement: Subscription status matrix
The plan resolver SHALL map every subscription state to a plan as the scenarios in this capability describe: no row, `active`, `trialing`, `past_due`, a scheduled cancellation, `canceled`, `incomplete`, `incomplete_expired`, `unpaid`, `paused`, and a different price or type. Each state SHALL be covered by a unit test.

#### Scenario: Trial is Paid
- **WHEN** a subscription for the paid price has status `trialing` and a future `trial_ends_at`
- **THEN** their plan resolves to Paid

#### Scenario: Other price or type is Free
- **WHEN** the subscription price differs from the paid plan's `stripe_price_id`, that value is null, or the type is not `default`
- **THEN** their plan resolves to Free

### Requirement: Stripe webhook endpoint fails closed
The system SHALL NOT process a Stripe webhook request without a valid signature. Outside the `local` environment, when `STRIPE_WEBHOOK_SECRET` is not configured, Cashier's webhook endpoint SHALL reject every request with an error response, process nothing and log a configuration error without logging the payload. When a secret is configured, Cashier's signature verification SHALL remain in force.

#### Scenario: Unset secret outside local
- **WHEN** a webhook request arrives in a non-local environment with no webhook secret configured
- **THEN** it is rejected, no subscription or user record is created or changed, and a configuration error is logged without the payload

#### Scenario: Invalid signature
- **WHEN** a secret is configured and a request carries an invalid signature
- **THEN** it is rejected and nothing is processed

#### Scenario: Local development without a secret
- **WHEN** the environment is `local` and no secret is configured
- **THEN** the request reaches the handler; this is the only unsigned path

### Requirement: Space limit enforcement
The system SHALL enforce the plan's maximum number of spaces on the server when a space is created, inside a database transaction that locks the owner's row. A rejected creation SHALL persist nothing and SHALL show the owner a friendly message and an upgrade pointer; the UI SHALL also show usage and disable creation at the limit, but the server check is authoritative.

#### Scenario: Free user cannot create a fourth space
- **WHEN** a Free-plan user with 3 spaces submits a new space
- **THEN** no space is created and the response carries a plan-limit message

#### Scenario: Paid user can create up to 25
- **WHEN** a Paid-plan user with 24 spaces creates one more
- **THEN** the space is created
- **AND** a further creation is rejected

#### Scenario: Limit cannot be bypassed from the client
- **WHEN** a request is sent directly to the create endpoint while the UI is bypassed
- **THEN** the server still rejects it at the limit

### Requirement: Testimonial limit enforcement
The system SHALL enforce the owner's plan limit for testimonials per space when a submission is saved, counting non-deleted testimonials, inside a transaction that locks the space row. When a space is at its limit, the public page SHALL show a friendly "not accepting testimonials right now" message instead of the form, and a direct submission SHALL be rejected with that message and nothing stored.

#### Scenario: Space at limit rejects submissions
- **WHEN** a Free-plan owner's space already has 100 testimonials and a visitor submits another
- **THEN** nothing is stored and the visitor sees the friendly message

#### Scenario: Public page shows closed state
- **WHEN** a visitor opens the collection page of a space at its limit
- **THEN** the page shows the friendly message and no form

#### Scenario: Deleted testimonials free capacity
- **WHEN** a testimonial in a full space is soft-deleted
- **THEN** a new submission is accepted

### Requirement: Downgrade behavior
The system SHALL never delete existing data because of a plan change. When an account is over its limits after a downgrade, the system SHALL block new spaces and new submissions that would exceed the limit, while existing spaces, testimonials and management remain accessible.

#### Scenario: Downgraded owner keeps data but cannot add
- **WHEN** a former Paid user with 10 spaces resolves to Free
- **THEN** all 10 spaces remain viewable and editable
- **AND** creating an 11th space is rejected

### Requirement: Concurrency safety of limits
The system SHALL perform limit checks and inserts in a single transaction with a row lock on the parent record (user for spaces, space for testimonials) so concurrent requests cannot exceed a limit on databases that support row locking.

#### Scenario: Last slot is not double-booked
- **WHEN** two creations race for the last available slot on a locking-capable database
- **THEN** exactly one succeeds and the other is rejected

### Requirement: Stripe webhook event ledger
The system SHALL provide a `stripe_webhook_events` table with a unique `stripe_event_id`, event type, processed timestamp and created timestamp, so a future webhook handler can claim events, record them as processed only after successful handling, and skip duplicates. Webhook handling logic (signature verification, atomic claim, retries) is Phase 5 and out of scope for this change. Raw payloads are not stored and no retention job is included in the MVP.

#### Scenario: Duplicate event ID is rejected by the database
- **WHEN** two rows with the same `stripe_event_id` are inserted
- **THEN** the second insert fails on the unique constraint

### Requirement: Preserve existing authentication and Cashier schema
The change SHALL NOT modify, reorder or delete existing migrations, Fortify authentication code, or Cashier tables and columns.

#### Scenario: Existing migrations are unchanged
- **WHEN** the change is complete
- **THEN** the pre-existing migration files and Cashier tables are byte-for-byte unchanged and the existing authentication tests still pass

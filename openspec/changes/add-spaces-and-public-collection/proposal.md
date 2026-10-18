## Why

The app is currently the Laravel React starter kit with Fortify auth, passkeys, two-factor and Laravel Cashier installed. `User` is the only model, the dashboard is a placeholder, and there is no way to create a testimonial space or collect a testimonial. Phase 2 of `docs/PRD.md` ("Spaces and Collection") delivers the first end-to-end product loop: an owner creates and configures a space, shares a public link, and a logged-out customer submits a testimonial with consent. Phases 3 to 5 (inbox, dashboard, widgets, billing) all build on the data created here.

The PRD's acceptance criteria for spaces and submissions include server-side plan limits (3 spaces / 100 testimonials per space on Free, 25 / 1,000 on Paid). Those limits need a plan source of truth now, so this change also adds the small app-owned plan table and the webhook-idempotency ledger the PRD calls for. It does not change Cashier's tables.

The complete, agreed schema for the whole PRD is in `docs/data-model.md`. This change implements the Phase 2 subset of it and must stay consistent with it. Server-side plan limits are enforced in this change, intentionally earlier than PRD Phase 3, because the PRD's acceptance criteria for spaces and submissions already require them.

## What Changes

- Add six app-owned tables in six new migrations: `plans` (seeded), `stripe_webhook_events`, `spaces`, `space_questions`, `testimonials`, `testimonial_answers`. The five domain tables use UUID primary keys; `plans` and `stripe_webhook_events` use bigint. All indexes listed in `docs/data-model.md` for these tables are created with them. Cashier's migrations and the existing authentication code are left untouched.
- Add `Plan`, `StripeWebhookEvent`, `Space`, `SpaceQuestion`, `Testimonial` and `TestimonialAnswer` models and factories, plus `SpaceTheme` and `FormFieldMode` enums.
- Add an owner area under `/spaces` (list, create, creation-success page, show, edit, update, delete) behind the existing `auth` + `verified` middleware, protected by a `SpacePolicy` that returns 404 for spaces the user does not own.
- Support up to 10 optional text questions per space, each with its own required flag, and a `form_config` JSON field where company name, social profile URL and profile photo are each `off`, `optional` or `required`.
- Add a public area under `/collect/{slug}` (form, submit, confirmation) with no login. The form always has name, email and a required "Your testimonial" box, plus one answer box per custom question. Submissions are validated on the server, stored as a main text plus per-question answers with a prompt snapshot, record consent (the consent timestamp is set only when consent is given), and are protected by a submission token, a named rate limiter and a honeypot field.
- Add a plan layer: an idempotent `PlanSeeder` (Free and Paid), a `PlanResolver` that derives a user's plan from the existing Cashier subscription with an explicit status check, registration of `Cashier::keepPastDueSubscriptionsActive()` so `past_due` stays Paid, and server-side space and testimonial limit enforcement inside transactions.
- Add a fail-closed guard on Cashier's already-registered webhook route: outside the `local` environment, an unset `STRIPE_WEBHOOK_SECRET` makes it reject every request.
- Protect public submissions against stale forms: the form carries a derived version of its questions, and a submission made after a question was deleted or changed is rejected with a friendly message and stores nothing.
- Make account deletion clean up every space's photo files before the account is deleted.
- Add React/Inertia pages for the owner area and the public pages, a "Spaces" sidebar entry, and per-space public themes. No new npm or composer packages.
- Add PHPUnit unit and feature tests for every new behavior, including integrity tests for the unique-with-nulls answer constraint and for consent.
- Add new env keys to `.env.example` (testimonial media disk, paid plan Stripe price ID).

Out of scope for this change (later phases): inbox and testimonial management actions, inbox search, Wall of Love, dashboard analytics, the `embed_widgets` table and widgets (Phase 4), the billing page, Checkout, the webhook handler, the Phase 5 `subscriptions` columns, webhook event claiming and idempotency logic, the payment-failed notice, Stripe cancellation on account deletion, S3 adapter installation, moderation, email notifications.

## Capabilities

### New Capabilities

- `space-management`: creating, configuring, listing, viewing, editing and deleting spaces; custom questions with required flags, form-field configuration, rating toggle, themes, public slug, success page, ownership isolation, cascade on space and account deletion.
- `public-testimonial-collection`: the logged-out collection page, submission validation, main text plus per-question answers, consent recording, optional profile photo, duplicate and spam protection, confirmation page, answer integrity when questions change.
- `plan-entitlements`: Free and Paid plan definitions, plan resolution from Cashier, server-side space and testimonial limits, downgrade behavior, and the Stripe webhook event ledger table.

### Modified Capabilities

None. `openspec/specs/` is empty; existing authentication, settings and Cashier setup are not specified in OpenSpec and are preserved as-is.

## Impact

- **Database:** six new migrations dated after the existing Cashier migrations. No edits to existing migrations. The configured engine is MySQL and stays so; the schema uses only portable features and works on SQLite too, which the default test run uses (index names respect MySQL's 64-character limit; the null-unique behavior is documented for MySQL and was verified on SQLite and PostgreSQL).
- **Backend code:** new `app/Models`, `app/Enums`, `app/Policies`, `app/Actions`, `app/Services/Billing`, controllers under `app/Http/Controllers/Spaces` and `app/Http/Controllers/Collect`, Form Requests, shared validation concerns, a rate limiter registered in `AppServiceProvider`, and an exception for plan limits. `routes/web.php` gains the new routes.
- **Frontend:** new pages under `resources/js/pages/spaces/*` and `resources/js/pages/collect/*`, a question editor and public-form components, a `Textarea` UI component (no new package), one-line changes to the layout switch in `resources/js/app.tsx` and to the sidebar nav. Wayfinder TypeScript routes and actions are regenerated.
- **Config and ops:** new env keys; `composer setup` gains a plan-seeding step so fresh installs have plans; documented requirement to run `php artisan storage:link` when using the default `public` media disk.
- **Dependencies:** none added or upgraded. Production S3 storage will need `league/flysystem-aws-s3-v3` later; this is deferred and documented.
- **Time zone:** all timestamps and dashboard buckets are UTC. The `mysql` connection has no `timezone` setting; whether to configure it is flagged for implementation and needs approval (not changed here).
- **Risks:** public endpoints are new attack surface (spam, uploads, enumeration); concurrency guarantees depend on InnoDB row locks (real on MySQL, not provided by the SQLite test database). See `design.md`.

## Context

**Observed in the repository (read-only review):**

- Stack: Laravel 13.37, Inertia 3.5, React 19, Tailwind 4, shadcn-style components over Radix, Wayfinder-generated TypeScript routes and actions, MySQL (the project's `.env` sets `DB_CONNECTION=mysql`; `.env.example` still shows SQLite; session, cache and queue use the database driver; the default PHPUnit run uses SQLite in memory), PHPUnit 12, Pint (`laravel` preset), PHPStan level 7 via Larastan, CI runs `composer ci:check`.
- Auth: Fortify with passkeys and two-factor. `User` uses a bigint ID, `MustVerifyEmail`, and Cashier's `Billable`. Authenticated routes use `['auth', 'verified']` (dashboard) or `['auth']` (profile). The existing "delete account" action calls `$user->delete()`.
- Cashier schema already migrated: Stripe columns on `users`; `subscriptions` (`user_id`, `type`, `stripe_id`, `stripe_status`, `stripe_price`, `quantity`, `trial_ends_at`, `ends_at`); `subscription_items`. No plans table, no billing period columns, no webhook ledger.
- Conventions followed: controllers in `app/Http/Controllers/<Area>`, Form Requests in `app/Http/Requests/<Area>` with shared rules in `app/Concerns`, domain classes in `app/Actions`, feature tests as PHPUnit classes with `RefreshDatabase`, toasts via `Inertia::flash('toast', ...)`, rate limiters via `RateLimiter::for`, layout chosen by page-name prefix in `resources/js/app.tsx`.
- Gaps: only `User` exists as a model; dashboard and sidebar are starter placeholders; no S3 adapter; `.env.example` has no Stripe or media-disk keys; no `Textarea` UI component.
- `docs/PRD.md` suggests Next.js, Prisma and PostgreSQL "unless the project repository already establishes different conventions". The repository does, so the PRD is mapped onto Laravel.
- **`docs/data-model.md` is the authoritative schema** (agreed through a decision interview). This design implements its Phase 2 subset and must not contradict it. Where the two ever differ, `docs/data-model.md` wins and this file is updated.

## Goals / Non-Goals

**Goals:**

- Deliver PRD Phase 2 end to end: create and configure spaces with questions and form fields, share a public link, collect testimonials with per-question answers and consent, store them for later phases.
- Enforce space and testimonial plan limits on the server from day one.
- Add only the app-owned plan and webhook-ledger schema the PRD requires; leave Cashier and auth untouched.
- Target MySQL (the configured database) and keep every migration working on SQLite too, because the default test run uses SQLite in memory.
- Ship with automated tests that exercise ownership, validation, limits, answer integrity, consent and public-page privacy.
- Add no new dependencies and leave the database configuration (`.env`, `phpunit.xml`) untouched.

**Non-Goals:**

- Inbox, favourite, hide, edit and delete-testimonial actions, inbox search, Wall of Love management (Phase 3). The columns they need exist, and the inbox and widget indexes are created now.
- Dashboard cards and charts (Phase 3).
- `embed_widgets` and `embed.js` (Phase 4).
- Billing page, Checkout, webhook event handling (claim, idempotency, retries), the Phase 5 `subscriptions` columns, the payment-failed notice and account-deletion Stripe cancellation (Phase 5). The `past_due` setting and the webhook fail-closed guard are in this change (Decisions 7 and 18).
- S3 adapter and production storage (Phase 6).
- Email notifications, moderation, custom domains, return URLs.

## Decisions

### 1. Map the PRD onto Laravel conventions
Use Eloquent, Form Requests, policies, actions and Inertia pages rather than the PRD's Next.js/Prisma suggestion, as the PRD itself allows.

### 2. Keys
`spaces`, `space_questions`, `testimonials`, `testimonial_answers` (and later `embed_widgets`) use UUID primary keys via `HasUuids`. Time-ordered IDs suit InnoDB's clustered key and `HasUuids` is believed to produce them, but this is unverified against the installed Laravel 13.37: check the framework source first and pick the trait or generator that yields time-ordered IDs. `user_id` stays bigint. `plans` and `stripe_webhook_events` stay bigint (internal lookup and ledger, never exposed; documented in `docs/data-model.md`). Authenticated routes bind by UUID and always authorize ownership. Public pages use the slug; the one unavoidable exposure of an ID is the question UUID in the public form, needed to key answers.

### 3. Data model (Phase 2 subset; columns and indexes are specified in `docs/data-model.md`)
- `plans`: key (`free`/`paid`), limits, price, optional Stripe price ID; seeded idempotently.
- `stripe_webhook_events`: unique `stripe_event_id`, `event_type`, nullable `processed_at`, `created_at`. Table only; the handler is Phase 5.
- `spaces`: name, slug, title, subtitle, `form_config` JSON, `ratings_enabled` (default true), `theme`.
- `space_questions`: prompt, `is_required` (default false), `position`; zero to 10 per space.
- `testimonials`: customer details, `customer_email_normalized`, `testimonial_text` (required), `rating`, `social_sharing_consent`, **nullable** `consent_recorded_at`, `is_favourite`, `is_wall_of_love`, `is_visible` (default true), `submission_token`, `submitted_at`, soft deletes.
- `testimonial_answers`: `question_id` (nullable, set null on question delete), `question_prompt` snapshot, `answer_text`, `position`; unique `(testimonial_id, question_id)`.

*Why tables for questions and answers:* each answer is a real relationship to its question, which makes single-answer edits, answer search and a future "answers to question X" widget rule plain SQL, and the snapshot keeps history readable.
*Why a main text plus answers:* the PRD requires a testimonial text for submission, inbox search and editing, and public display; answers are additional, per-question detail.
*Why soft deletes on testimonials only:* PRD 8 and 13.5 call for immediate removal from public views and analytics with a recoverable row; spaces and users are hard-deleted with a full cascade.

### 4. Form configuration
`form_config` stores one state per optional field (`company_name`, `social_profile_url`, `profile_photo`): `off`, `optional` or `required`. Missing keys mean `off`. A `FormFieldMode` enum and a Form Request rule reject unknown keys and values. Name and email are always shown and required and are not stored. Turning a field off never deletes collected data; `required` applies to new submissions only.

### 5. Answer integrity and the unique-with-nulls constraint
The unique constraint on `(testimonial_id, question_id)` relies on null values being distinct. MySQL documents this behavior for UNIQUE indexes; SQLite and PostgreSQL were verified to behave the same (see `docs/data-model.md` section 7). It has not been run against a live MySQL during design, so the integrity test must be run on a scratch MySQL database before approval. Consequences: several orphaned answers per testimonial are allowed after questions are deleted; `NULLS NOT DISTINCT` must never be used; the application must never insert a null `question_id` (a test asserts it); foreign keys must be enforced (always on InnoDB; Laravel's default on SQLite), and the set-null test fails loudly if they are not.

### 6. Consent
`social_sharing_consent` defaults to false and is never required to be true. `consent_recorded_at` is set from the same clock reading as `submitted_at` only when consent is true, and is null otherwise. It is never accepted from the client and never editable by the owner. The invariant (timestamp present if and only if consent is true) is enforced in code and tested; no database CHECK is used so the migrations stay identical on MySQL and SQLite (MySQL before 8.0.16 ignores CHECK constraints).

### 7. Plans live in an app table; Cashier is not altered
Plan limits come from `plans`. A `PlanResolver` maps a user to a plan: a `default` subscription whose Stripe price equals the paid plan's `stripe_price_id`, whose status is not `incomplete`, `incomplete_expired`, `unpaid` or `paused`, and for which Cashier's `valid()` is true is Paid (the explicit status check is needed because `valid()` is `active() || onTrial() || onGracePeriod()` and the last two look only at dates); everything else is Free (Free users have no subscription row; a cancelled subscription stays Paid until `ends_at`). No plan foreign key or billing-period columns are added now (Phase 5, additively). `PlanSeeder` uses `updateOrCreate` on `key`, reads `STRIPE_PRICE_ID` for the paid plan, is called from `DatabaseSeeder`, and `composer setup` gains a seeding step. The resolver throws a clear exception if plans are missing. `Cashier::keepPastDueSubscriptionsActive()` is registered in `AppServiceProvider::boot` in this change, so `past_due` stays Paid during Stripe's retries; `incomplete`, `incomplete_expired` and `unpaid` stay Free. The complete status matrix is in `docs/data-model.md` section 6.5 and is a required unit test. `paused` is treated as Free (assumption, confirm before Phase 5).

### 8. Authorization: policy plus 404
`SpacePolicy` has `view`, `update` and `delete`, each returning `Response::denyAsNotFound()` for non-owners. `Route::resource('spaces', ...)` sits inside `['auth', 'verified']` with route-model binding by UUID. The space-limit check is a business rule, not authorization.

### 9. Routes

| Method | URI | Name | Access |
|---|---|---|---|
| GET | `/spaces` | `spaces.index` | owner |
| GET | `/spaces/create` | `spaces.create` | owner |
| POST | `/spaces` | `spaces.store` | owner |
| GET | `/spaces/{space}/created` | `spaces.created` | owner (success page) |
| GET | `/spaces/{space}` | `spaces.show` | owner |
| GET | `/spaces/{space}/edit` | `spaces.edit` | owner |
| PUT/PATCH | `/spaces/{space}` | `spaces.update` | owner |
| DELETE | `/spaces/{space}` | `spaces.destroy` | owner |
| GET | `/collect/{space:slug}` | `collect.show` | public |
| POST | `/collect/{space:slug}` | `collect.store` | public, `throttle:testimonial-submissions` |
| GET | `/collect/{space:slug}/thanks` | `collect.thanks` | public, requires session marker |

### 10. Question editing
`CreateSpace` and `UpdateSpace` sync the question list inside the same transaction as the space: existing questions keep their UUID when edited, new ones get a UUID and the next position, omitted ones are deleted (their answers' `question_id` becomes null by the foreign key), and positions are rewritten to match the submitted order. At most 10 questions are accepted; zero is valid; no default question is created.

### 11. Plan enforcement in actions, inside transactions
- `CreateSpace`: `DB::transaction` that locks the owner row (`lockForUpdate`), counts the owner's spaces, compares with the plan's `max_spaces`, and either creates the space or throws `PlanLimitExceeded('spaces')`.
- `SubmitTestimonial`: `DB::transaction` that locks the space row, re-checks the submission token, counts non-deleted testimonials against the owner's plan, then inserts the testimonial and its answers.
- Controllers catch `PlanLimitExceeded` and return a validation-style error (space create) or the closed state (public form) with a friendly message. Downgrades need no extra mechanism: checks are "count >= limit" and nothing is ever deleted.
- *Concurrency:* on MySQL (InnoDB) `lockForUpdate` is a real row lock, so the design works as intended. The lock is taken first in the transaction, before the count, and the transaction uses a small retry count for deadlocks. SQLite (default tests) ignores `lockForUpdate` but serializes writers, so tests cover the logic sequentially.

### 12. Public submission flow
1. `collect.show` loads the space by slug, builds a minimal prop set (title, subtitle, questions with ID, prompt and required flag, field states, ratings flag, theme, a new UUID submission token, `form_version`, `acceptingSubmissions`), and never includes owner data or the space UUID.
2. `SubmitTestimonialRequest` builds rules from the space: name, email (`email:rfc`, max 254) and `testimonial_text` (required, max 2,000) always; optional fields per `form_config` (URL restricted to `http`/`https`, photo rules); rating `integer|between:1,5` only when ratings are enabled; each custom question's answer (max 1,000, required when the question is required, whitespace-only counts as empty); `social_sharing_consent` boolean (absent means false); `submission_token` UUID; honeypot `website` must be empty.
3. Controller: a filled honeypot returns a fake success without writing. Order of checks: honeypot, duplicate-token no-op, form version (Decision 17), validation. Otherwise `SubmitTestimonial` runs: validate and store the photo (if any), then the transaction inserts the testimonial (email normalized by trim and lowercase; `consent_recorded_at` set only if consent is true) and one answer row per answered question (`question_id`, prompt snapshot, position). No row is created for unanswered optional questions. If the insert fails the stored file is removed.
4. Success sets a session marker and redirects to `collect.thanks`; direct visits without the marker redirect to the form. A repeated token is a successful no-op.
5. Rate limiter `testimonial-submissions` in `AppServiceProvider`: 5 per minute and 30 per hour keyed by IP plus slug.

Text is trimmed, stripped of control characters, stored as plain text and rendered only through React's escaping.

### 13. Deletion
Deleting a space removes photo files by application code first (failures are logged and never block), then deletes the space; foreign keys cascade to questions, testimonials (including soft-deleted) and answers. Deleting a user cascades the same way through the existing account-deletion action. The "cancel the Stripe subscription first" rule is recorded for Phase 5.

### 14. Profile photos without new packages
Photos use Laravel's `Storage` on the disk named by `TESTIMONIAL_MEDIA_DISK` (default `public`). Rules: `image`, `mimes:jpg,jpeg,png,webp` (SVG excluded), `max:2048` KB. Keys follow `testimonial-photos/{space_id}/{uuid}.{ext}` and are stored in `profile_photo_path`. Production S3 needs `league/flysystem-aws-s3-v3`, deliberately not installed now. The `public` disk requires `php artisan storage:link`.

### 15. Frontend
- Pages: `spaces/index`, `spaces/create`, `spaces/edit`, `spaces/show`, `spaces/created`, `collect/show`, `collect/thanks`.
- `SpaceForm` (shared by create and edit): name, title, subtitle, a question list editor (add, edit, remove, reorder, per-question required checkbox, 10-question cap), a three-way selector (`off` / `optional` / `required`) for each optional field, a ratings checkbox, and a theme select. A plain `Textarea` UI component is added; destructive confirmation uses the existing `Dialog`.
- Public form: name, email, enabled optional fields, one answer box per custom question (required markers per question), the required "Your testimonial" box with a fixed label, a rating input when enabled, an unchecked consent checkbox, a hidden honeypot and submission token. The submit button disables while processing.
- `app.tsx` layout switch: `collect/*` renders without the authenticated layout. Sidebar gets one "Spaces" item.
- Public themes are scoped CSS variable classes in `resources/js/lib/space-themes.ts`; body text must meet WCAG AA contrast (4.5:1), checked manually.
- Wayfinder routes and actions are regenerated after adding routes.

### 16. Testing
PHPUnit in the existing style. Unit: `PlanResolver` (including status transitions), plan seeder idempotency, form-config validation, slug generation, question sync, email normalization. Feature: space CRUD, ownership 404s, plan limits and downgrade, public page props and privacy, submission validation matrix (including per-question required answers and zero-question spaces), consent (timestamp set only when consent is true; client value ignored), photo handling (`Storage::fake`), answers and snapshots, duplicate token, honeypot, throttling, confirmation guard, space and user deletion cascades, and `TestimonialAnswerIntegrityTest` (set null on question delete with snapshots kept, two null rows per testimonial, duplicate non-null answer rejected, no null `question_id` inserted by submissions; also run on a scratch MySQL database, with a MySQL-only InnoDB and index assertion). Quality gates: Pint, PHPStan level 7, `npm run check`, `npm run types:check`, full suite.

### 17. Stale public forms
The public form carries `form_version`, a hash of the space's questions (`id`, `prompt`, `is_required`, `position`, in position order); it is computed, not stored. `SubmitTestimonial` recomputes it inside the transaction that locks the space row and rejects a mismatch with a validation error ("This form was updated while you were filling it in. Please check the questions below and submit again."), storing nothing and keeping the typed input. `UpdateSpace` locks the same space row, so a question cannot vanish between the check and the insert; a foreign-key violation (SQLSTATE 23000) during the answer insert is caught as a second line of defense, rolls back, removes any stored photo and returns the same validation error, never a 500. Answers are never silently accepted for deleted or changed questions, and answers for foreign question IDs are rejected.

### 18. Webhook route fails closed
Cashier registers `/stripe/webhook` by default and the application does not disable it; Cashier verifies the signature only when `cashier.webhook.secret` is set. A guard (middleware or controller subclass on that route, chosen at implementation) makes the route answer with an error, process nothing and log a configuration error without the payload when the secret is unset and the environment is not `local`. With a secret set, Cashier's signature check applies unchanged. The webhook handler's claim and idempotency logic stays in Phase 5.

### 19. Account deletion
The existing account-deletion action first deletes each of the user's spaces through `DeleteSpace` (photo cleanup, failures logged and non-blocking), then deletes the user; the foreign-key cascade removes what remains. The Phase 5 rule (Stripe cancellation must succeed first, otherwise deletion is blocked with an error) is recorded and not implemented here.

### 20. Time zone policy
All stored timestamps and all dashboard buckets and ranges are UTC (`config/app.php` already fixes `'timezone' => 'UTC'`). Day, ISO week (Monday) and month boundaries are UTC and come from one shared helper. The `mysql` connection has no `timezone` key; MySQL converts `TIMESTAMP` values through the session time zone. Implementation must either set the connection `timezone` to `+00:00` (a configuration change that needs approval, not made here) or bucket in PHP, and a scratch-MySQL test must prove a 23:30 UTC timestamp lands in the right UTC day under a non-UTC session.

## Risks / Trade-offs

- **Public spam and abuse** (new unauthenticated writes and uploads) → rate limiter, honeypot, strict validation, size and type limits, random file names. Captcha is not added.
- **Slug enumeration** → random suffix; spaces are public by design once shared.
- **MySQL specifics** → identifier names are limited to 64 characters (the widget-feed index is named explicitly), JSON columns take no literal default, the default collation is case-insensitive (the widget token is lowercase), and DDL is not transactional (one table per migration). A migration that passes on SQLite can still fail on MySQL, so the migrations are also run on a scratch MySQL database (task 8.8).
- **Concurrency on SQLite tests** → see Decision 11.
- **Null-distinct unique constraint** → relies on default behavior (documented for MySQL, verified on SQLite and PostgreSQL); guarded by tests and the "never `NULLS NOT DISTINCT`" rule.
- **Plan seeding forgotten in an environment** → resolver fails loudly; `composer setup` and the seeder cover the common paths.
- **Email verification gate** (`verified` on `/spaces`) with `MAIL_MAILER=log` means local developers read the log to verify; this matches the existing dashboard and is unchanged.
- **Live webhook route** → Cashier's route exists and verifies signatures only when a secret is set; the Phase 2 fail-closed guard (Decision 18) removes the unsigned path outside `local`.
- **Unverified on MySQL** → migrations, the integrity test, the time-zone behavior and `HasUuids` ordering have not been run; they stay marked unverified until task 8.8 runs.
- **Cascade deletes** → deleting a user deletes their spaces and testimonials, covered by a test. Cashier subscription cancellation on account deletion is a Phase 5 item and, until then, a known gap.
- **Soft-deleted testimonials keep customer emails** until their space is deleted; there is no purge in the MVP, documented as a retention limit.
- **Paid price semantics (999)** → stored as given; unit and currency are confirmed before Phase 5.

## Migration Plan

1. Add six migrations dated after `2026_10_18_051321`, in dependency order: `plans`, `stripe_webhook_events`, `spaces`, `space_questions`, `testimonials`, `testimonial_answers`. No existing table changes.
2. Run `php artisan db:seed --class=PlanSeeder` (also added to `composer setup`).
3. Add `TESTIMONIAL_MEDIA_DISK` and `STRIPE_PRICE_ID` to `.env.example`; run `php artisan storage:link` where the `public` disk is used.
4. Regenerate Wayfinder output and rebuild assets.
5. Rollback: each migration's `down()` drops only its own table; Cashier and auth tables are unaffected.

## Deviations from the PRD

The full list is in `docs/data-model.md` section 10. The ones that affect this change: normalized `space_questions` and `testimonial_answers` instead of JSON (allowed by the PRD); `form_config` is JSON with a single state per field; `consent_recorded_at` is nullable and set only when consent is given; `profile_photo_path` (object key) replaces `profile_photo_url`; bigint keys on `plans` and `stripe_webhook_events`; supporting columns `customer_email_normalized`, `submission_token`, `space_questions.is_required` and `plans.key`.

## Open Questions

1. **MySQL connection time zone:** the `mysql` connection has no `timezone` setting. Decide, with your approval, between setting it to `+00:00` and doing all date bucketing in PHP (Decision 20).
2. **`paused` subscriptions:** treated as Free until confirmed.
3. **`HasUuids` ordering:** verify against the installed Laravel 13.37 (Decision 2); unverified.
4. **Paid price and currency:** 999 in which currency and unit; resolved before Phase 5.
5. **End-to-end test tooling:** how to satisfy PRD 17's end-to-end list without adding packages; decided before Phase 6.

All earlier open questions (per-question answers, ratings default, consent, question limit, text length, space deletion) are resolved in `docs/data-model.md`.

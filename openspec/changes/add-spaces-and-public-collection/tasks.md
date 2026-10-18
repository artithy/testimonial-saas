## 1. Schema, enums and models

Authoritative column and index definitions are in `docs/data-model.md` section 4 and 5.

- [ ] 1.1 Create migration `create_plans_table` (bigint pk, unique `key`, limits, price, nullable unique `stripe_price_id`), dated after `2026_10_18_051321`
- [ ] 1.2 Create migration `create_stripe_webhook_events_table` (bigint pk, unique `stripe_event_id`, `event_type`, nullable `processed_at`, `created_at`)
- [ ] 1.3 Create migration `create_spaces_table` (UUID pk, `user_id` bigint foreign key with cascade, unique `slug`, JSON `form_config`, `ratings_enabled` default true, `theme`, listed indexes)
- [ ] 1.4 Create migration `create_space_questions_table` (UUID pk, `space_id` cascade, `prompt`, `is_required` default false, `position`, listed indexes)
- [ ] 1.5 Create migration `create_testimonials_table` (UUID pk, `space_id` cascade, soft deletes, `customer_email_normalized`, `testimonial_text`, `rating`, `social_sharing_consent` default false, nullable `consent_recorded_at`, `is_visible` default true, `profile_photo_path`, `submission_token`, `submitted_at`, listed indexes including unique `(space_id, submission_token)`; give the `(space_id, is_wall_of_love, is_visible, submitted_at)` index the explicit name `testimonials_widget_feed_index`, because the generated name exceeds MySQL's 64-character limit)
- [ ] 1.6 Create migration `create_testimonial_answers_table` (UUID pk, `testimonial_id` cascade, nullable `question_id` UUID foreign key with set-null on delete, `question_prompt` snapshot, `answer_text`, `position`, unique `(testimonial_id, question_id)`); do not use `NULLS NOT DISTINCT` or any engine-specific syntax; use `foreignUuid` for UUID foreign keys and `foreignId` for `user_id`; give `form_config` no database default Each migration has a `down()` that drops only its own table
- [ ] 1.7 Add enums `App\Enums\SpaceTheme` (`light`, `dark`, `minimal`) and `App\Enums\FormFieldMode` (`off`, `optional`, `required`)
- [ ] 1.8 Before relying on it, verify in the installed Laravel 13.37 source that `HasUuids` generates time-ordered UUIDs; if not, use the trait or generator that does
- [ ] 1.9 Add models `Plan`, `StripeWebhookEvent`, `Space` (`HasUuids`, casts, `user()`, `questions()`, `testimonials()`), `SpaceQuestion` (`HasUuids`, ordered by position), `Testimonial` (`HasUuids`, `SoftDeletes`, `CREATED_AT = 'submitted_at'`, `answers()`, casts), `TestimonialAnswer` (`HasUuids`, nullable `question()`); add `User::spaces()` without touching auth traits
- [ ] 1.10 Add factories for `Space`, `SpaceQuestion`, `Testimonial`, `TestimonialAnswer` (and `Plan` states `free` / `paid`)
- [ ] 1.11 Confirm existing Cashier and auth migration files are unmodified (`git diff` shows only additions under `database/migrations`)

## 2. Plan entitlements

- [ ] 2.1 Add `PlanSeeder` (idempotent `updateOrCreate` on `key`; Free 3/100, Paid 25/1000; paid `stripe_price_id` from `STRIPE_PRICE_ID`) and call it from `DatabaseSeeder`
- [ ] 2.2 Append `@php artisan db:seed --class=PlanSeeder --force` to the `setup` script in `composer.json`
- [ ] 2.3 Add `App\Services\Billing\PlanResolver`: Paid only when the subscription type is `default`, its `stripe_price` equals the paid plan's `stripe_price_id`, its status is not `incomplete`, `incomplete_expired`, `unpaid` or `paused`, and Cashier's `valid()` is true; otherwise Free; clear exception when plans are missing. The explicit status check is required because `valid()` is `active() || onTrial() || onGracePeriod()` and the last two ignore the status
- [ ] 2.4 Add `App\Exceptions\PlanLimitExceeded` carrying the resource (`spaces` or `testimonials`) and a friendly message
- [ ] 2.5 Add `.env.example` keys `STRIPE_PRICE_ID` and `TESTIMONIAL_MEDIA_DISK=public`
- [ ] 2.6 Register `Cashier::keepPastDueSubscriptionsActive()` in `AppServiceProvider::boot` so `past_due` stays Paid during Stripe's retries
- [ ] 2.7 Add the fail-closed webhook guard on Cashier's webhook route (middleware or controller subclass; document the choice): outside the `local` environment, when `STRIPE_WEBHOOK_SECRET` is unset, answer with an error (503), process nothing and log a configuration error without the payload; with a secret set, leave Cashier's signature verification untouched; never process an unsigned request outside `local`

## 3. Space management backend

- [ ] 3.1 Add `SpacePolicy` (`view`, `update`, `delete` using `Response::denyAsNotFound()`) and register it
- [ ] 3.2 Add shared rules concern `SpaceValidationRules` and Form Requests `StoreSpaceRequest` / `UpdateSpaceRequest` (name, title defaulting, subtitle, `form_config` with each of `company_name`, `social_profile_url`, `profile_photo` validated by `Rule::enum(FormFieldMode::class)` and unknown keys rejected, `questions` array of 0 to 10 items each with optional UUID `id`, `prompt` max 200 and boolean `is_required`, ratings boolean, `Rule::enum(SpaceTheme::class)`)
- [ ] 3.3 Add form-config normalization that treats missing keys as `off`
- [ ] 3.4 Add slug generator (slugified name + random suffix, retry on collision, never changed on update)
- [ ] 3.5 Add `CreateSpace` (transaction, owner-row lock, space-limit check, question creation; no default question) and `UpdateSpace` (locks the space row first so it serializes with submissions; question sync: keep IDs of existing questions, create new ones at the next position, delete omitted ones, rewrite positions to match the submitted order)
- [ ] 3.6 Add `DeleteSpace` action (remove photo files by application code first, log failures without blocking, then delete the space so foreign keys cascade to questions, testimonials including soft-deleted, and answers)
- [ ] 3.7 Add `SpaceController` (index, create, store, created, show, edit, update, destroy) returning Inertia pages; pass plan usage (`used`, `max`) to index and create
- [ ] 3.8 Register `Route::resource('spaces', ...)` plus `spaces.created` inside the `['auth', 'verified']` group in `routes/web.php`
- [ ] 3.9 Change the existing account-deletion action so it first calls `DeleteSpace` for each of the user's spaces (every photo file cleaned up, failures logged and non-blocking) and only then deletes the user; record, without implementing it, that in Phase 5 Stripe cancellation must succeed before deletion proceeds and that a failure blocks deletion with an error

## 4. Public collection backend

- [ ] 4.1 Add `CollectController` with `show`, `store`, `thanks`, using `{space:slug}` binding
- [ ] 4.2 Build the minimal public prop set (title, subtitle, questions with ID, prompt and required flag, field states, ratings flag, theme, submission token, `form_version`, `acceptingSubmissions`) with no space UUID, user ID or owner data
- [ ] 4.3 Add `SubmitTestimonialRequest` with rules derived from the space (name, email, required `testimonial_text` max 2,000; optional fields per `form_config`; URL scheme limited to http/https; rating 1-5 only when enabled; photo image rules; one answer per custom question, max 1,000, required when the question is required, whitespace-only counts as empty; consent boolean defaulting to false; honeypot `website`)
- [ ] 4.4 Add `SubmitTestimonial` action (token idempotency, owner plan limit under a space-row lock, plain-text sanitizing, email normalization by trim and lowercase, `consent_recorded_at` set from the same clock reading as `submitted_at` only when consent is true, one answer row per answered question with prompt snapshot and position and never a null `question_id`, no rows for unanswered optional questions, photo stored before insert and removed if the insert fails)
- [ ] 4.5 Register `testimonial-submissions` rate limiter (5/min and 30/hour per IP + slug) in `AppServiceProvider` and attach `throttle:testimonial-submissions` to the POST route
- [ ] 4.6 Implement confirmation guard: success sets a session marker; `thanks` without it redirects to the form
- [ ] 4.7 Handle `PlanLimitExceeded` on the public flow by returning the friendly closed state and storing nothing
- [ ] 4.8 Register public routes (`collect.show`, `collect.store`, `collect.thanks`) outside `auth` middleware
- [ ] 4.9 Add stale-form protection: compute `form_version` (hash of each question's `id`, `prompt`, `is_required`, `position` in position order; not stored); in `SubmitTestimonial` recompute it inside the space-row lock after the duplicate-token check and reject a mismatch with the friendly validation message, storing nothing and keeping the typed input; catch a foreign-key violation (SQLSTATE 23000) during the answer insert, roll back, remove any stored photo and return the same message instead of a 500; never silently accept answers to deleted or changed questions

## 5. Owner frontend (Inertia/React)

- [ ] 5.1 Add `resources/js/components/ui/textarea.tsx` (no new package)
- [ ] 5.2 Build `SpaceForm` component: name, title, subtitle, question list editor (add/edit/remove/reorder, per-question required checkbox, 10-question cap, zero questions allowed), a three-way `off` / `optional` / `required` selector for each optional field, ratings checkbox, theme `Select`
- [ ] 5.3 Build pages `spaces/index` (list, usage vs limit, empty state with create action, disabled create at limit with friendly message), `spaces/create`, `spaces/edit`, `spaces/show`
- [ ] 5.4 Build `spaces/created` success page (confirmation, public URL, copy-link, open page, back to dashboard)
- [ ] 5.5 Add delete-space confirmation using the existing `Dialog`, stating that testimonials and photos are permanently deleted
- [ ] 5.6 Add a "Spaces" item to the sidebar nav (`app-sidebar.tsx`) and breadcrumbs for the new pages
- [ ] 5.7 Regenerate Wayfinder routes/actions (`php artisan wayfinder:generate` or via the Vite plugin) and fix any type errors

## 6. Public frontend

- [ ] 6.1 Update the layout switch in `resources/js/app.tsx` so `collect/*` pages render without the authenticated layout
- [ ] 6.2 Add `resources/js/lib/space-themes.ts` with `light`, `dark` and `minimal` scoped theme classes and verify body text contrast is at least 4.5:1
- [ ] 6.3 Build `collect/show`: title, subtitle, name and email, enabled fields with required markers, one answer box per custom question with its own required marker, the required "Your testimonial" box, accessible star-rating input, consent checkbox (unchecked), hidden honeypot, hidden submission token, hidden `form_version`, validation errors including the stale-form message with typed values preserved, submit button disabled while processing
- [ ] 6.4 Build the closed state for spaces at their testimonial limit
- [ ] 6.5 Build `collect/thanks` with the friendly confirmation copy
- [ ] 6.6 Verify keyboard navigation, labels and visible focus states on the public form, and layout at mobile, tablet and desktop widths

## 7. Tests

- [ ] 7.1 Unit: `PlanResolver` including the complete subscription-status matrix (7.17) and missing plans, plan seeder idempotency, form-config validation (missing keys mean off, unknown keys and values rejected), slug uniqueness and stability, question sync (IDs kept, positions rewritten, omitted removed), email normalization
- [ ] 7.2 Feature: guests redirected from every `/spaces` route; unverified users blocked as on the dashboard
- [ ] 7.3 Feature: create/list/show/edit/update/delete for owners; title defaulting; zero questions allowed and no default question created; more than 10 questions rejected; invalid input errors; success page contents
- [ ] 7.4 Feature: ownership isolation returns 404 for show, edit, update and delete on another user's space
- [ ] 7.5 Feature: space limits (Free at 3, Paid at 25, deleting frees capacity, downgraded user keeps data but cannot add, direct POST rejected)
- [ ] 7.6 Feature: public page renders for guests; unknown slug 404; props contain no user ID, owner email, space UUID or testimonials; disabled fields absent; consent unchecked; space with no questions still renders name, email and testimonial fields
- [ ] 7.7 Feature: submission validation matrix (required fields including `testimonial_text`, field set to required, email, URL scheme, rating required/out of range/ignored when disabled, disabled fields ignored, required question left empty or whitespace-only rejected, optional question left empty accepted)
- [ ] 7.8 Feature: defaults (visible, not favourite, not Wall of Love, `submitted_at`); answers stored with `question_id`, prompt snapshot and position; no answer row for unanswered optional questions; snapshot survives prompt edits
- [ ] 7.9 Feature: profile photo accepted and stored (`Storage::fake`), PDF/SVG/oversized rejected with nothing stored
- [ ] 7.10 Feature: duplicate token creates one record; honeypot stores nothing but shows confirmation; rate limiter returns 429 after the threshold
- [ ] 7.11 Feature: confirmation shown only after persistence; direct visit redirects; failed validation never shows confirmation
- [ ] 7.12 Feature: testimonial limit (space at limit shows closed state and rejects POST; soft-deleted testimonial frees capacity)
- [ ] 7.13 Feature: HTML in submitted text is stored literally; space deletion cascades to questions, testimonials (including soft-deleted) and answers and removes photo files, and a file-removal failure is logged without blocking; user deletion cascades to spaces and everything beneath; `stripe_webhook_events` rejects duplicate `stripe_event_id`
- [ ] 7.14 Feature `tests/Feature/Testimonials/SubmitTestimonialConsentTest.php`: consent true sets `social_sharing_consent` and a `consent_recorded_at` equal to `submitted_at`; consent false or absent leaves `consent_recorded_at` null; a client-supplied `consent_recorded_at` is ignored
- [ ] 7.15 Feature `tests/Feature/Testimonials/TestimonialAnswerIntegrityTest.php`: deleting a question sets `question_id` to null and keeps the prompt snapshot and answer text; after deleting two questions the testimonial has two null-`question_id` rows without a unique violation; a duplicate answer for the same non-null question is rejected even after other questions were deleted; deleting a space cascades to answers; the submission action never inserts a null `question_id`; the class runs on whichever connection the suite uses; add a MySQL-only test (skipped elsewhere) asserting InnoDB tables and a unique `(testimonial_id, question_id)` index via `SHOW INDEX`, which refuses to run unless the database name ends in `_test`
- [ ] 7.16 Confirm the existing auth, settings and dashboard tests still pass unchanged
- [ ] 7.17 Unit, complete subscription-status matrix, one test per row of `docs/data-model.md` section 6.5: no subscription; `active`; `trialing` with future `trial_ends_at`; `past_due`; cancel scheduled with future `ends_at`; `canceled` with past `ends_at`; `incomplete`, `incomplete_expired` and `unpaid`, each with and without a future `trial_ends_at` or `ends_at`; `paused`; mismatching price; null paid `stripe_price_id`; non-`default` type; plus a test that `Cashier::$deactivatePastDue` is false after boot
- [ ] 7.18 Feature, stale forms: question deleted, prompt edited, required flag changed and questions reordered after the form was opened are each rejected with the friendly message, store nothing (no testimonial, answers or file) and keep the typed input; a forced foreign-key violation during the answer insert returns the same validation error, not a 500; an answer for another space's question id is rejected; a change to optional-field states alone is not rejected as stale; a duplicate token is a no-op before the version check
- [ ] 7.19 Feature, account deletion: deleting an account that owns spaces with stored photos (`Storage::fake`) removes every photo file and every row; a forced file-removal failure is logged and deletion still completes
- [ ] 7.20 Feature, webhook guard: outside `local` with `STRIPE_WEBHOOK_SECRET` unset, a request to the webhook route is rejected and creates or changes no subscription; with a secret set and an invalid signature it is rejected; only the `local` environment without a secret reaches the handler
- [ ] 7.21 MySQL scratch run, time zone: store a timestamp at 23:30 UTC under a non-UTC session time zone and assert it falls in the correct UTC day (see 8.10)

## 8. Quality gates and documentation

- [ ] 8.1 `composer lint:check` (Pint) passes
- [ ] 8.2 `composer types:check` (PHPStan level 7) passes with no baseline additions
- [ ] 8.3 `npm run check` and `npm run types:check` pass
- [ ] 8.4 `php artisan test` passes in full
- [ ] 8.5 Confirm `composer.json` dependencies and `package.json` dependencies are unchanged (only the `setup` script line differs)
- [ ] 8.6 Add short setup notes (UTC policy, `STRIPE_WEBHOOK_SECRET` must be set outside local, plans seeding, `storage:link`, new env keys, S3 adapter deferred, how to run the integrity test on a scratch MySQL database, foreign keys must stay enabled on SQLite, retention limit for soft-deleted testimonials) and record the deviations listed in `design.md`
- [ ] 8.7 Manual walkthrough: register, create a space with questions/fields/theme, open the public link in a logged-out session, submit with and without consent, then confirm the records in the database; try exceeding the Free space limit
- [ ] 8.8 On a scratch MySQL database whose name ends in `_test` (never the development database), run `php artisan migrate:fresh` and then `DB_CONNECTION=mysql DB_DATABASE=<scratch> php artisan test --filter=TestimonialAnswerIntegrityTest`; confirm no identifier-length or foreign-key-type errors and that the null-unique cases pass. Until this task has actually run and its result is recorded in the verification table in `docs/data-model.md` section 7, all MySQL behavior (migrations, null-unique constraint, engine, time zone, UUID ordering) is **unverified**, and Phase 2 must not be described as MySQL-verified
- [ ] 8.9 Confirm `.env`, `.env.example` database values and `phpunit.xml` are unchanged by this change
- [ ] 8.10 Time zone: on the scratch MySQL database, check how the `mysql` connection (which has no `timezone` setting) behaves when the server session time zone is not UTC; if the 7.21 test fails, propose the `timezone` => `+00:00` connection setting (or PHP-side bucketing) to the project owner and apply a configuration change only after approval
- [ ] 8.11 Record in the final documentation that plan-limit enforcement was intentionally moved from PRD Phase 3 to Phase 2, that name and email are always shown and required (not configurable), that `paused` subscriptions resolve to Free, and that the storage engine is the server default, expected InnoDB, verified by the MySQL-only test

# Data Model — Minimal Testimonial SaaS Platform

**Status:** Approved design, revised 2026-10-18 after review (decisions 18 to 22). Implementation has not started. MySQL behavior is unverified until the scratch-database run in section 7 is done.
**Date:** 2026-10-18
**Source requirements:** `docs/PRD.md` (sections 5, 6, 8, 9, 10, 11, 13, 14, 16, 17)
**Stack:** Laravel 13, Inertia 3, React 19, Laravel Cashier 16.9 for Stripe. **Database: MySQL** (the project's `.env` sets `DB_CONNECTION=mysql`; `.env.example` still shows SQLite). The default PHPUnit run uses SQLite in memory (`phpunit.xml`), so the schema must also work there. See section 7.
**How this was produced:** a one-question-at-a-time design interview; the decisions are recorded in section 11. Where this document and the PRD differ, section 10 explains why.

---

## 1. Scope and phasing

This document is the authoritative schema for the whole PRD. The migrations are delivered in phases:

| Phase | Tables and changes |
|---|---|
| Phase 2 (OpenSpec change `add-spaces-and-public-collection`) | `plans` (seeded), `stripe_webhook_events`, `spaces`, `space_questions`, `testimonials`, `testimonial_answers` |
| Phase 4 | `embed_widgets` |
| Phase 5 | Additive migration on Cashier's `subscriptions` table: nullable `plan_id`, `current_period_start`, `current_period_end` |

Phase 2 also delivers, without schema changes: server-side plan-limit enforcement (**intentionally moved forward from PRD Phase 3**, because the PRD's space and submission acceptance criteria already require it), the plan resolver with `Cashier::keepPastDueSubscriptionsActive()` registered, and a fail-closed guard on Cashier's webhook route (section 6.6).

Existing tables are never edited by migration: `users`, Fortify and passkey tables, and Cashier's migrations stay exactly as they are.

---

## 2. Sample dataset used in examples

Two users, each owning two spaces.

| user | name | plan |
|---|---|---|
| 1 | Amina Rahman | Free |
| 2 | Rafi Hasan | Paid |

| space | owner | public title | custom questions |
|---|---|---|---|
| Pixel Studio – Clients | Amina | Tell us about working with Pixel Studio | Q1 "What problem were we solving for you?" (required), Q2 "What was the result?" (required), Q3 "Anything you'd tell a future client?" (optional) |
| Studio Newsletter | Amina | What do you think of our newsletter? | none |
| Python Bootcamp | Rafi | How was the bootcamp? | Q1 "How did the bootcamp change your skills?" |
| Rafi Coaching | Rafi | Your coaching experience | Q1 "What did we work on?", Q2 "What changed for you?" |

Ratings are **enabled in every sample space except Studio Newsletter**, where they are disabled. That is why T4 and T5 have no rating; every testimonial in the other three spaces has one.

| # | space | customer | email as typed | rating | consent | `consent_recorded_at` | state |
|---|---|---|---|---|---|---|---|
| T1 | Pixel Studio | Tasnim Ahmed | `Tasnim@Cafe.example` | 5 | yes | = `submitted_at` | active, Wall of Love |
| T2 | Pixel Studio | Jon Lee | `jon@lee.example` | 4 | no | null | active |
| T3 | Pixel Studio | Mira Das | `mira@das.example` | 5 | yes | = `submitted_at` | hidden |
| T4 | Newsletter | Jonathan Lee | `Jon@lee.example ` | none | no | null | active |
| T5 | Newsletter | Sara K. | `sara@k.example` | none | yes | = `submitted_at` | soft-deleted |
| T6 | Bootcamp | Tasnim Ahmed | `tasnim@cafe.example` | 5 | yes | = `submitted_at` | active, Wall of Love |
| T7 | Bootcamp | Omar F. | `omar@f.example` | 3 | no | null | active |
| T8 | Coaching | Lina B. | `lina@b.example` | 5 | yes | = `submitted_at` | active |

Expected "unique customers": Amina 3 (Tasnim, Jon, Mira; Sara is deleted), Rafi 3 (Tasnim, Omar, Lina).

---

## 3. Relationships and cascades

```
users (bigint) 1──* spaces (uuid) 1──* space_questions (uuid)
                         │ 1──* testimonials (uuid) 1──* testimonial_answers (uuid) *──0..1 space_questions
                         └ 1──* embed_widgets (uuid)           (Phase 4)

plans (bigint) 1──* subscriptions.plan_id (Phase 5, Cashier-owned table)
stripe_webhook_events (bigint)   standalone ledger
```

| parent | child | when the parent is deleted |
|---|---|---|
| `users` | `spaces` | cascade (account deletion; see section 6.5 for the Stripe rule) |
| `spaces` | `space_questions`, `testimonials`, `embed_widgets` | cascade |
| `testimonials` | `testimonial_answers` | cascade |
| `space_questions` | `testimonial_answers.question_id` | **set null**; the prompt snapshot keeps the answer readable |
| `plans` | `subscriptions.plan_id` | restrict |

---

## 4. Tables

Types are given in Laravel schema-builder terms. All timestamps are UTC.

### 4.1 `plans` (bigint key)

| column | type | notes |
|---|---|---|
| id | bigint pk | |
| key | string(20) | unique: `free`, `paid` |
| name | string | |
| price | unsigned integer | 0 (Free) / 999 (Paid); unit and currency confirmed before Phase 5 |
| currency | string(3), null | |
| billing_interval | string(10) | `month` |
| max_spaces | unsigned smallint | Free 3, Paid 25 |
| max_testimonials_per_space | unsigned integer | Free 100, Paid 1,000 |
| stripe_price_id | string, null | unique; filled from environment when seeding |
| is_active | boolean | default true |
| created_at, updated_at | timestamps | |

Seeded idempotently (`updateOrCreate` on `key`).

### 4.2 `stripe_webhook_events` (bigint key)

| column | type | notes |
|---|---|---|
| id | bigint pk | |
| stripe_event_id | string | unique |
| event_type | string | |
| processed_at | timestamp, null | |
| created_at | timestamp | no `updated_at` |

Rules are in section 6.6.

### 4.3 `spaces`

| column | type | notes |
|---|---|---|
| id | uuid pk | |
| user_id | bigint FK → `users.id` | cascade on delete |
| name | string(100) | internal name |
| slug | string | unique; slugified name plus a random suffix; never changes after creation |
| title | string(120) | public page title; defaults to `name` when omitted |
| subtitle | string(240), null | |
| form_config | json | see below |
| ratings_enabled | boolean | default true |
| theme | string(20) | `light` (default), `dark`, `minimal`; validated against an enum |
| created_at, updated_at | timestamps | |

`form_config` holds one state per optional field: `off`, `optional` or `required`.

```json
{ "company_name": "required", "social_profile_url": "optional", "profile_photo": "optional" }
```

- Missing keys mean `off`; a new space may store `{}`. MySQL does not allow a plain literal default on a JSON column, so there is no database default: the application always writes the value.
- Unknown keys and unknown values are rejected by application validation.
- Name and email are always shown and required, so they are not stored in `form_config`.
- Turning a field off never deletes data already collected. Changing a field to `required` affects only new submissions.

### 4.4 `space_questions`

| column | type | notes |
|---|---|---|
| id | uuid pk | stable identifier; kept when the prompt is edited |
| space_id | uuid FK → `spaces.id` | cascade |
| prompt | string(200) | |
| is_required | boolean | default false |
| position | unsigned tinyint | display order, 1 to 10 |
| created_at, updated_at | timestamps | |

- A space may have zero to 10 questions. No default question is created.
- The limit of 10 is enforced in application code inside a transaction (no database constraint).
- Text questions only in the MVP.

### 4.5 `testimonials`

| column | type | notes |
|---|---|---|
| id | uuid pk | |
| space_id | uuid FK → `spaces.id` | cascade |
| customer_name | string(120) | |
| customer_email | string(254) | as typed, trimmed |
| customer_email_normalized | string(254) | trimmed and lowercased; nothing else is stripped |
| company_name | string(120), null | |
| social_profile_url | string(2048), null | `http` or `https` only |
| profile_photo_path | string, null | object key on the media disk |
| testimonial_text | text | required; at most 2,000 characters |
| rating | unsigned tinyint, null | 1 to 5; null when ratings were disabled |
| social_sharing_consent | boolean | default false |
| consent_recorded_at | timestamp, **null** | set only when consent is true (section 6.3) |
| is_favourite | boolean | default false |
| is_wall_of_love | boolean | default false |
| is_visible | boolean | default true |
| submission_token | string(36) | per rendered form; unique with `space_id` |
| submitted_at | timestamp | the model's creation timestamp |
| updated_at | timestamp | |
| deleted_at | timestamp, null | soft delete |

There is no `user_id` on testimonials; ownership is always derived through the space.

### 4.6 `testimonial_answers`

| column | type | notes |
|---|---|---|
| id | uuid pk | |
| testimonial_id | uuid FK → `testimonials.id` | cascade |
| question_id | uuid, null FK → `space_questions.id` | **set null** when the question is deleted |
| question_prompt | string(200) | snapshot of the prompt shown at submission |
| answer_text | text | at most 1,000 characters |
| position | unsigned tinyint | snapshot of the question's order |
| created_at, updated_at | timestamps | |

- Unique constraint on `(testimonial_id, question_id)`. Behavior with null values is verified in section 7.
- No row is created for an unanswered optional question; whitespace-only answers count as unanswered.
- `question_id` is never null when a row is created by a submission; it becomes null only through question deletion.

### 4.7 `embed_widgets` (Phase 4)

| column | type | notes |
|---|---|---|
| id | uuid pk | |
| space_id | uuid FK → `spaces.id` | cascade |
| public_widget_id | string(32) | unique, random, stable; separate from `id`. Generated from lowercase letters and digits only (at least 24 characters), because MySQL's default collation compares case-insensitively and a mixed-case token would lose entropy and could collide |
| name | string(100) | internal label shown only in the owner's dashboard; generated default such as "Pixel Studio widget"; editable; not unique |
| layout | string | `masonry` (default), `carousel` |
| theme | string | `light` (default), `dark` |
| animation_enabled | boolean | default true |
| background_color | string(7), null | `#RRGGBB`; null means the theme default |
| heading | string(120), null | optional public heading, separate from `name` |
| created_at, updated_at | timestamps | |

### 4.8 `subscriptions` (Cashier-owned; Phase 5 additions only)

Cashier's table stays the single source of truth. A new, additive migration in Phase 5 adds `plan_id` (nullable FK → `plans.id`), `current_period_start` (nullable) and `current_period_end` (nullable). The billing and webhook code keeps them in sync. Existing Cashier migrations are never edited.

---

## 5. Indexes and their purpose

| table | index | purpose |
|---|---|---|
| `spaces` | unique `slug` | Public collection page lookup. |
| `spaces` | `user_id` | The owner's spaces list; ownership scoping; cascade from `users`. |
| `space_questions` | `(space_id, position)` | Loading questions in display order; cascade from `spaces`. |
| `testimonials` | `(space_id, submitted_at)` | Inbox sorting (newest/oldest) and dashboard date ranges. |
| `testimonials` | `(space_id, is_favourite, submitted_at)` | Favourite-first sorting. |
| `testimonials` | `(space_id, is_wall_of_love, is_visible, submitted_at)`, explicitly named `testimonials_widget_feed_index` | The public widget query, the hottest public read. |
| `testimonials` | `(space_id, customer_email_normalized)` | Unique-customer counts and email search. |
| `testimonials` | unique `(space_id, submission_token)` | Duplicate-submission guard. |
| `testimonial_answers` | `(testimonial_id, position)` | Showing answers in order; cascade from `testimonials`. |
| `testimonial_answers` | unique `(testimonial_id, question_id)` | One answer per question per testimonial. |
| `testimonial_answers` | `question_id` | Makes the set-null on question deletion efficient. |
| `embed_widgets` | `space_id` | Listing a space's widgets; cascade from `spaces`. |
| `embed_widgets` | unique `public_widget_id` | Public widget lookup. |
| `plans` | unique `key`, unique `stripe_price_id` | Plan lookup by key; resolving a Stripe price to a plan. |
| `stripe_webhook_events` | unique `stripe_event_id` | Idempotency claim. |

**MySQL identifier limit:** index and constraint names are limited to 64 characters. Laravel's generated name for the widget-feed index would be 67 characters and fail on MySQL, so it is named explicitly; every other generated name is at most 53 characters (checked). Any index added later must be checked the same way.

Every foreign key column leads at least one listed index, which InnoDB requires (it would create a hidden one otherwise) and which SQLite and PostgreSQL would not create at all. No separate indexes are kept on `rating`, `is_visible` alone, or `deleted_at`: at the plan limits (at most 25 spaces × 1,000 testimonials per owner) they add write cost without helping reads.

---

## 6. Business rules

### 6.1 Questions and answers
- The public form always has name, email and a required "Your testimonial" box. The label is fixed in the interface and not stored or configurable.
- Each custom question has its own answer box. A question marked required must be answered; an optional one may be left blank.
- Answers are stored in `testimonial_answers` with a snapshot of the prompt, so later edits or deletions of a question do not change history.
- Widget cards show `testimonial_text` in the MVP. How answers appear on cards is deferred.

### 6.2 Customers and email
- Customer details live on each testimonial. There is no `customers` table.
- A unique customer is a distinct `customer_email_normalized` within an owner's spaces. Plus-tags and Gmail dots are not stripped.
- Hidden testimonials count as customers; soft-deleted testimonials do not.
- Inbox search covers `customer_name`, `customer_email_normalized`, `testimonial_text` and `testimonial_answers.answer_text`.

### 6.3 Consent
- `social_sharing_consent` is a boolean that defaults to false. The checkbox is unchecked by default, and submission does not require consent.
- `consent_recorded_at` is **nullable** and is set **only when consent is true**. It is null whenever consent is false.
- The server sets it from the same clock reading as `submitted_at`. A client-supplied value is ignored, and owners cannot edit consent or its timestamp.
- Invariant: `consent_recorded_at IS NOT NULL` if and only if `social_sharing_consent = true`. It is enforced in application code and covered by tests. A database CHECK constraint is deliberately not used: MySQL before 8.0.16 parses but silently ignores CHECK constraints, and SQLite cannot add one to an existing table, so it could not be relied on across the engines this project runs on.
- PRD 6.2 asks to store "the consent value and submission timestamp". Both are stored: the consent value, and `submitted_at` as the moment of the decision. For true consent, `consent_recorded_at` equals `submitted_at`.
- Consent does not publish anything. Widget display follows visibility and the Wall of Love flag only.

### 6.4 Deletion
- Deleting a testimonial sets `deleted_at`. It disappears from the inbox, public views and analytics. Its row, answers and photo remain until its parent space is deleted.
- Deleting a space permanently deletes the space, all its testimonials (including soft-deleted ones), their answers, its questions, its widgets and the uploaded photo files.
- Photo files are removed by application code. Failures are logged and never leave the space undeleted.
- Deleting a user account permanently deletes their spaces and all dependent data. The account-deletion flow first deletes each of the user's spaces through the same routine used for space deletion, so every photo file is cleaned up (the database cascade alone would leave the files behind; failures are logged and never block). Then the account is deleted and the cascade removes everything that remains.
- Order in Phase 5: the Stripe subscription cancellation must succeed first; if it fails, account deletion is blocked and an error is shown. Nothing is deleted in that case.

### 6.5 Plans and subscriptions
- Free is the absence of a valid paid subscription; Free users have no subscription row.
- The plan is resolved from Cashier's subscription validity plus a match against `plans.stripe_price_id`.
- A cancelled subscription stays Paid through `ends_at`, then resolves to Free.
- `past_due` stays Paid while Stripe retries. From Phase 2, `Cashier::keepPastDueSubscriptionsActive()` is registered in the application's service provider together with the plan resolver. The payment-failed notice on the dashboard and billing page is built in Phase 5. `incomplete`, `incomplete_expired` and `unpaid` resolve to Free. There is no app-managed grace period.
- The resolver does not trust Cashier's `valid()` alone. In the installed Cashier, `valid()` is `active() || onTrial() || onGracePeriod()`, and `onTrial()` and `onGracePeriod()` look only at dates, not at the Stripe status. A subscription in an excluded status that still has a future `trial_ends_at` or `ends_at` would look valid. The resolver therefore also checks the status explicitly.
- Paid requires all of: type `default`; `stripe_price` equal to the paid plan's `stripe_price_id`; status not in {`incomplete`, `incomplete_expired`, `unpaid`, `paused`}; and `valid()` true. Complete matrix (every row is a required unit test, section 8):

| situation | resolves to |
|---|---|
| no subscription row | Free |
| `active`, `ends_at` null | Paid |
| `trialing`, `trial_ends_at` in the future | Paid |
| `past_due` (Stripe retrying) | Paid |
| cancel scheduled, `ends_at` in the future (status `active`, `trialing` or `past_due`) | Paid until `ends_at` |
| `canceled`, `ends_at` in the past | Free |
| `incomplete` (also when `trial_ends_at` or `ends_at` is in the future) | Free |
| `incomplete_expired` (same date variations) | Free |
| `unpaid` (same date variations) | Free |
| `paused` | Free (assumption, see section 9) |
| price different from the paid plan's `stripe_price_id`, or paid `stripe_price_id` null | Free |
| subscription type other than `default` | Free |
- Paid access activates only after a signature-verified webhook confirms a usable subscription.
- Account deletion must cancel the Stripe subscription first (Phase 5). If cancellation fails, deletion is blocked and an error is shown (section 6.4).
- Limits are enforced on the server inside transactions: the owner row is locked when creating a space, and the space row when saving a testimonial (counting non-deleted testimonials). Nothing is deleted when a plan changes; over-limit accounts simply cannot add. On MySQL (InnoDB) `lockForUpdate` takes a real row lock; the lock must be the first statement in the transaction, before the count, so the count sees the latest committed rows. Transactions that lock use a small retry count (for example `DB::transaction($callback, 3)`) to absorb deadlocks.

### 6.6 Stripe webhook ledger
- The signature is verified before anything is recorded or processed. Cashier verifies only when a secret is configured, so outside local development the endpoint must refuse to process webhooks when `STRIPE_WEBHOOK_SECRET` is unset. **This guard is delivered in Phase 2**, because Cashier's webhook route is already registered in the project (Cashier registers it by default and the application does not disable it) and Phase 2 begins trusting the `subscriptions` table. Outside the `local` environment, an unset secret makes the endpoint fail closed: it answers with an error, processes nothing and logs a configuration error without the payload. With a secret set, Cashier's own signature check stays in force. The only unsigned path is the `local` environment; setting the secret locally (the Stripe CLI prints one) is still recommended.
- Every verified event gets a row, including event types the app does not act on. The row is the atomic claim (unique `stripe_event_id`).
- Claiming, handling and setting `processed_at` happen in one database transaction. If handling fails, the claim rolls back and a retry runs normally. A committed row means the event is done.
- An already-processed event returns success without repeating its effects.
- No raw payloads are stored and no retention job is included in the MVP.

### 6.7 Widgets
- A widget shows a testimonial only when it belongs to the widget's space, `is_visible` is true, `is_wall_of_love` is true and `deleted_at` is null.
- Hidden testimonials never appear, even when marked for the Wall of Love. Un-hiding makes a testimonial appear in every widget of the space.
- A new widget starts by showing all eligible testimonials. There is no per-widget selection, no rating or quantity filter and no widget-count limit in the MVP.

### 6.8 Owner edits
Owners can edit `testimonial_text`, `customer_name`, `company_name`, `social_profile_url` and the text of answers. Email, rating, consent, photo and `submitted_at` are not editable. There is no audit history; `updated_at` records the change.

### 6.9 Search
- Matching is case-insensitive and uses the same query on MySQL and SQLite. On MySQL this comes from the column collation (Laravel's default `utf8mb4_unicode_ci`, which is also accent-insensitive; accepted). Email search compares against the stored lowercase `customer_email_normalized`.
- `%`, `_` and the escape character in the search term are escaped so they match literally, using `LIKE ? ESCAPE '!'` with `!` as the escape character. A backslash is not used, because MySQL and SQLite treat `'\\'` differently in an `ESCAPE` clause. The term is capped at 100 characters.
- Every inbox query is scoped to the signed-in owner's spaces and excludes soft-deleted testimonials.
- Answer text is searched with an `EXISTS` query. There is no full-text engine and no database-specific FTS schema.

### 6.10 Ratings
Ratings default to enabled for new spaces. When ratings are later disabled, existing rating values are kept, but rating statistics exclude spaces with ratings disabled (PRD 5.4).

### 6.11 Stale public forms
- A customer's form goes stale when the owner edits questions while it is open. A stale submission is **never accepted silently and never partly saved**.
- The public form carries a `form_version`: a hash of the space's questions (each question's `id`, `prompt`, `is_required` and `position`, in position order). It is derived on demand and not stored, so there is no column. Adding, removing, reordering or editing any question, or changing its required flag, changes the version.
- On submit, inside the transaction that locks the space row, the server recomputes the version from the current questions and compares it. A mismatch is rejected with a validation error: "This form was updated while you were filling it in. Please check the questions below and submit again." The page reloads with the current questions and keeps what the customer typed (answers are matched by question id; answers to deleted questions are dropped from the form, not saved).
- `UpdateSpace` locks the same space row, so a question cannot be deleted between the version check and the insert. As a second line of defense, a foreign-key violation raised while inserting answers (SQLSTATE 23000) is caught, the transaction rolls back, any stored photo is removed, and the same validation error is returned. It never surfaces as a 500.
- Answers for question ids that never belonged to the space are rejected as invalid input.
- Other configuration changes need no version: fields turned off are ignored, and fields that became required fail ordinary validation.
- Order of checks: honeypot, duplicate-token no-op, form version, validation.

### 6.12 Time and time zones
- **Storage:** every timestamp is UTC. The project's `config/app.php` fixes `'timezone' => 'UTC'` (checked).
- **Dashboard policy (Phase 3):** every date range and bucket is computed in UTC. A day is a UTC calendar day, a week is an ISO week starting Monday in UTC, a month is a UTC calendar month, and 7, 30 and 90 days are rolling windows ending at the current UTC instant and aligned to UTC day boundaries. One shared helper is used by all analytics queries, and chart labels state UTC.
- **MySQL connection time zone (flagged, not changed):** the project's `mysql` connection in `config/database.php` has no `timezone` setting (checked), so the session uses the MySQL server's time zone. MySQL converts `TIMESTAMP` values through the session time zone, which can shift stored instants when the server is not on UTC, and SQL date functions (`DATE()`, `GROUP BY`) would bucket in the session zone. **Required for implementation and testing:** either set the connection `timezone` to `+00:00` (a configuration change that needs your approval first) or do all bucketing in PHP from UTC values. The scratch-database run must include a test that stores a timestamp at 23:30 UTC under a non-UTC session time zone and asserts that it lands in the correct UTC day.

---

## 7. Database engine: MySQL first, SQLite-compatible

**Engine.** The project's `.env` sets `DB_CONNECTION=mysql` (host `127.0.0.1`, port `3311`, database `Testimonials`); no credentials are recorded here. MySQL is the target for development and production. The default PHPUnit run still uses SQLite in memory (`phpunit.xml`), so every migration must work on both. Nothing in this model changes the database configuration.

**Rules for MySQL compatibility**

| topic | rule |
|---|---|
| Storage engine | Server default, expected InnoDB: the project's `mysql` connection sets `'engine' => null`, so tables use whatever default the MySQL server has. InnoDB is required for foreign keys, cascade and set-null. The MySQL-only test (section 8) verifies the actual engine of the created tables. |
| UUID keys | Laravel's `uuid`/`foreignUuid` create `CHAR(36)`. Foreign keys must use `foreignUuid` so type and collation match the referenced key. Time-ordered UUIDs suit InnoDB's clustered primary key; Laravel's `HasUuids` is believed to generate them in Laravel 11 and later, but **this is unverified against the installed Laravel 13.37**. Verify it in the installed framework source before relying on it, and choose the trait or generator that produces time-ordered IDs if it does not. |
| Bigint keys | `spaces.user_id` uses `foreignId`, matching `users.id` (unsigned bigint). |
| Unique with nulls | MySQL permits several NULLs in a UNIQUE index (documented behavior); see the table below. |
| JSON | `form_config` is the only JSON column. No database default (section 4.3). It is never indexed or queried by path. |
| Identifier length | Names are at most 64 characters; one index is named explicitly (section 5). |
| Index size | The widest index (`space_id` plus an email, utf8mb4) is about 1.2 KB, well under InnoDB's 3,072-byte limit. |
| Collation | Default `utf8mb4_unicode_ci`: case-insensitive, accent-insensitive. Harmless for `slug`, UUIDs and `plans.key`; the widget token is lowercase for this reason. Stripe IDs (`evt_…`, `price_…`) are case-sensitive in principle, but two distinct IDs differing only by case are not a realistic risk, so the default collation is accepted. Unique-customer counts are also accent-insensitive for non-ASCII addresses (accepted). |
| Timestamps | `TIMESTAMP` with second precision. `consent_recorded_at` and `submitted_at` are written from the same value, so they are equal when read back. Nullable timestamp columns are declared `nullable()` explicitly. |
| Time zone | All stored timestamps are UTC; the MySQL connection has no `timezone` configured. See section 6.12 for the required follow-up. |
| DDL | MySQL DDL is not transactional. Each migration creates exactly one table, so a failed migration leaves at most that table; `migrate:fresh` on a scratch database recovers. |
| Not used | `NULLS NOT DISTINCT` (PostgreSQL only; never use it), CHECK constraints, partial indexes, triggers, full-text indexes, generated columns. |

**Unique `(testimonial_id, question_id)` when `question_id` becomes null**

- **MySQL (target):** the MySQL manual states that a UNIQUE index permits multiple NULL values in columns that may hold NULL, in InnoDB as in other engines. So after `ON DELETE SET NULL`, one testimonial may hold several answers with a null `question_id`, while a second answer for the same non-null question is still rejected. **This has not been run against a live MySQL by me:** the design workspace has no MySQL server and cannot reach yours. The test in section 8 verifies it on your MySQL.
- **SQLite and PostgreSQL (verified earlier, 2026-10-18, SQLite 3.45.1 with foreign keys on, PostgreSQL 16.15):** the same sequence gives the same result: second answer to (T1, Q1) rejected; deleting Q1 and Q3 leaves T1 with two null rows; a second answer to (T1, Q2) is still rejected; deleting the space cascades cleanly. These runs matter because the default test suite runs on SQLite.

Resulting rules:
1. Rely on null values being distinct in the unique index. This is required, because a testimonial can have answers to several questions that are later deleted.
2. Never declare the constraint with `NULLS NOT DISTINCT` (it would break the delete-two-questions case on PostgreSQL, and MySQL has no such option).
3. The database cannot forbid inserting a null `question_id`, so the application must never do it. A submission always sets `question_id`, and a test asserts this.
4. Foreign keys must be enforced. InnoDB always enforces them. On SQLite Laravel's default (`DB_FOREIGN_KEYS=true`) enables them; the set-null test fails loudly if an environment turns them off.

**Laravel and Cashier facts relied on** (read from the installed source): Cashier's `Subscription::valid()` is `active() || onTrial() || onGracePeriod()`; `Cashier::$deactivatePastDue` defaults to true and `keepPastDueSubscriptionsActive()` switches it; `unpaid` and `incomplete_expired` are never active; the webhook controller verifies signatures only when `cashier.webhook.secret` is set.

**Verification status** (update this table when the scratch-database run in task 8.8 has actually happened):

| item | status |
|---|---|
| Null-unique sequence on SQLite 3.45.1 and PostgreSQL 16.15 | Run in the design workspace on 2026-10-18 |
| Null-unique behavior on MySQL | **Unverified.** Documented MySQL behavior only; not run. |
| Phase 2 migrations on MySQL | **Unverified.** The migrations do not exist yet and have not run on MySQL. |
| `TestimonialAnswerIntegrityTest` on MySQL | **Unverified.** The test does not exist yet. |
| MySQL connection time zone behavior (section 6.12) | **Unverified** |
| `HasUuids` time-ordered IDs in the installed Laravel | **Unverified** |
| Cashier `valid()`, `active()`, `onTrial()`, `onGracePeriod()`, the `past_due` flag, route registration default and the conditional signature check | Read from the installed `Cashier.php`, `Subscription.php` and `WebhookController.php`; Cashier's route-registration file itself was not inspected |
| Collation `utf8mb4_unicode_ci`, `'engine' => null`, no connection `timezone`, app `'timezone' => 'UTC'` | Read from the project's `config/database.php` and `config/app.php` |

---

## 8. Tests the model requires

PHPUnit, in the repository's existing style (`RefreshDatabase`, `test_*` methods).

**Answer integrity** (`tests/Feature/Testimonials/TestimonialAnswerIntegrityTest.php`):
- Deleting questions sets `question_id` to null on their answers and keeps the prompt snapshots and answer text unchanged.
- A testimonial whose Q1 and Q3 are both deleted ends with two null-`question_id` rows and no error.
- A second answer to the same question for the same testimonial is rejected, including after other questions of that testimonial were deleted.
- Deleting a space removes its questions, testimonials and answers without a constraint error.
- A submission never creates an answer with a null `question_id`.
- The class runs on whichever connection the suite uses, so it passes on the default SQLite run and on MySQL. It must be run on MySQL before the schema is approved (see below).
- MySQL-only test (skipped on other engines): the three answer-related tables use InnoDB, and the unique index on `(testimonial_id, question_id)` is reported as unique by `SHOW INDEX`.
- Run on MySQL against a **separate scratch database**, never the development database: `RefreshDatabase` drops and recreates every table. Example: `DB_CONNECTION=mysql DB_DATABASE=testimonials_test php artisan test --filter=TestimonialAnswerIntegrityTest` (credentials come from the existing environment; nothing is written to `.env` or `phpunit.xml`). The MySQL-specific test refuses to run unless the database name ends in `_test`.
- Running the full migration set with `php artisan migrate:fresh` on that scratch database also proves the identifier-length and foreign-key-type rules.

**Consent** (`tests/Feature/Testimonials/SubmitTestimonialConsentTest.php`):
- Consent given → `social_sharing_consent` true and `consent_recorded_at` equal to `submitted_at` (both read back from the database, since MySQL stores whole seconds).
- Checkbox unticked or field absent → `social_sharing_consent` false and `consent_recorded_at` null.
- A client-supplied `consent_recorded_at` is ignored in both cases.
- The owner's edit endpoint cannot change either consent field.

**Plan resolution** (unit): one test per row of the status matrix in section 6.5, including the `incomplete`, `incomplete_expired` and `unpaid` rows with a future `trial_ends_at` and with a future `ends_at`, a `past_due` subscription resolving to Paid, a cancelled subscription flipping to Free once `ends_at` passes, a mismatching price, a null paid `stripe_price_id`, and a non-`default` type. A separate test asserts that `Cashier::$deactivatePastDue` is false after the application boots.

**Stale forms** (feature): a question deleted after the form was opened; a prompt edited; the required flag changed; questions reordered; each is rejected with the friendly message, stores nothing (no testimonial, no answers, no file) and keeps the typed input. A foreign-key violation forced during the answer insert returns the same validation error and not a 500. An answer for a question id of another space is rejected. A change to optional-field states alone is not rejected as stale.

**Webhook guard** (feature): with `STRIPE_WEBHOOK_SECRET` unset and the environment not `local`, a request to Cashier's webhook route is rejected and changes nothing (no subscription is created or updated); with a secret set and an invalid signature it is rejected; with the `local` environment and no secret it is the only case that reaches the handler.

**Account deletion** (feature): deleting an account that owns spaces with stored photos removes every photo file and every row; a file-removal failure is logged and does not stop deletion.

**Time zone** (MySQL scratch run): the 23:30 UTC test from section 6.12.

**Other model tests** (detail in the OpenSpec change): plan resolution for every Stripe status including `past_due`; space and testimonial limits and downgrade behavior; form-config validation (unknown keys and values rejected, missing keys mean `off`); question limit of 10 and zero questions allowed; per-question required validation; duplicate submission token; email normalization and unique-customer counts; ownership returning 404; cascade on space and user deletion; widget eligibility; webhook-event uniqueness and claim semantics; inbox search escaping and owner scoping.

---

## 9. Pending items that do not block Phase 2

- **Paid price and currency:** the PRD says 999 in "the product's configured billing currency". Confirm the unit and currency before Phase 5; until then `plans.currency` stays null.
- **MySQL connection time zone:** needs your approval before any configuration change (section 6.12).
- **`paused` subscriptions:** treated as Free. The PRD does not mention them and Cashier's `active()` does not exclude them, so the resolver excludes them explicitly. Confirm or change before Phase 5.
- **`HasUuids` ordering:** verify against the installed Laravel 13.37 (section 7).
- **MySQL version and scratch database:** confirm the server version (`SELECT VERSION();`; MySQL 8.0 or later is assumed for JSON and CHECK behavior) and create a scratch database ending in `_test` for the MySQL test run. Neither is changed by this design.
- **Test tooling for the PRD's end-to-end flow:** no browser-test package is installed and none is added by this design. Decide how to satisfy PRD 17's end-to-end list before Phase 6.

---

## 10. Additions beyond the PRD and deviations

**Supporting additions** (columns or tables the PRD does not list):

| item | purpose |
|---|---|
| `space_questions`, `testimonial_answers` tables | Normalized replacement for the PRD's `questions` and `question_responses` JSON; the PRD allows either. |
| `space_questions.is_required` | Per-question required setting. |
| `customer_email_normalized` | Unique-customer counts and email search on a lowercase value. |
| `submission_token` | Duplicate-submission protection (PRD 6.3). |
| `embed_widgets.name` | Internal label for telling widgets apart. |
| `plans.key` | Stable lookup (`free`, `paid`) independent of generated IDs. |
| `form_version` (computed, no column) | Detects stale public forms (section 6.11). |

**Deviations and mappings** (a PRD-specified item is stored differently):

| PRD | this model | reason |
|---|---|---|
| PostgreSQL suggested by the PRD (section on stack) | MySQL, as configured in the project's `.env` | The PRD defers to the repository's conventions; the schema stays free of engine-specific features. |
| `form_config (JSON)` | JSON, with one state per field (`off`, `optional`, `required`) | Still JSON; the shape makes "off but required" impossible. |
| `consent_recorded_at` | Nullable; set only when consent is given | Records when consent was granted; the decision time is `submitted_at`. |
| `profile_photo_url` | `profile_photo_path` (object key) | PRD 12 allows URLs or object keys; a key is not tied to a storage domain. Column name only. |
| Opaque IDs for internal records (PRD 13) | Bigint on `plans` and `stripe_webhook_events` | These are an internal lookup and an internal ledger that are never exposed in URLs or payloads, and a bigint `plan_id` sits naturally beside Cashier's bigint columns. All domain tables use UUIDs. |
| Own `subscriptions` table with `plan_id`, period dates and Stripe IDs (13.3) | Cashier's `subscriptions` plus additive Phase 5 columns | Single source of truth. Customer ID is `users.stripe_id`; subscription ID is `subscriptions.stripe_id`. |
| `users.password_hash` (13.1) | Laravel's `password` column | Existing authentication is preserved. |
| `testimonial_text` plus `question_responses` (13.5) | `testimonial_text` plus `testimonial_answers` | Matches the PRD's two-part shape. |
| Plan-limit enforcement listed under Phase 3 (section 18) | Enforced from Phase 2 | **Intentional.** The PRD's acceptance criteria for spaces (5) and submissions (10.1) already require server-side limits, so they ship with those features. |
| Name and email "enabled and required by default" (5.3) | Always shown and always required; not configurable | **Interpretation.** The PRD table lists their required setting as "Required", not "Configurable"; only company name, social profile URL and profile photo are configurable. Revisit if owners should be able to turn them off. |

---

## 11. Decision log

| # | Decision |
|---|---|
| 1′ | Required main `testimonial_text`, plus per-question answers in `testimonial_answers` with a prompt snapshot. Widget cards show `testimonial_text` for the MVP. |
| 2′ | Per-question required flag. There is no "at least one non-empty answer" rule. |
| 3 | `space_questions` and `testimonial_answers` tables. Foreign key set to null on question delete, with a prompt snapshot. No rows for unanswered optional questions. Maximum 10 questions per space. |
| 4 | Customer details live on each testimonial. Stored normalized email (trim and lowercase only). Hidden testimonials count as customers, soft-deleted ones do not. No `user_id` on testimonials. |
| 5 | Zero to 10 custom questions, no default question. The public form always has name, email and a required "Your testimonial" box with a fixed interface label. |
| 6 | `form_config` is JSON with one state per optional field. Missing keys mean `off`. Unknown keys and values are rejected. Turning a field off keeps old data; `required` applies only to new submissions. |
| 7 | Testimonials are soft-deleted. Spaces and users are hard-deleted with a full cascade, including photo files (failures are logged and never block). Account deletion also cancels the Stripe subscription first. |
| 8 | Cashier's `subscriptions` is the single source of truth, with additive Phase 5 columns. Free means no row. Plan resolved from Cashier validity plus a price match. A cancelled subscription stays Paid until `ends_at`. |
| 9 | UUID primary keys for the five domain tables; bigint for `plans` and `stripe_webhook_events`; `users.id` and Cashier IDs stay bigint. Authenticated routes bind by UUID with ownership checks. Public pages use the slug and `public_widget_id`. |
| 10 | Widgets select testimonials through the space-wide Wall of Love flag. No per-widget selection, filters or widget limit in the MVP. `public_widget_id` is a stable random URL-safe token. |
| 11 | `embed_widgets.name` is a required, editable, non-unique internal label with a generated default. `heading` stays the public heading. |
| 12 | Webhook ledger: verify first, claim with a unique constraint inside the handling transaction, store every verified event, mark `processed_at` only on success, no raw payloads, no retention job. The endpoint refuses to process webhooks when the signing secret is unset outside local development. |
| 13 | From Phase 2 (see 18), `past_due` stays Paid during Stripe's retries; a payment-failed notice is shown (Phase 5); `incomplete`, `incomplete_expired` and `unpaid` are Free; Paid activates only after a signature-verified webhook; no app-managed grace period; status transitions are unit tested. |
| 14 | Inbox search covers name, normalized email, main text and answer text; case-insensitive and portable; escaped wildcards; owner-scoped; 100-character cap; `EXISTS` for answers; no FTS. The indexes in section 5 are part of the model. |
| 15 | `consent_recorded_at` is nullable and set only when consent is true; null when consent is false. Server-set, never client-supplied, never owner-editable. |
| 16 | The unique constraint on `(testimonial_id, question_id)` relies on null values being distinct: documented MySQL behavior, plus verified identical on SQLite and PostgreSQL. `NULLS NOT DISTINCT` is never used; submissions never insert a null `question_id`; integrity tests are required and must also be run on MySQL. |
| 17 | The target database is MySQL (the project's `.env`), with the default test run on SQLite. Schema and migrations avoid engine-specific features, stay within MySQL's 64-character identifier limit (the widget-feed index is named explicitly), use no database default on JSON, and use a lowercase widget token. The storage engine is "server default, expected InnoDB", verified by a MySQL-only test. |
| 18 | `Cashier::keepPastDueSubscriptionsActive()` is registered in Phase 2 with the plan resolver. The resolver checks status explicitly, because `valid()` ignores status through its trial and grace-period branches. `incomplete`, `incomplete_expired`, `unpaid` (and `paused`, assumed) are Free. A complete status matrix is a required unit test. |
| 19 | Account deletion first deletes each space's photo files through the space-deletion routine, then the account. In Phase 5, Stripe cancellation must succeed first; otherwise deletion is blocked with an error. |
| 20 | Stale public forms: a derived `form_version` over the questions; a mismatch is rejected with a friendly validation message, nothing is stored, input is kept; the check runs under the space-row lock; a foreign-key race is caught and returned as a validation error, never a 500; answers to deleted or changed questions are never silently accepted. |
| 21 | Time policy: all timestamps and all dashboard buckets and ranges are UTC. The missing MySQL connection `timezone` is flagged for implementation and testing; no configuration is changed without approval. |
| 22 | Webhook security: from Phase 2, outside `local`, the webhook endpoint fails closed when `STRIPE_WEBHOOK_SECRET` is unset and never processes an unsigned request. |
| 23 | Recorded deviations: plan-limit enforcement moved from PRD Phase 3 to Phase 2; name and email are always on. MySQL results stay marked unverified until the scratch-database run happens. |

**Accepted assumptions:** `paused` subscriptions resolve to Free; ratings default to enabled; consent is optional (revised by decision 15); the length limits shown in section 4; new questions default to optional; `profile_photo_path` holds an object key; owner edits are limited as in section 6.8; widget defaults are masonry, light theme, animation on and no background color; the question limit is enforced in code.

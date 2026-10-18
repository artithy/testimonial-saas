# Product Requirements Document (PRD)

## Minimal Testimonial SaaS Platform

**Version:** 1.0  
**Status:** Ready for MVP development  
**Product Type:** Multi-tenant SaaS web application  
**Reference Product:** Testimonial.to

---

## 1. Product Overview

Build a minimal SaaS platform that allows businesses, creators, and individuals to collect, manage, and display customer testimonials through shareable testimonial pages and embeddable widgets.

Users can create spaces, configure testimonial forms, share public links, manage submissions, view analytics, and upgrade their subscription.

Customers submitting testimonials must not need an account.

### 1.1 Product Goals

- Make testimonial collection simple and accessible.
- Provide customizable testimonial collection pages.
- Centralize testimonial management in an inbox.
- Allow testimonials to be displayed on external websites.
- Provide useful collection analytics.
- Implement a freemium subscription model with Stripe integration.

### 1.2 MVP Success Criteria

- Users can register, log in, and create spaces.
- Customers can submit testimonials through public links.
- Users can manage, favourite, hide, edit, and delete testimonials.
- Public testimonial widgets display approved, visible testimonials.
- Dashboard analytics reflect actual database records.
- Subscription limits are enforced on the server.
- Stripe payments and subscription status are synchronized correctly.

---

## 2. User Roles

### 2.1 Space Owner

An authenticated user who can:

- Create and configure spaces.
- Access the dashboard and analytics.
- View and manage testimonials.
- Configure and generate embed widgets.
- Manage subscription and billing.

### 2.2 Customer / Testimonial Submitter

A public visitor who can:

- Open a shared testimonial page.
- Enter their details and testimonial.
- Provide a rating if enabled.
- Give consent for social media sharing.
- Submit without registration or login.

### 2.3 System

The application is responsible for authentication, authorization, plan enforcement, data storage, public rendering, analytics, and payment synchronization.

---

## 3. Application Modules

The authenticated application should provide the following navigation:

1. Dashboard
2. Spaces
3. Inbox
4. Wall of Love
5. Embed Widgets
6. Billing
7. Account Settings

The public application should provide:

- Public testimonial collection pages.
- Submission confirmation pages.
- Public testimonial embed rendering.

---

## 4. Authentication and Account Management

### Functional Requirements

- Users can register with name, email, and password.
- Users can log in and log out.
- Users can reset forgotten passwords.
- Email addresses must be unique.
- Protected pages must require authentication.
- Users can only access their own spaces and associated data.
- Unauthenticated visitors can access public testimonial pages and public embed widgets.

### Acceptance Criteria

- Successful registration creates a user account.
- Unauthenticated requests to protected routes redirect to login.
- Users cannot access another user's private data by changing URL parameters or resource IDs.

---

## 5. Space Management

A **space** represents a product, service, brand, or project for which testimonials are collected.

### 5.1 Create Space

Users can create a space by configuring:

- Space name (internal identifier).
- Public page title.
- Public page subtitle.
- Custom questions or prompts.
- Enabled form fields and their required status.
- Whether ratings are enabled.
- Page theme.

The internal space name and public page title may be the same by default but should be stored separately.

### 5.2 Custom Questions

- Users can add multiple testimonial questions or prompts.
- Questions are displayed on the public collection page.
- Users can edit or remove questions.
- Each question should have a stable ID and display order.
- Questions must be stored with their submitted responses so historical testimonials remain understandable if a question changes later.

For the MVP, support text-based questions only.

### 5.3 Form Configuration

The following fields should be available:

| Field              | Default  | Required setting |
| ------------------ | -------- | ---------------- |
| Full name          | Enabled  | Required         |
| Email address      | Enabled  | Required         |
| Company name       | Disabled | Configurable     |
| Social profile URL | Disabled | Configurable     |
| Profile photo      | Disabled | Configurable     |

- Name and email must be enabled and required by default.
- Optional predefined fields can be enabled or disabled.
- Users can configure the required status of optional fields.
- The public form must validate all enabled required fields.
- Email and URL values must be validated.
- Profile photo uploads must enforce file type and size restrictions.

### 5.4 Rating Configuration

- Users can enable or disable ratings when creating a space.
- When enabled, customers can submit a rating from 1 to 5 stars.
- When disabled, the rating field must not appear on the public form.
- Rating statistics must exclude spaces where ratings are disabled.

### 5.5 Themes

Provide at least three predefined themes:

- Light
- Dark
- Minimal

Themes should change the public page's colors, typography, and visual treatment while preserving readability and accessibility.

### 5.6 Space Creation Success Page

After creating a space, display a success page containing:

- Confirmation message.
- Public testimonial collection URL.
- Copy-link action.
- Open-public-page action.
- Option to return to the dashboard.

### Acceptance Criteria

- A user can create, edit, and view their spaces.
- The system generates a unique public slug for each space.
- Public URLs do not expose internal database IDs.
- Users cannot exceed their plan's space limit.
- Space configuration is saved and reflected on the public page.

---

## 6. Public Testimonial Collection

Each space has a publicly accessible URL, for example:

`https://yourdomain.com/collect/{space-slug}`

### 6.1 Public Page

Display:

- Space title and subtitle.
- Configured questions.
- Enabled form fields.
- Rating input when enabled.
- Consent checkbox for social media sharing.
- Submit button.

### 6.2 Submission Requirements

- No account or login is required.
- Name and email are required by default.
- The testimonial text is required.
- Every enabled required field must be completed.
- Rating is required when ratings are enabled.
- Social media sharing consent must be explicitly collected.
- Consent must default to unchecked.
- Store the consent value and submission timestamp.
- Validate all submitted data on the server.
- Apply basic rate limiting and spam protection.

The consent applies to social media sharing, not automatic publication everywhere. Public website display must follow the platform's visibility and publication rules.

### 6.3 Confirmation Page

After a successful submission, display a friendly confirmation message.

Suggested copy:

**"Boom! Your kind words are officially in the love bank. Thanks for making someone's day!"**

Use a lighthearted confirmation design with an option to return to the submitting business's website, if a return URL is configured in the future.

### Acceptance Criteria

- Valid submissions are saved to the correct space.
- Invalid submissions display clear validation errors.
- Duplicate submissions are not silently created because of repeated requests or double-clicks.
- Submission success is displayed only after successful persistence.
- Public visitors cannot access private inboxes or customer records belonging to space owners.

---

## 7. Dashboard and Analytics

The dashboard should display useful summary statistics and a time-series chart.

### 7.1 Summary Cards

Display:

- Total testimonials collected.
- Total unique customers who submitted testimonials.
- Total spaces.
- Average rating, where applicable.
- New testimonials within the selected date range.

Define a unique customer using a normalized email address within the owner's account. Use consistent rules for all analytics.

### 7.2 Testimonial Collection Chart

Display the number of testimonials collected over time.

Provide filters for:

- Date range: 7 days, 30 days, 90 days, all time.
- Data density: daily, weekly, monthly, where appropriate for the selected range.
- Space: all spaces or a selected space.

### 7.3 Additional Information

Optionally display:

- Recent testimonials.
- Most recent activity.
- Space-level testimonial counts.
- Average rating per space.

### Acceptance Criteria

- Analytics are derived from persisted records.
- Date filtering uses a consistent timezone policy.
- Chart aggregation matches the selected date range and density.
- Customer counts use distinct normalized email addresses.
- Deleted testimonials are excluded from active collection statistics.

---

## 8. Testimonial Inbox

The inbox is the central location for testimonial management.

### 8.1 Testimonial List

Each item should display:

- Submitter name.
- Email address.
- Company name, if available.
- Profile photo, if available.
- Testimonial text.
- Rating, if enabled for the space.
- Submission date.
- Favourite status.
- Wall of Love status.
- Visibility status.

### 8.2 Inbox Controls

Provide:

- Search by submitter name, email, or testimonial text.
- Filter by space.
- Filter by rating.
- Filter by visibility status.
- Sort by newest and oldest.
- Favourite-first sorting.

### 8.3 Testimonial Actions

All testimonial management must happen in the inbox.

**Favourite**

- Toggle a testimonial's favourite status.
- Favourited testimonials appear at the top when favourite-first sorting is active.

**Wall of Love**

- Toggle a heart icon to include or remove a testimonial from the Wall of Love.
- Only eligible, visible testimonials should appear in the public Wall of Love or embed widgets.

**Edit**

- Allow the owner to edit testimonial text and supported metadata.
- Preserve the original submission timestamp.
- Record an updated timestamp.
- Store the original submission separately only if an audit history is implemented.

**Hide / Show**

- Hide a testimonial from all public displays without deleting it.
- Hidden testimonials remain visible in the owner's inbox.
- Allow the owner to restore visibility.

**Delete**

- Require confirmation before deletion.
- Remove the testimonial from public displays immediately.
- Use a consistent deletion policy and ensure deleted content is excluded from analytics.

**Additional Actions**

- Provide an ellipsis menu containing Edit, Hide/Show, Delete, and other appropriate actions.

### 8.4 Wall of Love Page

Provide a dedicated authenticated page listing testimonials marked for the Wall of Love.

For the MVP, this page is a management view. A separate public Wall of Love page can be added later.

### Acceptance Criteria

- All actions persist in the database.
- Hidden testimonials never appear in public widgets.
- Favourites affect ordering only, not public visibility.
- Wall of Love selection does not override visibility restrictions.
- Owners can only manage testimonials belonging to their spaces.

---

## 9. Embed Widget Builder

Users can create widgets to display collected testimonials on external websites.

### 9.1 Widget Configuration

Each widget must be associated with a space.

Provide these options:

- Layout: Masonry or Carousel.
- Theme: Light or Dark.
- Animation: Enabled or Disabled.
- Background color: Configurable color picker.
- Optional heading or title.
- Widget preview.

### 9.2 Live Preview

- Display a preview while settings are changed.
- Reflect the selected layout, theme, animation, and background color.
- Use the same rendering logic for preview and public widget output.
- Provide representative testimonial data if the space has no eligible testimonials.

### 9.3 Embed Code Generation

Generate copyable embed code using either:

- JavaScript embed, preferred for flexible responsive rendering.
- Iframe embed, as an alternative for stronger rendering isolation.

Example JavaScript embed:

```html
<div id="testimonial-widget"></div>
<script
    src="https://yourdomain.com/embed.js"
    data-widget-id="PUBLIC_WIDGET_ID"
    defer
></script>
```

The actual implementation may use a different initialization format as long as it is documented and works on external websites.

### 9.4 Public Widget Rendering

- Render only visible testimonials explicitly selected for public display.
- Support responsive layouts.
- Masonry layout displays testimonial cards in a staggered grid.
- Carousel layout displays cards in a navigable horizontal sequence.
- Apply the saved widget configuration.
- Show a reasonable empty state when no eligible testimonials exist.
- Support pagination or incremental loading for large collections.
- Do not expose private user information or internal database IDs.

### 9.5 Security and Compatibility

- Public widget endpoints must return only approved public data.
- Escape untrusted testimonial content to prevent cross-site scripting (XSS).
- Restrict image sources and validate uploaded images.
- Avoid requiring visitors to authenticate.
- Ensure embed scripts do not expose secret keys.
- Document any limitations related to content security policies (CSP), iframe permissions, and cross-origin embedding.

### Acceptance Criteria

- Users can create and save multiple widgets, subject to any defined widget limit.
- The preview updates when settings change.
- Generated code works on a separate website.
- Widget settings persist after reload.
- Hidden and deleted testimonials never appear in the widget.
- Copy-code functionality works.

---

## 10. Freemium Plans and Usage Limits

The MVP has two plans only.

| Feature                        | Free |  Paid |
| ------------------------------ | ---: | ----: |
| Monthly price                  |    0 |   999 |
| Maximum spaces                 |    3 |    25 |
| Maximum testimonials per space |  100 | 1,000 |
| Public testimonial collection  |  Yes |   Yes |
| Inbox and management           |  Yes |   Yes |
| Dashboard analytics            |  Yes |   Yes |
| Embed widgets                  |  Yes |   Yes |

**Pricing assumption:** The paid price is 999 in the product's configured billing currency. Confirm the currency, billing interval, and tax treatment before enabling live payments. Default the initial billing interval to monthly.

### 10.1 Enforcement Rules

- Free users cannot create more than 3 spaces.
- Paid users cannot create more than 25 spaces.
- A space cannot accept more than its plan's testimonial limit.
- Limits must be enforced server-side, not only in the UI.
- Rejected submissions must display a friendly message when a space reaches its limit.
- Existing data must not be deleted when a subscription expires or is downgraded.
- When an account exceeds its downgraded limits, block new spaces and new submissions as appropriate until usage is within limits. Preserve access to existing data and management functions.

### 10.2 Upgrade Notifications

- Free users see an upgrade banner on the dashboard.
- The banner explains the key benefits of upgrading.
- Provide a button that navigates to Billing.
- Paid users should not see the free-plan upgrade banner.

---

## 11. Billing and Stripe Integration

### 11.1 Billing Page

Display both plans with:

- Price and billing interval.
- Space and testimonial limits.
- Feature comparison.
- Current plan indicator.
- Upgrade button for free users.
- Current subscription status for paid users.

### 11.2 Checkout Flow

1. User selects the paid plan.
2. The backend creates a Stripe Checkout Session.
3. The user is redirected to Stripe's secure hosted checkout.
4. The user completes payment.
5. Stripe redirects the user to a success or cancellation URL.
6. The backend verifies the subscription through a signed webhook.
7. The application updates the user's subscription record.
8. The billing page displays the correct plan and status.

### 11.3 Webhook Requirements

Implement Stripe webhook handling for the relevant subscription and checkout events, including:

- `checkout.session.completed`
- `customer.subscription.updated`
- `customer.subscription.deleted`
- `invoice.paid`
- `invoice.payment_failed`

Use Stripe's current event documentation to select and implement the final event-handling logic.

Requirements:

- Verify webhook signatures.
- Make event processing idempotent.
- Store processed event IDs to prevent duplicate processing.
- Never trust the success redirect alone as proof of payment.
- Store Stripe customer and subscription identifiers.
- Handle failed payments, cancellations, and expired subscriptions.
- Keep subscription status synchronized between Stripe and the application.

### 11.4 Subscription Data

Maintain:

- User ID.
- Plan ID.
- Subscription status.
- Stripe customer ID.
- Stripe subscription ID.
- Billing period start and end.
- Created and updated timestamps.

### Acceptance Criteria

- A successful verified subscription activates the paid plan.
- Returning to Billing displays the correct current plan.
- Failed or cancelled checkout does not activate the paid plan.
- Webhook retries do not create duplicate subscriptions.
- The application uses environment variables for Stripe secrets and price IDs.
- Test mode is supported before live payment activation.

---

## 12. Suggested Technical Architecture

Use the following stack unless the project repository already establishes different conventions.

### Frontend

- Next.js with TypeScript.
- Tailwind CSS.
- A consistent component library, such as shadcn/ui.
- Responsive dashboard and public pages.

### Backend

- Next.js server-side functionality and API routes.
- Server-side validation for all mutations.
- A clear service layer for spaces, testimonials, analytics, embeds, and billing.

### Database

- PostgreSQL.
- Prisma ORM or an equivalent typed ORM.
- Database migrations tracked in version control.

### Authentication

- A proven authentication library compatible with the selected stack.
- Secure password hashing, session management, and password reset.

### File Storage

- S3-compatible object storage for profile photos.
- Store file URLs or object keys in the database.
- Validate MIME type, file size, and upload permissions.

### Payments

- Stripe Checkout.
- Stripe webhooks.

### Deployment

- Deploy the web application to a suitable Next.js hosting platform.
- Use managed PostgreSQL and object storage.
- Configure HTTPS, environment variables, database backups, and error logging.

### Architecture Principles

- Use a modular monolith for the MVP; do not introduce microservices.
- Separate public endpoints from authenticated management endpoints.
- Enforce ownership and plan limits in the backend.
- Use reusable components and typed request/response schemas.
- Keep secrets out of client-side code.
- Provide development, testing, and production configurations.

---

## 13. Initial Database Schema

Use UUIDs or equivalent opaque identifiers for internal records.

### 13.1 Users

- `id`
- `name`
- `email`
- `password_hash` or authentication-provider identifier
- `created_at`
- `updated_at`

### 13.2 Plans

- `id`
- `name`
- `price`
- `currency`
- `billing_interval`
- `max_spaces`
- `max_testimonials_per_space`
- `stripe_price_id`
- `is_active`

Seed the Free and Paid plans.

### 13.3 Subscriptions

- `id`
- `user_id`
- `plan_id`
- `stripe_customer_id`
- `stripe_subscription_id`
- `status`
- `current_period_start`
- `current_period_end`
- `created_at`
- `updated_at`

### 13.4 Spaces

- `id`
- `user_id`
- `name`
- `slug`
- `title`
- `subtitle`
- `questions` (JSON or normalized related records)
- `form_config` (JSON)
- `ratings_enabled`
- `theme`
- `created_at`
- `updated_at`

### 13.5 Testimonials

- `id`
- `space_id`
- `customer_name`
- `customer_email`
- `company_name` (nullable)
- `social_profile_url` (nullable)
- `profile_photo_url` (nullable)
- `testimonial_text`
- `rating` (nullable)
- `question_responses` (JSON or normalized related records)
- `social_sharing_consent`
- `consent_recorded_at`
- `is_favourite`
- `is_wall_of_love`
- `is_visible`
- `submitted_at`
- `updated_at`
- `deleted_at` (nullable, if using soft deletion)

### 13.6 Embed Widgets

- `id`
- `space_id`
- `public_widget_id`
- `layout`
- `theme`
- `animation_enabled`
- `background_color`
- `heading` (nullable)
- `created_at`
- `updated_at`

### 13.7 Stripe Webhook Events

- `id`
- `stripe_event_id` (unique)
- `event_type`
- `processed_at`
- `created_at`

Add database indexes for user ownership, space slugs, testimonial submission dates, favourite status, and other commonly filtered fields.

Use appropriate foreign keys and unique constraints. If questions are stored as JSON, preserve historical question wording with each testimonial response.

---

## 14. API and Server Actions

Implement equivalent server actions or REST endpoints according to the chosen framework.

| Module                 | Required operations                                      |
| ---------------------- | -------------------------------------------------------- |
| Authentication         | Register, login, logout, reset password                  |
| Spaces                 | Create, list, read, update, delete                       |
| Public collection      | Get public space, submit testimonial                     |
| Inbox                  | List, search, filter, sort testimonials                  |
| Testimonial management | Edit, favourite, Wall of Love, hide/show, delete         |
| Dashboard              | Summary statistics, time-series analytics                |
| Embed widgets          | Create, list, update, delete, public rendering           |
| Billing                | Get plans, current subscription, create checkout session |
| Stripe                 | Signed webhook event processing                          |

### Implementation Rules

- Validate all inputs with a schema validation library.
- Apply authentication and ownership checks to private operations.
- Enforce plan limits inside the relevant database transaction or server-side service.
- Use pagination for inboxes and other potentially large lists.
- Return consistent error responses.
- Do not expose stack traces, passwords, API keys, or private records.

---

## 15. UI/UX Requirements

- Use a clean, modern SaaS interface.
- Prioritize clarity and ease of use over excessive visual effects.
- Provide a responsive layout for desktop, tablet, and mobile.
- Include loading, empty, success, and error states.
- Use confirmation dialogs for destructive actions.
- Display clear feedback after saving settings or updating testimonials.
- Provide accessible form labels, keyboard navigation, and visible focus states.
- Use consistent colors, typography, spacing, and component behavior.

### Important Empty States

- No spaces created.
- No testimonials collected.
- No search results.
- No Wall of Love selections.
- No eligible testimonials for an embed.
- No active paid subscription.

Each empty state should provide a useful next action.

---

## 16. Security, Privacy, and Reliability

- Enforce per-user resource ownership on the server.
- Validate and sanitize all input.
- Prevent SQL injection, XSS, CSRF where applicable, and unauthorized object access.
- Rate-limit authentication and public submission endpoints.
- Validate uploaded files and restrict their size.
- Protect Stripe secrets and verify webhook signatures.
- Store consent records accurately.
- Do not publicly display email addresses unless explicitly supported by a future, consent-aware feature.
- Ensure public embed responses contain only fields needed for display.
- Use database transactions for operations that must remain consistent.
- Add error logging without logging sensitive payment or authentication data.
- Provide a documented backup and recovery approach.

---

## 17. Testing Requirements

### Unit Tests

- Plan-limit calculations.
- Required-field validation.
- Rating validation.
- Testimonial visibility rules.
- Favourite and Wall of Love logic.
- Analytics aggregation.
- Subscription status transitions.

### Integration Tests

- Registration and login.
- Space creation and configuration.
- Public testimonial submission.
- Inbox management actions.
- Embed widget configuration and rendering.
- Stripe webhook verification and idempotency.
- Plan enforcement under concurrent requests.

### End-to-End Tests

1. Register a new account.
2. Create a space.
3. Configure questions, fields, theme, and ratings.
4. Open the public link in a logged-out session.
5. Submit a testimonial.
6. Confirm that it appears in the owner's inbox and analytics.
7. Favourite, hide, edit, and delete testimonials.
8. Create an embed and verify its output on an external test page.
9. Attempt to exceed Free plan limits.
10. Complete a Stripe test checkout and verify that the Paid plan is activated.
11. Verify that hidden or deleted testimonials never appear in public widgets.

---

## 18. Development Phases

### Phase 1 — Foundation

- Set up the project and database.
- Configure authentication.
- Implement the base layout and protected routes.
- Seed the Free and Paid plans.

### Phase 2 — Spaces and Collection

- Implement space creation and editing.
- Implement form configuration and themes.
- Build public testimonial collection pages.
- Store testimonial submissions and consent.

### Phase 3 — Inbox and Dashboard

- Build testimonial management actions.
- Implement the Wall of Love view.
- Add summary cards and time-series analytics.
- Enforce testimonial and space limits.

### Phase 4 — Embed Widgets

- Build widget configuration.
- Implement live preview.
- Generate JavaScript and iframe embed code.
- Implement secure public widget rendering.

### Phase 5 — Billing

- Build the billing page.
- Integrate Stripe Checkout.
- Implement webhooks and subscription synchronization.
- Add upgrade notifications and usage-limit handling.

### Phase 6 — Quality and Release

- Complete automated tests.
- Perform security and authorization checks.
- Test responsive layouts.
- Configure production environment variables, monitoring, and backups.
- Deploy the MVP.

---

## 19. Out of Scope for the MVP

Do not implement these unless specifically requested:

- AI-generated testimonials or AI rewriting.
- Automated email collection campaigns.
- Email notifications and reminder sequences.
- Third-party CRM integrations.
- Team members and role-based collaboration.
- Advanced moderation workflows.
- Multiple currencies or complex tax management.
- Annual subscriptions and additional pricing tiers.
- Advanced branding customization and custom domains.
- Advanced analytics, conversion tracking, and A/B testing.
- Public testimonial discovery marketplace.

---

## 20. Definition of Done

The MVP is complete when:

- Users can register and securely access their accounts.
- Users can create and configure testimonial spaces.
- Customers can submit testimonials without accounts.
- Owners can manage every testimonial from the inbox.
- Dashboard statistics accurately reflect stored records.
- Embed widgets render eligible testimonials on external websites.
- Free and Paid limits are enforced server-side.
- Stripe Checkout and webhooks synchronize subscription status.
- The billing page reflects the current plan.
- All critical user flows pass automated tests.
- The application is deployable with documented environment variables and setup instructions.

---

## 21. Assumptions and Decisions to Confirm

Use these defaults for initial development unless the product owner specifies otherwise.

1. **Currency and price:** Paid plan costs 999 per month. Confirm whether the currency is BDT or another currency before live payments.
2. **Public visibility:** Newly submitted testimonials are stored as visible by default unless the product owner chooses a moderation-first workflow. Hidden testimonials are excluded from all public displays.
3. **Wall of Love:** This is an inbox selection and public-widget eligibility flag, not a separate moderation system.
4. **Customer identity:** Unique customer counts use normalized email addresses within an owner's account.
5. **Question history:** Submitted responses retain the question wording used at submission time.
6. **Subscription downgrade:** Existing data remains accessible; new resource creation and submissions are restricted when limits are exceeded.
7. **Embed access:** Widgets are publicly readable using unguessable public identifiers; private management operations require authentication.
8. **Deletion:** Deletion removes testimonials from public displays immediately and follows a documented soft- or hard-deletion policy.
9. **Profile photos:** Uploaded photos use managed object storage with file validation.
10. **Plan changes:** Subscription status is updated only after verified server-side payment events.

**Instruction to the AI coding agent:** Implement this PRD incrementally, starting with the foundation and core collection flow. Before writing significant code, inspect the existing repository and follow its established conventions where reasonable. Create database migrations, seed plans, implement authorization and server-side plan enforcement, and add tests alongside each module. Do not leave critical features as mock implementations. Use Stripe test mode until payment flows and webhooks have been verified. Document any unavoidable deviations from this PRD.

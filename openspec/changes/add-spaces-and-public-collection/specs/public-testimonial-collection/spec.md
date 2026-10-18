## ADDED Requirements

### Requirement: Public collection page without login
The system SHALL serve each space's collection page at `/collect/{slug}` to any visitor without authentication. The page data SHALL contain only what is needed to render the form: title, subtitle, questions (ID, prompt and required flag), field states, ratings flag, theme, a per-render submission token and a form version derived from the questions. It SHALL NOT include the owner's identity, the space's internal ID or other spaces' data.

#### Scenario: Logged-out visitor opens a space
- **WHEN** a logged-out visitor requests `/collect/{slug}` for an existing space
- **THEN** the page renders the title, subtitle, name and email fields, the required testimonial text box, one answer box per configured question, enabled optional fields, rating input if enabled, a consent checkbox and a submit button

#### Scenario: Space without custom questions still works
- **WHEN** a visitor opens a space that has no custom questions
- **THEN** the form shows name, email and the required testimonial text, and can be submitted

#### Scenario: Unknown slug
- **WHEN** a visitor requests a slug that does not exist
- **THEN** the response is 404

#### Scenario: Page data does not leak private information
- **WHEN** the page props for a public collection page are inspected
- **THEN** they contain no user ID, owner email, space UUID or testimonial records

### Requirement: Form reflects space configuration
The public form SHALL render the always-present fields (name, email, testimonial text), the optional fields whose state is `optional` or `required`, one answer input per custom question in position order, and the rating input only when ratings are enabled. Required fields and required questions SHALL be marked. The consent checkbox SHALL be unchecked by default.

#### Scenario: Fields that are off are absent
- **WHEN** company name, social profile URL and profile photo are `off`
- **THEN** none of those inputs appear on the page

#### Scenario: Required questions are marked
- **WHEN** a space has one required and one optional question
- **THEN** only the required question is marked as required

#### Scenario: Consent starts unchecked
- **WHEN** the page loads
- **THEN** the social sharing consent checkbox is unchecked

### Requirement: Server-side submission validation
The system SHALL validate every submission on the server against the space's configuration. Name, email and the main testimonial text are always required, and each optional field whose state is `required` SHALL be present. Values for fields that are `off` SHALL be ignored and not stored.

#### Scenario: Missing required fields
- **WHEN** a submission omits the name, email or testimonial text
- **THEN** nothing is persisted and the response contains clear field errors

#### Scenario: Required optional field must be present
- **WHEN** company name is `required` and a submission omits it
- **THEN** the submission is rejected with an error on the company name field

#### Scenario: Off field values are ignored
- **WHEN** company name is `off` and a submission still sends a company name
- **THEN** the stored testimonial has no company name

### Requirement: Field format and length limits
Email SHALL be a valid email address of at most 254 characters, the testimonial text SHALL be at most 2,000 characters, the social profile URL SHALL be a valid `http` or `https` URL, and all other text fields SHALL have maximum lengths.

#### Scenario: Unsafe URL scheme is rejected
- **WHEN** a submission supplies `javascript:alert(1)` as the social profile URL
- **THEN** the submission is rejected with an error on that field

#### Scenario: Invalid email is rejected
- **WHEN** a submission supplies an email that is not a valid address or is longer than 254 characters
- **THEN** the submission is rejected with an error on the email field

#### Scenario: Over-long testimonial text is rejected
- **WHEN** a submission's testimonial text is longer than 2,000 characters
- **THEN** the submission is rejected with an error on that field

### Requirement: Per-question answers
The system SHALL accept at most one answer per custom question of the space, of at most 1,000 characters each. An answer that is empty or contains only whitespace SHALL count as empty. A question marked required SHALL have a non-empty answer; an optional question MAY be left empty. Answers for question IDs that do not belong to the space SHALL be rejected. The system SHALL NOT require any answer when the space has no required questions, because the main testimonial text is always required.

#### Scenario: Required question must be answered
- **WHEN** a question is required and the submission leaves it empty or whitespace-only
- **THEN** the submission is rejected with an error on that question's answer

#### Scenario: Optional question may be skipped
- **WHEN** an optional question is left empty
- **THEN** the submission is accepted and no answer row is created for that question

#### Scenario: Answers are stored with their question
- **WHEN** a submission answers two questions
- **THEN** two answer rows exist for the testimonial, each with its `question_id`, a snapshot of the question's prompt, the answer text and its position

#### Scenario: Foreign question ID is rejected
- **WHEN** a submission contains an answer for a question ID belonging to another space
- **THEN** the submission is rejected and nothing is stored

### Requirement: Answer snapshot survives question changes
Each stored answer SHALL keep its own copy of the question prompt. Deleting a question SHALL set the `question_id` of its answers to null and SHALL leave their prompt snapshot and text unchanged.

#### Scenario: Deleting a question keeps the answer
- **WHEN** a question that has answers is deleted
- **THEN** each of its answers has a null `question_id` and still has its original prompt snapshot and answer text

### Requirement: One answer per question with null-safe uniqueness
The system SHALL enforce one answer per testimonial and question through a unique constraint on `(testimonial_id, question_id)`, relying on null values being distinct so that a testimonial MAY hold several answers whose question was deleted. The system SHALL NOT use `NULLS NOT DISTINCT`.

#### Scenario: Several orphaned answers coexist
- **WHEN** two questions answered by the same testimonial are both deleted
- **THEN** the testimonial has two answer rows with a null `question_id` and no unique violation occurs

#### Scenario: Duplicate answer for a live question is rejected
- **WHEN** a second answer for the same testimonial and the same non-null question is inserted, even after other questions of the testimonial were deleted
- **THEN** the insert fails on the unique constraint

### Requirement: Submissions never insert a null question reference
The submission action SHALL NEVER insert an answer with a null `question_id`; the reference becomes null only through question deletion.

#### Scenario: Submissions never insert a null question reference
- **WHEN** any valid submission is stored
- **THEN** none of its answer rows has a null `question_id`

### Requirement: Same answer integrity on MySQL and SQLite
Answer integrity behavior SHALL be the same on MySQL (InnoDB), the configured database, and on SQLite with foreign keys enabled, which the default test run uses.

#### Scenario: Integrity tests agree across engines
- **WHEN** the answer integrity tests run on MySQL and on SQLite
- **THEN** the set-null, several-orphans and duplicate-rejection results are the same

### Requirement: Stale form detection
The public form SHALL carry a form version computed from the space's questions (each question's ID, prompt, required flag and position, in position order); it SHALL be derived and not stored. On submission the system SHALL recompute the version inside the transaction that locks the space row, after the duplicate-token check, and compare it.

#### Scenario: Question deleted while the form is open
- **WHEN** a customer submits a form after the owner deleted one of its questions
- **THEN** the submission is rejected with the friendly message and nothing is stored

#### Scenario: Question changed while the form is open
- **WHEN** the owner edits a question's prompt, changes its required flag or reorders the questions after the form was rendered
- **THEN** the submission is rejected with the friendly message and nothing is stored

### Requirement: Stale submissions are rejected cleanly
A version mismatch SHALL be rejected with the validation message "This form was updated while you were filling it in. Please check the questions below and submit again.", SHALL store nothing (no testimonial, answers or file) and SHALL keep the customer's typed input. The system SHALL NEVER silently accept an answer to a deleted or changed question.

#### Scenario: Typed input is kept
- **WHEN** a stale submission is rejected
- **THEN** the reloaded form shows the current questions and the values the customer already typed for questions that still exist

### Requirement: Question edits serialize with submissions
Space edits SHALL lock the same space row so a question cannot disappear between the version check and the insert. A foreign-key violation raised while inserting answers SHALL be handled as the same validation error and SHALL NOT produce a 500 response.

#### Scenario: Foreign-key race is not a server error
- **WHEN** a foreign-key violation occurs while inserting answers
- **THEN** the transaction rolls back, any stored photo is removed and the customer sees the same validation message, not a 500

### Requirement: Other configuration changes do not make a form stale
Changes to optional-field states or to the ratings setting alone SHALL NOT make a form stale; they are handled by ordinary validation against the current configuration.

#### Scenario: Other configuration changes are not stale
- **WHEN** only an optional field's state or the ratings setting changed since the form was opened
- **THEN** the submission is validated normally against the current configuration

### Requirement: Rating validation
When ratings are enabled the system SHALL require an integer rating from 1 to 5. When ratings are disabled the system SHALL not store a rating.

#### Scenario: Rating required when enabled
- **WHEN** ratings are enabled and a submission has no rating
- **THEN** the submission is rejected with a rating error

#### Scenario: Out-of-range rating
- **WHEN** a submission has a rating of 0 or 6
- **THEN** the submission is rejected with a rating error

#### Scenario: Rating ignored when disabled
- **WHEN** ratings are disabled and a submission includes a rating
- **THEN** the stored testimonial's rating is null

### Requirement: Consent recording
The system SHALL record the submitter's social media sharing choice in `social_sharing_consent`, which defaults to false when the checkbox is not ticked. Submission SHALL NOT require consent to be true, and consent SHALL NOT by itself make a testimonial publicly visible anywhere.

#### Scenario: Consent is optional
- **WHEN** a submitter leaves the checkbox unticked
- **THEN** the submission is accepted and `social_sharing_consent` is false

#### Scenario: Consent does not change visibility
- **WHEN** a submitter ticks the consent checkbox
- **THEN** the stored testimonial keeps the default state (`is_visible` true, `is_wall_of_love` false) and consent alone makes nothing public

### Requirement: Consent timestamp
`consent_recorded_at` SHALL be nullable and SHALL be set, on the server and from the same clock reading as `submitted_at`, if and only if the customer explicitly gave consent; when consent is false it SHALL remain null.

#### Scenario: Consent given
- **WHEN** a submitter ticks the consent checkbox and submits
- **THEN** the stored testimonial has `social_sharing_consent` true and a non-null `consent_recorded_at` equal to `submitted_at`

#### Scenario: Consent not given
- **WHEN** a submitter leaves the checkbox unticked, or the field is absent
- **THEN** the stored testimonial has `social_sharing_consent` false and `consent_recorded_at` null

### Requirement: Consent timestamp is server-controlled
The timestamp SHALL never be taken from client input and SHALL NOT be editable by the owner. The invariant is enforced in application code and tests, without a database CHECK constraint, so migrations stay identical on MySQL and SQLite.

#### Scenario: Client-supplied timestamp is ignored
- **WHEN** a submission includes a `consent_recorded_at` value, with or without consent
- **THEN** the stored value is determined only by the server as specified above

### Requirement: Optional profile photo upload
When the profile photo field is `optional` or `required`, the system SHALL accept a single image upload of type JPEG, PNG or WebP no larger than 2 MB, validate it on the server, store it under a randomly generated file name on the configured media disk, and save only the object key in `profile_photo_path`. Files that fail validation SHALL NOT be stored.

#### Scenario: Valid photo is stored
- **WHEN** a submission includes a 500 KB PNG and the field is enabled
- **THEN** the file is stored on the media disk under a generated name and the testimonial references its key

#### Scenario: Invalid photo is rejected
- **WHEN** a submission includes a PDF, an SVG, or an image over 2 MB
- **THEN** the submission is rejected with an error on the photo field and no file is stored

### Requirement: Customer email normalization
The system SHALL store the submitted email as entered (trimmed) and a normalized copy in `customer_email_normalized`, produced by trimming whitespace and converting to lowercase only. It SHALL NOT remove plus-tags or dots.

#### Scenario: Email is normalized for matching
- **WHEN** a submission supplies ` Jane.Doe+Promo@Example.com `
- **THEN** `customer_email_normalized` is `jane.doe+promo@example.com`

### Requirement: Default state of new testimonials
New testimonials SHALL be stored as visible, not favourite and not on the Wall of Love, with a submission timestamp, not soft-deleted, and linked to the correct space.

#### Scenario: Defaults on creation
- **WHEN** a valid submission is saved
- **THEN** `is_visible` is true, `is_favourite` is false, `is_wall_of_love` is false, `deleted_at` is null, and `submitted_at` is set

### Requirement: Duplicate submission protection
The system SHALL embed a unique submission token in each rendered form and enforce uniqueness of the token per space, so repeated requests or double-clicks result in exactly one stored testimonial. A repeated token SHALL lead to the same confirmation outcome without creating another record.

#### Scenario: Double submit creates one record
- **WHEN** the same submission token is posted twice
- **THEN** only one testimonial exists and both requests end on the confirmation page

### Requirement: Rate limiting and spam protection
The system SHALL apply a named rate limiter to the public submission endpoint, keyed by client IP address and space, and SHALL include a honeypot field. Submissions that fill the honeypot SHALL NOT be stored and SHALL receive the normal confirmation response.

#### Scenario: Too many submissions are throttled
- **WHEN** a client exceeds the submission rate limit for a space
- **THEN** further submissions receive a 429 response and are not stored

#### Scenario: Honeypot submission is discarded
- **WHEN** a submission has a non-empty honeypot field
- **THEN** no testimonial is stored and the visitor still sees the confirmation page

### Requirement: Confirmation page after persistence
The system SHALL show the friendly confirmation page only after the testimonial has been persisted. Visiting the confirmation URL without a just-completed submission SHALL redirect to the collection page.

#### Scenario: Success shows confirmation
- **WHEN** a valid submission is stored
- **THEN** the visitor is redirected to the confirmation page showing the friendly message

#### Scenario: Failed submission never shows confirmation
- **WHEN** validation fails
- **THEN** the visitor stays on the form with errors and sees no confirmation

#### Scenario: Direct visit to confirmation
- **WHEN** a visitor opens the confirmation URL without submitting
- **THEN** they are redirected to the collection page

### Requirement: Plain-text storage and private records stay private
The system SHALL store submitted text (main text and answers) as plain text (trimmed, without control characters) and SHALL rely on escaped rendering rather than HTML in output. Public routes SHALL never return other visitors' testimonials, emails or owner inbox data.

#### Scenario: HTML in text is not interpreted
- **WHEN** a submission contains `<script>alert(1)</script>` in the testimonial text or an answer
- **THEN** it is stored as literal text and displayed escaped wherever it is rendered

#### Scenario: Public routes expose no testimonial lists
- **WHEN** any `/collect/*` response is inspected
- **THEN** it contains no previously submitted testimonial or customer email

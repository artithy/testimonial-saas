## ADDED Requirements

### Requirement: Authenticated access to space management
The system SHALL require an authenticated, email-verified user for every space management route, consistent with the existing dashboard routes.

#### Scenario: Guest is redirected to login
- **WHEN** an unauthenticated visitor requests any `/spaces` route
- **THEN** the system redirects them to the login page

### Requirement: Create a space with configuration
The system SHALL let an owner create a space by providing an internal name, a public page title, an optional public subtitle, zero to 10 custom questions, form-field configuration, a ratings toggle and a theme. The internal name and public title SHALL be stored in separate columns. When the public title is omitted, the system SHALL default it to the internal name.

#### Scenario: Space is created with valid configuration
- **WHEN** an owner submits a valid space configuration
- **THEN** a space is persisted for that owner with the submitted name, title, subtitle, form configuration, ratings setting and theme, and its questions are persisted as `space_questions` rows
- **AND** the owner is redirected to the space creation success page

#### Scenario: Title defaults to the internal name
- **WHEN** an owner creates a space and leaves the public title empty
- **THEN** the stored title equals the internal name

#### Scenario: Invalid configuration is rejected
- **WHEN** an owner submits a space with a missing name, an unknown theme, or a name longer than the allowed length
- **THEN** nothing is persisted and the response contains field-level validation errors

### Requirement: Unique, opaque public slug
The system SHALL generate a unique public slug for every space when it is created. The slug SHALL be derived from the space name plus a random suffix, SHALL NOT contain or encode an internal database ID, and SHALL remain unchanged when the space is edited so that shared links keep working.

#### Scenario: Two spaces with the same name get different slugs
- **WHEN** two spaces are created with the same internal name
- **THEN** their slugs are different and both are unique in the database

#### Scenario: Slug is stable across edits
- **WHEN** an owner renames a space
- **THEN** the slug is unchanged

### Requirement: Custom text questions with required flags
The system SHALL let an owner add, edit, reorder and remove text-based questions on a space, from zero up to 10 questions per space. Each question SHALL have a stable UUID, a display position and its own required or optional setting that defaults to optional. Existing questions SHALL keep their ID when edited. The system SHALL NOT create a default question when a space is created, and a space with no questions SHALL be valid.

#### Scenario: Space without questions is valid
- **WHEN** an owner creates a space with no questions
- **THEN** the space is created and has no question rows

#### Scenario: Questions keep their IDs when edited
- **WHEN** an owner changes the prompt of an existing question and saves
- **THEN** that question keeps its original ID

#### Scenario: New questions receive IDs and positions
- **WHEN** an owner adds a question without an ID
- **THEN** the system assigns it a new UUID and the position matching the submitted order

#### Scenario: Required flag is stored per question
- **WHEN** an owner marks one question required and another optional
- **THEN** the stored `is_required` values reflect each choice

#### Scenario: Removing a question keeps historical answers readable
- **WHEN** an owner removes a question that testimonials have already answered
- **THEN** the question row is deleted and the existing answers keep their prompt snapshot and text with a null question reference

#### Scenario: Question limit is enforced
- **WHEN** an owner submits more than 10 questions
- **THEN** the request is rejected with a validation error on the questions field

### Requirement: Form field configuration
The system SHALL show full name, email address and the testimonial text on the public form always and as required (not configurable), and SHALL let company name, social profile URL and profile photo each be configured as `off`, `optional` or `required` in the space's `form_config`. A missing key SHALL mean `off`.

#### Scenario: Defaults for a new space
- **WHEN** a space is created without explicit field configuration
- **THEN** company name, social profile URL and profile photo are treated as `off`

#### Scenario: Optional field can be made required
- **WHEN** an owner sets company name to `required`
- **THEN** the stored configuration reflects it

### Requirement: Form configuration validation and data retention
The server SHALL reject unknown keys and unknown values in `form_config`. Turning a field off SHALL NOT delete data already collected, and changing a field to required SHALL apply only to new submissions.

#### Scenario: Unknown key or value is rejected
- **WHEN** a request supplies an unknown field key or a state other than `off`, `optional` or `required`
- **THEN** the request is rejected with a validation error

#### Scenario: Turning a field off keeps collected data
- **WHEN** an owner sets a field to `off` after testimonials have already supplied it
- **THEN** the previously stored values remain on those testimonials

#### Scenario: Required applies to new submissions only
- **WHEN** an owner changes a field to `required` after testimonials were collected without it
- **THEN** existing testimonials are unchanged and only new submissions must provide it

### Requirement: Rating configuration
The system SHALL let an owner enable or disable ratings per space. When ratings are enabled, customers can give a 1 to 5 star rating; when disabled, the public form SHALL NOT show a rating field. New spaces SHALL have ratings enabled by default.

#### Scenario: Ratings default to enabled
- **WHEN** a space is created without specifying ratings
- **THEN** `ratings_enabled` is true

#### Scenario: Ratings can be disabled
- **WHEN** an owner saves a space with ratings disabled
- **THEN** the public collection page for that space has no rating input

### Requirement: Predefined themes
The system SHALL offer at least three predefined themes, `light`, `dark` and `minimal`, that change the public page's colors, typography and visual treatment while keeping body text readable.

#### Scenario: Theme is applied to the public page
- **WHEN** a space is saved with the `dark` theme
- **THEN** the public collection page renders with the dark theme styling

#### Scenario: Only known themes are accepted
- **WHEN** a request supplies a theme value outside the predefined set
- **THEN** the request is rejected with a validation error

### Requirement: Space creation success page
After creating a space, the system SHALL display a success page with a confirmation message, the full public collection URL, a copy-link action, an open-public-page action and a link back to the dashboard.

#### Scenario: Success page shows the public URL
- **WHEN** an owner creates a space
- **THEN** the success page shows `{app url}/collect/{slug}` for that space, a copy-link button, an open-page link and a dashboard link

### Requirement: List, view and edit own spaces
The system SHALL let an owner list their spaces, view a space, and edit its configuration. Saved changes SHALL be reflected on the public page.

#### Scenario: Owner sees only their own spaces
- **WHEN** an owner opens the spaces list
- **THEN** only spaces owned by that user are listed, including an empty state with a create action when there are none

#### Scenario: Edits appear on the public page
- **WHEN** an owner changes the public title and saves
- **THEN** the public collection page shows the new title

### Requirement: Ownership isolation
The system SHALL ensure an owner can only view, edit or delete their own spaces, enforced by a policy on the server. Requests for another user's space SHALL receive a 404 response so that resource existence is not revealed. Authenticated routes SHALL bind spaces by UUID.

#### Scenario: Another user's space cannot be viewed
- **WHEN** user B requests the show, edit, update or delete route for user A's space
- **THEN** the response is 404 and no data from the space is returned or changed

### Requirement: Delete a space
The system SHALL let an owner delete a space after an explicit confirmation. Deleting a space SHALL permanently delete its questions, all its testimonials (including previously soft-deleted ones), their answers and their uploaded photo files, and its public URL SHALL immediately return 404. A failure to remove a photo file SHALL be logged and SHALL NOT prevent the space from being deleted.

#### Scenario: Deleting removes the space and everything beneath it
- **WHEN** an owner confirms deletion of a space that has questions, testimonials (one of them soft-deleted) and answers
- **THEN** the space, questions, all testimonials and all answers are removed and the public URL returns 404

#### Scenario: Photo cleanup failure does not block deletion
- **WHEN** removing a photo file fails while a space is being deleted
- **THEN** the failure is logged and the space is still deleted

#### Scenario: Deleting frees plan capacity
- **WHEN** a Free-plan owner at the 3-space limit deletes a space
- **THEN** the owner can create another space

### Requirement: Account deletion removes owned data and files
Deleting a user account SHALL first delete each of the user's spaces through the same routine used for space deletion, so that every uploaded photo file is removed (failures are logged and never block), and SHALL then delete the account, whose cascade removes all remaining dependent data.

#### Scenario: Account deletion cleans up photos and rows
- **WHEN** a user with spaces, testimonials, answers and stored photos deletes their account
- **THEN** every photo file, space, question, testimonial and answer is removed, followed by the account

#### Scenario: Photo cleanup failure does not block account deletion
- **WHEN** removing a photo file fails during account deletion
- **THEN** the failure is logged and the account is still deleted

### Requirement: Stripe cancellation precedes account deletion (Phase 5)
In Phase 5, cancelling the user's Stripe subscription SHALL succeed before account deletion proceeds; when cancellation fails, deletion SHALL be blocked, nothing SHALL be deleted and an error SHALL be shown. This requirement is recorded here and not implemented in this change.

#### Scenario: Failed cancellation blocks deletion (Phase 5)
- **WHEN** Stripe cancellation fails while a user with a subscription deletes their account
- **THEN** nothing is deleted and the user sees an error

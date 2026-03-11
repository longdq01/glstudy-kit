# Feature Specification: User Authentication System

**Feature Branch**: `001-user-auth`
**Created**: 2026-03-11
**Status**: Draft
**Sprint**: Sprint 2

---

## Clarifications

### Session 2026-03-11

- Q: Should the login endpoint enforce rate limiting or account lockout to prevent brute-force attacks? → A: No rate limiting in MVP — deferred to Phase 2.
- Q: Does logout end all active sessions across all devices, or only the current device's session? → A: Logout ends only the current device's session; other active sessions remain valid.
- Q: What happens when an admin sets a user's account status to inactive — can they still log in and do existing sessions survive? → A: Inactive users cannot log in; any existing sessions are immediately invalidated.
- Q: When an SSO email matches an existing account under a different provider, what does the confirm-link step require from the user? → A: A confirmation screen explaining the link — user clicks "Yes, link my account"; no password required.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 – Email Registration (Priority: P1)

A new learner visits GLStudy for the first time and creates an account using their email address, a chosen username, display name, and password. On successful submission they are immediately signed in and land on the dashboard.

**Why this priority**: Registration is the entry point to the platform. Without it, no other feature is accessible to new users. It is the single most critical path.

**Independent Test**: A tester can open `/register`, submit a valid form, and verify they are redirected to the dashboard as a logged-in learner — delivering immediate platform access.

**Acceptance Scenarios**:

1. **Given** a visitor on the registration page, **When** they submit a valid username, display name, email, and strong password, **Then** their account is created, they are signed in, and they are redirected to the dashboard.
2. **Given** a visitor who enters an email already registered, **When** they submit the form, **Then** a field-level error tells them the email is already in use.
3. **Given** a visitor who enters a username already taken, **When** they submit the form, **Then** a field-level error tells them the username is unavailable.
4. **Given** a visitor who enters a password that does not meet strength requirements, **When** they interact with the password field, **Then** inline validation describes exactly what is missing.
5. **Given** a visitor who enters an invalid email format, **When** they move focus off the email field, **Then** an inline validation error appears immediately without waiting for form submission.

---

### User Story 2 – Email/Password Login (Priority: P2)

A returning learner visits the login page and signs in with their registered email and password. On success they are taken to the dashboard, or to the protected page they were trying to access before being redirected.

**Why this priority**: Returning users must re-enter the platform daily. This is the primary daily entry point for the majority of users.

**Independent Test**: A tester with an existing account can go to `/login`, enter correct credentials, and verify they land on the dashboard with their name shown in the navigation.

**Acceptance Scenarios**:

1. **Given** a registered user on the login page, **When** they enter their correct email and password, **Then** they are signed in and redirected to the dashboard (or the originally requested protected page).
2. **Given** a registered user on the login page, **When** they enter a wrong password, **Then** a generic "invalid credentials" error is shown without revealing which field is wrong.
3. **Given** a visitor who was redirected to login from a protected URL, **When** they successfully sign in, **Then** they are taken to the originally requested URL.
4. **Given** a user who is already signed in, **When** they navigate to `/login` or `/register`, **Then** they are redirected to the dashboard without seeing the form.

---

### User Story 3 – SSO Login with Google or GitHub (Priority: P3)

A visitor clicks "Sign in with Google" (or GitHub), is redirected to the provider's consent screen, and after authorizing is either signed into an existing account or taken through a brief username-selection step to complete registration.

**Why this priority**: SSO lowers the barrier to entry significantly and is a standard expectation for modern platforms.

**Independent Test**: A tester can click the Google SSO button, complete the OAuth2 flow, and verify they are signed in or prompted for a username — without entering a password.

**Acceptance Scenarios**:

1. **Given** a visitor whose SSO provider email already exists as a local account, **When** the OAuth2 flow completes, **Then** they are shown a confirmation step to link the SSO provider to the existing account, and after confirming they are signed in.
2. **Given** a visitor whose SSO provider email has never been registered, **When** the OAuth2 flow completes, **Then** they are shown a username-selection form, and after submitting a valid username they are signed in as a new learner.
3. **Given** a returning user who previously authenticated via Google, **When** they click "Sign in with Google" and authorize, **Then** they are immediately signed in with no extra steps.
4. **Given** a visitor clicking "Sign in with GitHub", **When** the OAuth2 flow completes, **Then** the same new / returning / existing-email scenarios apply identically.

---

### User Story 4 – Session Persistence & Transparent Renewal (Priority: P4)

A logged-in learner closes their browser and returns the next day. Their session is still active. During an active session, when their short-lived credential expires, a new one is obtained transparently without prompting the user.

**Why this priority**: Users must not be forced to log in every 15 minutes. Silent renewal is a fundamental usability requirement.

**Independent Test**: A tester can log in, allow the short-lived credential to expire, navigate to a protected page, and verify they remain signed in with no login prompt.

**Acceptance Scenarios**:

1. **Given** a logged-in user whose short-lived session credential has expired, **When** they navigate to a protected page, **Then** the system silently obtains a new credential and serves the page without interruption.
2. **Given** a logged-in user whose renewal credential has also expired (after 7 days of inactivity), **When** they visit a protected page, **Then** they are redirected to the login page.
3. **Given** a user who has explicitly logged out, **When** their (now-invalidated) renewal credential is presented, **Then** it is rejected and no new session is issued.

---

### User Story 5 – Logout (Priority: P5)

A logged-in user clicks "Logout" in the navigation. Their session is immediately ended server-side, their credentials are cleared from the browser, and they are redirected to the landing page. Attempting to access a protected page afterwards brings them to the login page.

**Why this priority**: Logout is a basic security expectation, especially for shared-device scenarios.

**Independent Test**: A tester can log in, click logout, and verify: (1) they land on the public landing page, and (2) navigating to `/dashboard` redirects them to `/login`.

**Acceptance Scenarios**:

1. **Given** a logged-in user who clicks "Logout", **When** the action completes, **Then** their session is invalidated server-side and they are redirected to the landing page.
2. **Given** a user who has logged out, **When** they navigate to any protected route, **Then** they are redirected to the login page.
3. **Given** a user who has logged out in one browser tab, **When** another tab makes an authenticated request, **Then** the request fails and that tab redirects to login on the next navigation.

---

### User Story 6 – Profile Viewing & Editing (Priority: P6)

A logged-in learner navigates to their profile page and sees their current information (username, display name, email, linked providers). They can update their display name and, if they registered via SSO, add a local password to enable email/password login.

**Why this priority**: Profile management is required for personalisation and for SSO users to gain password-login capability. It does not block access to the platform, hence lower priority.

**Independent Test**: A tester can navigate to `/profile`, change their display name, refresh the page, and verify the change persisted and is reflected in the navigation bar.

**Acceptance Scenarios**:

1. **Given** a logged-in user on the profile page, **When** the page loads, **Then** it shows their username, display name, email, and the list of login providers they have linked.
2. **Given** a logged-in user on the profile page, **When** they submit a new display name, **Then** it is saved and the navigation bar immediately shows the updated name.
3. **Given** an SSO-only user (no local password) on the profile page, **When** they submit a new password meeting strength requirements, **Then** it is saved and they can now use email/password login.

---

### Edge Cases

- What happens when the OAuth2 provider (Google/GitHub) is unavailable during the SSO flow? → The login page shows a graceful error message; email/password login remains available.
- What happens when a user's SSO email matches an account already linked to a different SSO provider (e.g., Google email matches a GitHub-linked account)? → The confirm-link flow is shown so the user consciously merges the providers.
- What happens when a user submits a username or display name that exceeds maximum length? → Field-level validation blocks submission with a "too long" message (username ≤ 50 chars, display name ≤ 100 chars).
- What happens when an SSO-only user tries email/password login before setting a password? → A specific error: "No password set for this account. Sign in with your provider or set a password from your profile."
- What happens when a renewal credential is presented after the user has already logged out? → The credential is rejected; no new session is issued.
- What happens when a user with an active session on two devices logs out on one? → Only the session on the device that initiated logout is ended. The other session remains valid until its own renewal credential expires.
- What happens when an admin deactivates a user account? → The user cannot log in while inactive. Any active sessions are immediately invalidated — protected resources return an authentication error on the next request, redirecting the user to the login page.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow new users to create an account with a unique username, unique email address, display name, and password.
- **FR-002**: System MUST reject registration if the email or username is already in use, with a specific field-level error message.
- **FR-003**: System MUST enforce password strength: minimum 8 characters, at least one uppercase letter, one digit, and one special character (`@$!%*?&`).
- **FR-004**: System MUST allow registered users to sign in with their email address and password.
- **FR-005**: System MUST allow users to initiate sign-in via Google OAuth2.
- **FR-006**: System MUST allow users to initiate sign-in via GitHub OAuth2.
- **FR-007**: System MUST issue a short-lived session credential (15-minute lifetime) and a long-lived renewal credential (7-day lifetime) on every successful authentication.
- **FR-008**: System MUST transparently obtain a new session credential using the renewal credential when the session credential expires, without user action.
- **FR-009**: System MUST allow users to sign out, immediately invalidating both the session and renewal credentials.
- **FR-010**: System MUST check email and username availability in real time during registration, before form submission.
- **FR-011**: System MUST allow authenticated users to view their profile: username, display name, email, and linked login providers.
- **FR-012**: System MUST allow authenticated users to update their display name.
- **FR-013**: System MUST allow SSO-only users to set a local password from their profile page.
- **FR-014**: System MUST redirect unauthenticated users who access a protected page to the login page, preserving the original URL for post-login redirect.
- **FR-015**: System MUST redirect authenticated users who visit `/login` or `/register` directly to the dashboard.
- **FR-016**: When an SSO provider returns an email already associated with a different provider, system MUST present a confirmation screen that explains which account will be linked and requires the user to explicitly click "Yes, link my account" — no password entry is required.
- **FR-017**: When an SSO provider returns a new email, system MUST prompt the user to choose a username before completing account creation.
- **FR-018**: All session credentials MUST be stored exclusively in httpOnly secure cookies; they MUST NOT be accessible to client-side scripts.
- **FR-019**: System MUST reject login attempts from inactive user accounts with a clear error message.
- **FR-020**: When a user account is set to inactive, all of that user's active sessions MUST be immediately invalidated; subsequent authenticated requests MUST be rejected.

### Key Entities

- **User**: A platform account. Attributes: unique identifier, username (unique), display name, email (unique), account status (active/inactive), password (nullable for SSO-only users), email-verified flag, creation timestamp, last-login timestamp.
- **Auth Provider Link**: An association between a user and a third-party identity provider. Attributes: link to user, provider name (local / google / github), provider-assigned user identifier, whether it is the primary sign-in method, timestamp of when the link was established. A single user may have multiple provider links.
- **Session**: A pair of credentials issued on successful authentication. The short-lived credential (15 min) authorises access to protected resources. The long-lived renewal credential (7 days) enables silent renewal. Both are invalidated on explicit logout.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: New users can complete email registration (form fill → dashboard) in under 2 minutes on their first attempt.
- **SC-002**: Returning users can sign in (form fill → dashboard) in under 5 seconds after submitting credentials.
- **SC-003**: SSO login (button click → dashboard, including the OAuth2 redirect) completes in under 10 seconds under normal network conditions.
- **SC-004**: 100% of requests made after explicit logout are rejected — no protected resource is accessible using invalidated credentials.
- **SC-005**: 100% of unauthenticated attempts to access a protected page result in a redirect to login with the original URL preserved.
- **SC-006**: Email and username availability feedback appears within 1 second of the user pausing input, before form submission.
- **SC-007**: Sessions persist across browser restarts for up to 7 days without requiring the user to sign in again.
- **SC-008**: Authentication-related unit test coverage is at or above 70%.

---

## Assumptions

- Login rate limiting and brute-force protection are **out of scope for MVP** and deferred to Phase 2.

- Email verification (sending a confirmation email) is **out of scope for MVP**. Users can sign in immediately after registration.
- Password reset ("Forgot password") is **out of scope for this spec** and deferred to Phase 2.
- Admin accounts are created via database seed data; there is no self-registration path to the admin role.
- Avatar images are sourced from the OAuth2 provider profile photo for SSO users; local users have no avatar in MVP.
- OAuth2 application credentials (client ID, client secret) for Google and GitHub are pre-configured in the service environment.
- The BFF layer (Next.js API Routes) manages cookie lifecycle; the authentication service returns credentials in response bodies and the BFF sets the cookies.
- The platform targets web browsers only for MVP; mobile app authentication is not in scope.

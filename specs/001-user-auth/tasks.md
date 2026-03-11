# Tasks: User Authentication System

**Input**: Design documents from `/specs/001-user-auth/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅

**Tests**: Included — spec SC-008 requires ≥70% unit test coverage; constitution §V mandates three-layer tests.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

**Already built** (no tasks needed): login endpoint, signup endpoint, refresh-token, logout, `/users/me`, AuthGuard, Zustand store, useAuth hook, Axios interceptor, login/register UI, all existing BFF routes. See `research.md` for full audit.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete sibling tasks)
- **[Story]**: User story this task belongs to (US1–US6 from spec.md)
- Paths are relative to the sibling service repos (`../gl-auth-api/`, `../glstudy-frontend/`)

---

## Phase 1: Setup (Infrastructure Verification)

**Purpose**: Confirm the existing running infrastructure is ready for new code. No new repos or frameworks to initialize — all three services exist.

- [X] T001 Verify `data.sql` seeds `ROLE_LEARNER` and `ROLE_ADMIN` rows in `glauth.roles` on startup — confirm in `gl-auth-api/src/main/resources/data.sql`
- [X] T002 [P] Confirm `AUTH_API_URL` default in `glstudy-frontend/src/app/api/auth/login/route.ts` matches the actual `gl-auth-api` port (check `gl-auth-api/src/main/resources/application.yml` `server.port`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: New DTOs, repository query methods, and the FR-020 Redis status check that all user story phases depend on.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T003 [P] Add `CheckEmailRequest` DTO with `@Email @NotBlank` validation in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/dto/request/CheckEmailRequest.java`
- [X] T004 [P] Add `CheckUsernameRequest` DTO with `@NotBlank @Size(min=3,max=50)` validation in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/dto/request/CheckUsernameRequest.java`
- [X] T005 [P] Add `SetPasswordRequest` DTO with `@NotBlank` userId and password pattern validation in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/dto/request/SetPasswordRequest.java`
- [X] T006 [P] Add `AvailabilityResponse` DTO with `boolean exists` field — implemented as `CheckExistResponse` with `boolean exist` field
- [X] T007 Update `SignupRequest.username` to `@NotBlank` (was optional/derived) in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/dto/request/SignupRequest.java`
- [X] T008 Add `existsByEmail(String email)` and `existsByUsername(String username)` query methods to `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/repository/UserRepository.java`
- [X] T009 Add `user_status:{userId}` Redis check in `TokenServiceImpl.getTokenInfo()`: read key → if `"0"` throw `ApiException.ErrUnauthorized()`, else continue — in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/TokenServiceImpl.java`

**Checkpoint**: DTOs, repository methods, and FR-020 check in place — user story phases can now begin.

---

## Phase 3: User Story 1 – Email Registration (Priority: P1) 🎯 MVP

**Goal**: New users can register with explicit username, display name, email, and password. Real-time availability checks prevent duplicate submissions.

**Independent Test**: Open `/login` → Register tab → fill in username, display name, email, strong password → submit → lands on `/dashboard` as authenticated user. Username/email availability feedback appears within 1 second of pausing input.

### Backend — check-email & check-username endpoints

- [X] T010 [P] [US1] Implement `UserService.checkEmail(CheckEmailRequest)` returning `AvailabilityResponse` using `UserRepository.existsByEmail` in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/UserServiceImpl.java`
- [X] T011 [P] [US1] Implement `UserService.checkUsername(CheckUsernameRequest)` returning `AvailabilityResponse` using `UserRepository.existsByUsername` in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/UserServiceImpl.java`
- [X] T012 [US1] Add `POST /v1/users/check-email` and `POST /v1/users/check-username` endpoints to `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/controller/UserController.java` (no auth required, use `@Valid @RequestBody`)
- [X] T013 [P] [US1] Write `MockMvc` controller tests for check-email (valid email → exists/not-exists, blank email → 400) in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/controller/UserControllerTest.java`
- [X] T014 [P] [US1] Write Mockito service tests for `checkEmail` and `checkUsername` in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/service/UserServiceTest.java`

### Frontend — register form + BFF routes

- [X] T015 [P] [US1] Add BFF route `POST /api/users/check-email` (proxy to `gl-auth-api`, no auth) in `glstudy-frontend/src/app/api/users/check-email/route.ts`
- [X] T016 [P] [US1] Add BFF route `POST /api/users/check-username` (proxy to `gl-auth-api`, no auth) in `glstudy-frontend/src/app/api/users/check-username/route.ts`
- [X] T017 [US1] Add explicit `username` field to register form with debounced `check-username` availability check and field-level error in `glstudy-frontend/src/app/(auth)/login/page.tsx`
- [X] T018 [US1] Wire debounced `check-email` availability call to email field on register form in `glstudy-frontend/src/app/(auth)/login/page.tsx`
- [X] T019 [US1] Update BFF `POST /api/auth/register` to forward explicit `username` field to `gl-auth-api` in `glstudy-frontend/src/app/api/auth/register/route.ts`

**Checkpoint**: User Story 1 complete — email registration end-to-end functional with real-time availability checks.

---

## Phase 4: User Story 2 – Email/Password Login (Priority: P2)

**Goal**: Returning users sign in with email + password. Inactive accounts are rejected with a clear message.

**Independent Test**: Enter correct credentials → redirected to dashboard. Enter wrong password → generic error shown. Use an inactive account → specific "account inactive" message displayed.

### Backend — inactive account enforcement (FR-019)

- [X] T020 [US2] Add `user.status == 0` check in `AuthServiceImpl.login()` before password verification: throw `ApiException.ErrUnauthorized().message("Account is inactive. Contact support.").build()` in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/AuthenticationServiceImpl.java`
- [X] T021 [US2] Write `user_status:{userId}` Redis key on every successful login (`"1"`) and on inactive login attempt (`"0"`) in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/AuthenticationServiceImpl.java`
- [X] T022 [P] [US2] Write `MockMvc` test for `POST /v1/auth/login` with inactive account → 401 in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/controller/AuthenticationControllerTest.java`
- [X] T023 [P] [US2] Write Mockito service test for inactive account login in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/service/AuthServiceTest.java`

### Frontend — inactive account error display

- [X] T024 [US2] Handle `?error=account_inactive` query param on login page: show "Your account has been deactivated. Contact support." in `glstudy-frontend/src/app/(auth)/login/page.tsx`
- [X] T025 [US2] In Axios 401 interceptor: detect `ACCOUNT_INACTIVE` error code, clear cookies (call `/api/auth/logout`), redirect to `/login?error=account_inactive` without retry in `glstudy-frontend/src/lib/api-client.ts`

**Checkpoint**: User Story 2 complete — local login works end-to-end, inactive accounts blocked everywhere.

---

## Phase 5: User Story 3 – SSO Login with Google or GitHub (Priority: P3)

**Goal**: Users can sign in via Google or GitHub OAuth2. New SSO users choose a username; existing email conflicts show the confirm-link screen.

**Independent Test**: Click "Sign in with Google" → complete OAuth2 consent → either signed in, shown username form, or shown confirm-link modal — without entering a password.

### Frontend — OAuth2 initiation

- [X] T026 [P] [US3] Add Google OAuth2 redirect handler: build consent URL (`accounts.google.com/o/oauth2/v2/auth`) with `client_id`, `redirect_uri`, `scope=openid email profile`, `state` and redirect browser in `glstudy-frontend/src/app/(auth)/login/page.tsx`
- [X] T027 [P] [US3] Add GitHub OAuth2 redirect handler: build consent URL (`github.com/login/oauth/authorize`) with `client_id`, `redirect_uri`, `scope=user:email`, `state` and redirect browser in `glstudy-frontend/src/app/(auth)/login/page.tsx`

### Frontend — OAuth2 callback BFF

- [X] T028 [US3] Add OAuth2 callback BFF route: receive `code`+`state`, call `POST /v1/auth/login {provider, authorizationCode}`, handle all 3 backend scenarios (set cookies+redirect, sso-register redirect, confirm-link redirect) and error cases (sso_failed, account_inactive) in `glstudy-frontend/src/app/api/auth/oauth2/[provider]/callback/route.ts`

### Frontend — SSO new-user registration page

- [X] T029 [US3] Create SSO register page: read `token` from URL param, show username field with `check-username` validation, on submit call `POST /api/auth/register {username, fullName, token, social:true}` in `glstudy-frontend/src/app/(auth)/sso-register/page.tsx`

### Frontend — confirm-link flow

- [X] T030 [P] [US3] Add confirm-link BFF route `GET /api/auth/confirm-link`: forward `confirmCode` to `GET /v1/auth/confirm-link`, set cookies on success, return user info in `glstudy-frontend/src/app/api/auth/confirm-link/route.ts`
- [X] T031 [US3] Create confirm-link page: read `confirmCode` from URL param, show explanation of which account will be linked, "Yes, link my account" button calls BFF confirm-link route, then redirects to dashboard in `glstudy-frontend/src/app/(auth)/confirm-link/page.tsx`

**Checkpoint**: User Story 3 complete — full SSO flow works for Google and GitHub (new user, returning user, existing email).

---

## Phase 6: User Story 4 – Session Persistence & Transparent Renewal (Priority: P4)

**Goal**: Users stay logged in across browser restarts for up to 7 days. Token renewal is silent — no login prompt when access token expires.

**Independent Test**: Log in → let access token expire (or set short TTL) → navigate to protected page → remains logged in. After 7 days of inactivity → redirected to `/login`.

*Note: Core session storage (Redis refresh token, httpOnly cookies) and Axios renewal interceptor are already built. Tasks in this phase are verifications and edge-case handling.*

- [X] T032 [US4] Verify `AuthGuard` captures original URL and appends it as `?from=<url>` when redirecting to `/login`, then redirects back after login in `glstudy-frontend/src/components/auth/AuthGuard.tsx` (or the equivalent layout guard file)
- [X] T033 [US4] Write Mockito test for `TokenServiceImpl.getTokenInfo()` with `user_status:{userId}="0"` → `UNAUTHORIZED` thrown in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/service/TokenServiceTest.java`

**Checkpoint**: User Story 4 complete — sessions persist and renew silently; inactive accounts are blocked on every request.

---

## Phase 7: User Story 5 – Logout (Priority: P5)

**Goal**: Logout immediately invalidates the current device's session server-side; credentials cleared from browser; protected pages redirect to login.

**Independent Test**: Log in → click Logout → lands on landing page → navigate to `/dashboard` → redirected to `/login`.

*Note: Logout BFF route, blacklist mechanism, and cookie clearing are fully built. Tasks in this phase are integration testing only.*

- [X] T034 [US5] Write Spring Boot integration test: login → logout → present blacklisted access token → expect 401 in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/integration/AuthIntegrationTest.java`
- [X] T035 [US5] Write Spring Boot integration test: login → logout → present refresh token → expect 401 (no new session) in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/integration/AuthIntegrationTest.java`

**Checkpoint**: User Story 5 complete — logout is verified to invalidate both tokens server-side.

---

## Phase 8: User Story 6 – Profile Viewing & Editing (Priority: P6)

**Goal**: Logged-in users see their profile (username, display name, email, linked providers). They can update display name. SSO-only users can set a local password.

**Independent Test**: Navigate to `/profile` → see username, display name, email, linked providers list. Change display name → submit → nav bar shows new name. As SSO-only user, set password → can now log in with email + password.

### Backend — providers field + set-password

- [X] T036 [P] [US6] Add `providers: List<String>` field to `UserResponse` DTO, populate from `UserAuthProvider` entities in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/dto/response/UserResponse.java`
- [X] T037 [P] [US6] Update `UserMapper` to map `UserAuthProvider` list → provider name strings in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/mapper/UserMapper.java`
- [X] T038 [US6] Implement `UserService.setPassword(SetPasswordRequest, String callerUserId)`: validate caller matches userId, validate user has no existing password, bcrypt-hash and save in `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/UserServiceImpl.java`
- [X] T039 [US6] Add `POST /v1/users/set-password` endpoint (requires `Authorization` header, validates via `TokenService`) to `gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/controller/UserController.java`
- [X] T040 [P] [US6] Write `MockMvc` tests for set-password: success, password already set (400), caller mismatch (403), missing auth (401) in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/controller/UserControllerTest.java`
- [X] T041 [P] [US6] Write Mockito service tests for `setPassword` in `gl-auth-api/src/test/java/vn/glvideo/gl_auth_api/service/UserServiceTest.java`

### Frontend — profile page, set-password form, linked providers

- [X] T042 [P] [US6] Add BFF route `POST /api/users/set-password`: read `access_token` cookie, extract userId, forward `{userId, password}` to backend in `glstudy-frontend/src/app/api/users/set-password/route.ts`
- [X] T043 [P] [US6] Create `LinkedProviders` component: display list of provider names from user profile with icons in `glstudy-frontend/src/components/profile/LinkedProviders.tsx`
- [X] T044 [P] [US6] Create `SetPasswordForm` component: password field with strength validation (Zod), submit calls `POST /api/users/set-password`, show success/error feedback in `glstudy-frontend/src/components/profile/SetPasswordForm.tsx`
- [X] T045 [US6] Update profile page to include `LinkedProviders` (always visible) and `SetPasswordForm` (only when user has no local password) in `glstudy-frontend/src/app/(main)/profile/page.tsx`

**Checkpoint**: User Story 6 complete — profile displays linked providers; SSO users can set a password.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [X] T046 [P] Update `docs/gl-auth-api/api-specification.md` with the three new endpoints (check-email, check-username, set-password) and the inactive account error row for login
- [X] T047 [P] Update `docs/gl-auth-api/schema.md` with `idx_user_auth_providers_user_id` index and the UNIQUE constraint on `(user_id, provider)` in `user_auth_providers`
- [X] T048 [P] Update `docs/WORKLOG.md` — check off completed backend and frontend auth items in the Work Items section
- [ ] T049 Run full quickstart.md validation: execute each section of `specs/001-user-auth/quickstart.md` end-to-end and verify all flows pass

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)
  └→ Phase 2 (Foundational)  ← BLOCKS all user stories
       ├→ Phase 3 (US1 – Email Registration)   🎯 MVP start
       ├→ Phase 4 (US2 – Email Login)
       ├→ Phase 5 (US3 – SSO Login)
       ├→ Phase 6 (US4 – Session Persistence)
       ├→ Phase 7 (US5 – Logout)
       └→ Phase 8 (US6 – Profile)
            └→ Phase 9 (Polish)
```

### User Story Dependencies

- **US1 (P1)**: After Phase 2. No dependency on other stories. — **Start here for MVP**
- **US2 (P2)**: After Phase 2. No dependency on US1.
- **US3 (P3)**: After Phase 2. Depends on US1 completing the register BFF update (T019) for SSO new-user registration.
- **US4 (P4)**: After Phase 2. Independent of other stories.
- **US5 (P5)**: After Phase 2. Independent; integration tests reuse login from US2.
- **US6 (P6)**: After Phase 2. No dependency on other stories except US1 (set-password requires users exist).

### Within Each User Story

- Models/DTOs first (T003–T009 in Foundational)
- Services before controllers
- Backend contracts before frontend BFF routes
- BFF routes before UI wiring

### Parallel Opportunities

**Phase 2** — all T003–T006 (DTO creation) run in parallel:
```
T003 CheckEmailRequest  ↘
T004 CheckUsernameRequest → T007 UserRepository → T008 UserService → T009 TokenService
T005 SetPasswordRequest  ↗
T006 AvailabilityResponse ↗
```

**Phase 3** — backend and frontend can run in parallel once T008 is complete:
```
T010 checkEmail service  ↘
T011 checkUsername service → T012 UserController endpoints
T015 BFF check-email route  ↘
T016 BFF check-username route → T017+T018 form wiring → T019 register BFF update
T013 Controller tests  [P with T010+T011]
T014 Service tests     [P with T010+T011]
```

**Phase 5** — SSO initiation handlers run in parallel:
```
T026 Google handler  ↘
T027 GitHub handler   → T028 Callback BFF → T029 SSO register page
                                          → T030 Confirm-link BFF → T031 Confirm-link page
```

**Phase 8** — backend and frontend run in parallel:
```
T036 UserResponse providers  ↘
T037 UserMapper              → T038+T039 setPassword service+controller
T042 BFF set-password  [P]
T043 LinkedProviders    [P]
T044 SetPasswordForm    [P]
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001–T002)
2. Complete Phase 2: Foundational (T003–T009)
3. Complete Phase 3: User Story 1 (T010–T019)
4. **STOP and VALIDATE**: Email registration + availability checks working end-to-end
5. Deploy/demo — platform is usable for new learner onboarding

### Incremental Delivery

1. Phases 1–2 → Foundation ready
2. Phase 3 (US1) → Email registration → **Demo**
3. Phase 4 (US2) → Email login + inactive enforcement → **Demo**
4. Phase 5 (US3) → SSO login → **Demo**
5. Phases 6–7 (US4–US5) → Session + logout verification → **Demo**
6. Phase 8 (US6) → Profile + set-password → **Full feature complete**
7. Phase 9 → Polish → **PR ready**

### Parallel Team Strategy

After Phase 2 completes:
- **Backend developer**: US1 backend (T010–T014) + US2 backend (T020–T023) + US6 backend (T036–T041)
- **Frontend developer**: US1 frontend (T015–T019) + US2 frontend (T024–T025) + US3 frontend (T026–T031)
- **Both**: Phase 9 polish tasks in parallel

---

## Notes

- [P] tasks operate on different files — no merge conflicts when running in parallel
- Each [Story] label maps to the spec.md user story for acceptance scenario traceability
- Write tests BEFORE implementing (TDD) — ensure tests fail first for T013, T014, T022, T023, T033–T035, T040, T041
- Stop at each **Checkpoint** to validate the story independently before proceeding
- Update `docs/WORKLOG.md` task checkboxes as items complete
- All backend changes must be reflected in `docs/gl-auth-api/api-specification.md` and `docs/gl-auth-api/schema.md` (constitution §IV)

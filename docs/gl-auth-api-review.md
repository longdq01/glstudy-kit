# gl-auth-api — Code Review

> Review of the full backend codebase: controllers, services, security config, entities, exception handling.

---

## Overall Assessment

The codebase is **well-structured** with clean separation of concerns. Good use of Spring Security custom providers (Local + OAuth2), DTO pattern with MapStruct, and the Redis-backed token strategy is solid. However, there are a few bugs and security concerns to address.

---

## 🔴 Critical Bug

### 1. Refresh token cache uses MICROSECONDS instead of MILLISECONDS

**File:** [TokenServiceImpl.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/TokenServiceImpl.java#L133-L138)

```java
// Line 137
redisTemplate.opsForValue().set(redisKey, JsonUtils.serialize(userDTO),
        Constant.REFRESH_TOKEN_EXPIRATION, TimeUnit.MICROSECONDS);
//                                         ^^^^^^^^^^^^^^^^^^^^
//                                         Should be MILLISECONDS!
```

`REFRESH_TOKEN_EXPIRATION` is `7 * 24 * 60 * 60 * 1000L` (= 604,800,000 ms). With `TimeUnit.MICROSECONDS`, this becomes ~604 seconds ≈ **10 minutes** instead of 7 days. Refresh tokens will expire far too quickly!

**Fix:** Change to `TimeUnit.MILLISECONDS`.

---

## 🟠 High Priority

### 2. Access token expiry is 10 hours, not 15 minutes

**File:** [Constant.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/constant/Constant.java#L5)

```java
public static final Long ACCESS_TOKEN_EXPIRATION = 600 * 60 * 1000L; // comment says "1 hour"
```

- `600 * 60 * 1000` = 36,000,000 ms = **10 hours**
- Comment says "1 hour" — neither matches the docs (15 minutes)
- Per `architecture.md` and `gl-auth-api/README.md`: access tokens should be **15 minutes**

**Fix:**
```java
public static final Long ACCESS_TOKEN_EXPIRATION = 15 * 60 * 1000L; // 15 minutes
```

### 3. Sensitive data logged in plaintext

**File:** [AuthenticationServiceImpl.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/AuthenticationServiceImpl.java#L122)

```java
log.info("Authenticating with oauth provider with request: {}", JsonUtils.serialize(authRequest));
```

This logs the full `AuthRequest` including `authorizationCode` (and `password` for local login at line 175). Authorization codes are single-use secrets.

**File:** [UserServiceImpl.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/UserServiceImpl.java#L46)

```java
log.info("Signup request received: {}", request); // may include password
```

**Fix:** Use `@ToString.Exclude` on sensitive fields, or log only non-sensitive identifiers.

### 4. `UsernameNotFoundException` abused for flow control

**File:** [OAuth2AuthenticationProvider.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/config/security/OAuth2AuthenticationProvider.java#L70)

```java
throw new UsernameNotFoundException(token); // token as exception message!
```

Then caught in `AuthenticationServiceImpl` line 129:
```java
catch (UsernameNotFoundException e) {
    String token = e.getMessage(); // extracts token from exception message
```

This is an anti-pattern — using exceptions for normal flow control, and encoding business data in error messages. It also means Spring Security's default `UsernameNotFoundException` handling could interfere.

**Better approach:** Return a custom result object from the provider (e.g., `OAuth2AuthenticationResult` with a `needsRegistration` flag + token).

---

## 🟡 Medium Priority

### 5. Missing `@Transactional` on write operations

**Files:** `UserServiceImpl.signupWithPassword()`, `UserServiceImpl.signupOAuthProvider()`, `AuthenticationServiceImpl.confirmLinkProvider()`

These methods create/update database records but have no `@Transactional`. If an error occurs mid-operation (e.g., after saving user but before saving provider), the database could be left in an inconsistent state.

**Fix:** Add `@Transactional` to methods that modify the database.

### 6. `updated_at` uses `@CreatedDate` instead of `@LastModifiedDate`

**File:** [User.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/model/entity/User.java#L70-L72)

```java
@CreatedDate          // ❌ Should be @LastModifiedDate
@Column(name = "updated_at")
private Timestamp updatedAt;
```

This means `updated_at` is only set on creation and never updated. Use `@LastModifiedDate` instead.

### 7. CORS is wide open (`*`)

**File:** [SecurityConfiguration.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/config/security/SecurityConfiguration.java#L102)

```java
config.setAllowedOriginPatterns(Collections.singletonList("*"));
config.setAllowCredentials(true);
```

`allowCredentials=true` + `allowedOrigins=*` is a security risk. In production, restrict to known origins (e.g., `http://localhost:3000`).

### 8. `logout` endpoint is in `permitAll()` but requires `Authorization` header

**File:** [SecurityConfiguration.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/config/security/SecurityConfiguration.java#L55)

The `/v1/auth/logout` path is in the `permitAll()` list, but `AuthenticationController.login()` (the logout method — also a naming issue, see #10 below) requires a `@RequestHeader(Authorization)` header. This means:
- An unauthenticated request to `/logout` without a header will cause a 400 (missing header) rather than a 401
- The JWT filter won't reject revoked tokens on this path since it's `permitAll`

### 9. `setLastLogin` makes an extra DB round-trip

**File:** [AuthenticationServiceImpl.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/service/impl/AuthenticationServiceImpl.java#L202-L212)

```java
private void setLastLogin(String userId) {
    Optional<User> userOpt = userRepository.findById(userId);
    // ...
    user.setLastLogin(DateUtils.getCurrentDateTime());
    userRepository.save(user);
}
```

The user entity was already loaded during authentication. Instead of re-fetching, pass the entity or use an `@Modifying @Query` update.

---

## 🔵 Low Priority / Style

### 10. Logout method named `login`

**File:** [AuthenticationController.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/controller/AuthenticationController.java#L42)

```java
@PostMapping("/logout")
public ResponseEntity<BaseResponse> login(...) { // ❌ method named "login"
```

Should be `logout()`.

### 11. JwtAuthenticationFilter sets empty authorities

**File:** [JwtAuthenticationFilter.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/config/security/JwtAuthenticationFilter.java#L46)

```java
.authorities(Collections.emptyList())
```

Roles/permissions from the JWT are never extracted. This means role-based authorization (`@PreAuthorize("hasRole('ADMIN')")`) won't work even though the RBAC model exists in the database.

### 12. No `@Valid` on signup request

**File:** [UserController.java](file:///d:/Long/workspace/GLStudy/gl-auth-api/src/main/java/vn/glvideo/gl_auth_api/controller/UserController.java#L41)

```java
@PostMapping("/signup")
public ResponseEntity<BaseResponse> signup(@RequestBody SignupRequest request) {
//                                          ^^^^^^ missing @Valid
```

Bean validation annotations on `SignupRequest` won't trigger without `@Valid`.

---

## Summary Table

| # | Severity | Issue | File |
|---|---|---|---|
| 1 | 🔴 Critical | Refresh token TTL uses MICROSECONDS | `TokenServiceImpl` |
| 2 | 🟠 High | Access token = 10h, docs say 15min | `Constant` |
| 3 | 🟠 High | Passwords/codes logged in plaintext | `AuthenticationServiceImpl`, `UserServiceImpl` |
| 4 | 🟠 High | Exception for flow control (UsernameNotFoundException) | `OAuth2AuthenticationProvider` |
| 5 | 🟡 Medium | Missing `@Transactional` on writes | `UserServiceImpl`, `AuthenticationServiceImpl` |
| 6 | 🟡 Medium | `updated_at` uses `@CreatedDate` | `User.java` |
| 7 | 🟡 Medium | CORS allows all origins with credentials | `SecurityConfiguration` |
| 8 | 🟡 Medium | Logout in `permitAll` + requires auth header | `SecurityConfiguration` |
| 9 | 🟡 Medium | Extra DB round-trip in `setLastLogin` | `AuthenticationServiceImpl` |
| 10 | 🔵 Low | Logout method named `login` | `AuthenticationController` |
| 11 | 🔵 Low | JWT filter ignores roles | `JwtAuthenticationFilter` |
| 12 | 🔵 Low | Missing `@Valid` on signup | `UserController` |

# Authentication Testing Plan

Comprehensive manual testing plan covering all authentication flows, edge cases, and error scenarios.

---

## 1. Tenant Discovery

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 1.1 | Discover tenants with valid email (multiple tenants) | 1. Open app fresh (no stored tenant domain) 2. Enter a valid email associated with multiple tenants 3. Tap "Continue" | Loading spinner appears, then navigates to Tenant Selection screen showing all associated tenants |
| 1.2 | Discover tenants with valid email (single tenant) | 1. Open app fresh 2. Enter email associated with exactly one tenant 3. Tap "Continue" | Tenant is auto-selected, tenant domain and name are saved to MMKV and Redux, navigates directly to Sign In screen (skips selection) |
| 1.3 | Discover tenants with unregistered email | 1. Open app fresh 2. Enter an email not associated with any tenant 3. Tap "Continue" | Error message displayed (e.g., "No workspaces found for this email"), stays on discovery screen |
| 1.4 | Empty email field | 1. Open app fresh 2. Leave email field empty 3. Tap "Continue" | "Continue" button is disabled or validation error "Email is required" is shown |
| 1.5 | Invalid email format | 1. Open app fresh 2. Enter "notanemail" 3. Tap "Continue" | Validation error "Please enter a valid email address" is shown |
| 1.6 | Network error during discovery | 1. Turn off network/airplane mode 2. Enter valid email 3. Tap "Continue" | User-friendly error message shown (e.g., "Network error, please try again"), stays on discovery screen |
| 1.7 | Returning user with stored tenant domain | 1. Log out (tenant domain persists) 2. Kill and reopen app | App skips Tenant Discovery, shows Sign In screen directly with tenant name displayed |
| 1.8 | Switch workspace from Sign In | 1. Be on Sign In screen with a stored tenant 2. Tap the workspace/tenant switcher | Navigates back to Tenant Discovery screen to enter a different email |

---

## 2. Tenant Selection

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 2.1 | Select a tenant from list | 1. Discover tenants for email with multiple results 2. Tap on one tenant | Tenant domain and name saved to MMKV and Redux, navigates to Sign In screen with selected tenant displayed |
| 2.2 | Back navigation from selection | 1. Be on Tenant Selection screen 2. Tap back button | Returns to Tenant Discovery screen, previous email is retained |
| 2.3 | Tenant list rendering | 1. Discover tenants for email with multiple results | All tenants are displayed with correct names, list is scrollable if many tenants |

---

## 3. Sign In (Login)

### 3.1 Form Validation

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.1.1 | Empty email and password | 1. Open Sign In screen 2. Tap "Sign In" without entering anything | Validation errors shown for both fields: "Email is required", "Password is required" |
| 3.1.2 | Invalid email format | 1. Enter "invalid-email" in email field 2. Enter valid password 3. Tap "Sign In" | Validation error: "Please enter a valid email address" |
| 3.1.3 | Password too short | 1. Enter valid email 2. Enter password shorter than 6 characters 3. Tap "Sign In" | Validation error: "Password must be at least 6 characters" |
| 3.1.4 | Valid email and password format | 1. Enter valid email format 2. Enter password >= 6 characters | No validation errors, form is ready to submit |
| 3.1.5 | Email field pre-filled | 1. Log in successfully 2. Log out 3. Return to Sign In | Email field is pre-filled with previously used email (from SecureStore `user_email`) |

### 3.2 Successful Login

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.2.1 | Login with valid credentials | 1. Enter correct email and password 2. Tap "Sign In" | Loading spinner on button, API call succeeds, token saved to SecureStore, `is_logged_in` flag set in MMKV, user/tenant/permissions dispatched to Redux, navigates to Main Stack (Home screen) |
| 3.2.2 | Redux state after login | 1. Login successfully 2. Inspect Redux state | `isAuthenticated: true`, `user` populated with id/name/email, `tenant` populated with id/name, `permissions` array populated, `accessToken` set, `error: null` |
| 3.2.3 | SecureStore after login | 1. Login successfully | `auth_access_token` contains JWT, `user_email` contains login email |
| 3.2.4 | MMKV after login | 1. Login successfully | `is_logged_in` is `true`, `tenant_domain` set, `tenant_name` set |
| 3.2.5 | Device name sent in login request | 1. Login successfully 2. Inspect network request | Request body includes `device_name` in format "Device Model - OS Version" |

### 3.3 Failed Login

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 3.3.1 | Wrong password | 1. Enter valid email 2. Enter incorrect password 3. Tap "Sign In" | API returns 401, error message displayed on screen (from `error.response.data.message`), no tokens stored, `isAuthenticated` remains `false` |
| 3.3.2 | Non-existent email | 1. Enter email not registered with the tenant 2. Enter any password 3. Tap "Sign In" | API returns error, appropriate error message shown, state unchanged |
| 3.3.3 | Network error during login | 1. Turn off network 2. Enter valid credentials 3. Tap "Sign In" | Error message shown (e.g., "Network error"), loading state clears, user can retry |
| 3.3.4 | Server error (500) | 1. Trigger server error scenario 2. Attempt login | Error message shown (generic), loading state clears, no partial state saved |
| 3.3.5 | Clear error on retry | 1. Login with wrong credentials (error shows) 2. Modify credentials 3. Tap "Sign In" again | Previous error is cleared before new attempt |
| 3.3.6 | Rapid double-tap on Sign In | 1. Enter valid credentials 2. Quickly double-tap "Sign In" | Only one API call is made (button disabled during loading), no duplicate requests |

---

## 4. Session Restore (Cold Start)

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 4.1 | Restore with valid session | 1. Login successfully 2. Kill app completely 3. Reopen app | Splash/loading shown briefly, `is_logged_in` flag checked (true), token retrieved from SecureStore, refresh endpoint called to get fresh token + user data, Redux populated with fresh data, navigates directly to Main Stack — no Sign In screen shown |
| 4.2 | Restore when `is_logged_in` is false | 1. Fresh install or cleared data 2. Open app | `is_logged_in` check returns false, session restore skipped entirely, navigates to Auth Stack (Tenant Discovery or Sign In) |
| 4.3 | Restore with no stored token | 1. Somehow `is_logged_in` is true but token was cleared 2. Open app | Token retrieval returns null, all auth data cleared (SecureStore + MMKV), navigates to Auth Stack |
| 4.4 | Restore when refresh endpoint returns 401 | 1. Login successfully 2. Wait for token to expire (or manually expire) 3. Kill and reopen app | Refresh call fails with 401 (invalid/expired token), all auth data cleared, navigates to Sign In screen |
| 4.5 | Restore when refresh endpoint has network error | 1. Login successfully 2. Kill app 3. Turn off network 4. Reopen app | Refresh call fails with network error, retries up to 2 times, if all fail: does not clear auth data (token might still be valid), shows appropriate error or retry option |
| 4.6 | No duplicate Redux dispatches | 1. Login and kill app 2. Reopen app | `restoreSessionSuccess` is dispatched exactly once (dispatch guard via refs prevents duplicates even if React Query re-renders) |
| 4.7 | Loading state during restore | 1. Kill and reopen app with valid session | App shows loading/splash state while restore is in progress, no flash of Sign In screen before Main Stack appears |
| 4.8 | `sessionRestoreSettled` gate | 1. Open app | RootNavigator waits for `sessionRestoreSettled` to be true before rendering Auth or Main stack — prevents navigation flicker |

---

## 5. Token Refresh (Automatic via Interceptor)

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 5.1 | Automatic refresh on 401 | 1. Login successfully 2. Wait for token to expire 3. Make any API call | API returns 401, interceptor catches it, refresh endpoint called with current token, new token received, SecureStore updated, Redux `accessToken` updated, original request auto-retried with new token — user sees no interruption |
| 5.2 | Concurrent 401 requests queued | 1. Have expired token 2. Trigger multiple simultaneous API calls | First 401 triggers refresh, subsequent 401s are queued (not triggering separate refreshes), after refresh succeeds all queued requests retry with new token |
| 5.3 | Refresh uses separate axios instance | 1. Token expires 2. Trigger API call | Refresh request uses raw axios (not the main `apiClient`), so it does NOT attach the expired token via request interceptor |
| 5.4 | Refresh endpoint itself fails | 1. Token expires 2. Refresh token is also invalid 3. Make API call | Refresh call fails, all auth data cleared (SecureStore + MMKV + Redux), user force-logged out, navigates to Sign In screen |
| 5.5 | No infinite refresh loop | 1. Refresh endpoint returns 401 | Interceptor does NOT retry refresh on the refresh endpoint itself (checks URL to prevent loop), proceeds to logout |
| 5.6 | Token refresh with tenant domain fallback | 1. Token expires before Redux is hydrated (edge case) 2. Refresh triggered | Refresh API uses fallback: `Redux tenantDomain ?? MMKV tenantDomain` to construct the correct base URL |
| 5.7 | Queued requests get correct token | 1. Have 5 pending API calls during refresh | After refresh succeeds, all 5 queued requests have their Authorization header updated to the new token before retry |

---

## 6. Logout

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 6.1 | Successful logout | 1. Be logged in 2. Tap logout button | API `POST /v1/auth/logout` called, SecureStore tokens cleared (email cleared, but biometric preference preserved), MMKV `is_logged_in` cleared, token refresh manager state reset, React Query cache cleared, Redux `logoutSuccess` dispatched (`isAuthenticated: false`), navigates to Auth Stack |
| 6.2 | Logout API fails (network error) | 1. Be logged in 2. Turn off network 3. Tap logout | API call fails, but local cleanup continues anyway (resilient logout), SecureStore cleared, MMKV cleared, Redux cleared, user logged out locally even though server didn't invalidate token |
| 6.3 | Tenant domain persists after logout | 1. Be logged in to tenant "acme" 2. Logout | `tenant_domain` and `tenant_name` remain in MMKV and Redux, next app open goes directly to Sign In (not Tenant Discovery) |
| 6.4 | Email persists after logout | 1. Login with email "user@test.com" 2. Logout 3. Navigate to Sign In | Email field pre-filled with "user@test.com" |
| 6.5 | Redux state after logout | 1. Logout | `isAuthenticated: false`, `user: null`, `tenant: null` (but `tenantDomain` and `tenantName` persist), `permissions: []`, `accessToken: null`, `error: null` |
| 6.6 | Cannot access Main Stack after logout | 1. Logout | Navigation driven by `isAuthenticated`, Main Stack routes are not accessible, deep links to main screens redirect to Auth Stack |
| 6.7 | Token refresh state reset on logout | 1. Have a pending token refresh 2. Logout | `TokenRefreshManager` state is reset via `resetTokenRefreshState()`, no stale refresh attempts after re-login |

---

## 7. Forgot Password

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 7.1 | Navigate to Forgot Password | 1. Be on Sign In screen 2. Tap "Forgot Password?" link | Navigates to Forgot Password screen |
| 7.2 | Submit valid email | 1. Enter valid registered email 2. Tap "Submit" | API called, success ResultModal shown with message like "Password reset email sent", user can dismiss modal |
| 7.3 | Submit invalid email format | 1. Enter invalid email 2. Tap "Submit" | Validation error: "Please enter a valid email address" |
| 7.4 | Submit empty email | 1. Leave email empty 2. Tap "Submit" | Validation error: "Email is required" |
| 7.5 | Network error on submit | 1. Turn off network 2. Enter valid email 3. Tap "Submit" | Error ResultModal shown with appropriate message |
| 7.6 | Back navigation | 1. Be on Forgot Password screen 2. Tap back | Returns to Sign In screen |

---

## 8. Storage Integrity

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 8.1 | SecureStore write failure during login | 1. Simulate SecureStore write error during `setAccessToken` | Login mutation throws, no partial state saved, MMKV not written to, Redux not updated, error shown to user |
| 8.2 | MMKV write failure during login (rollback) | 1. Login successfully writes token to SecureStore 2. MMKV `setLoggedIn` fails | Rollback: token cleared from SecureStore via `clearTokens()`, error thrown, user sees login failure |
| 8.3 | MMKV corrupted data on read | 1. Manually corrupt MMKV stored JSON 2. Read user/tenant data | `safeJsonParse()` catches error, logs it, clears corrupted key, returns default value — app does not crash |
| 8.4 | SecureStore read returns null | 1. SecureStore `getAccessToken()` fails or returns null | Function returns null gracefully, caller handles null case (e.g., session restore skips) |
| 8.5 | Dual storage consistency after login | 1. Login successfully 2. Check both stores | SecureStore has token, MMKV has `is_logged_in: true` + tenant info, Redux matches both — all three layers in sync |
| 8.6 | Dual storage consistency after logout | 1. Logout 2. Check both stores | SecureStore token cleared, MMKV `is_logged_in` cleared (but tenant domain/name persist), Redux auth state reset |
| 8.7 | `clearAuthState()` cleanup function | 1. Call `clearAuthState()` from `authCleanup.ts` | All SecureStore auth keys cleared, all MMKV auth keys cleared, token refresh manager reset — single point of cleanup |

---

## 9. API Client & Interceptors

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 9.1 | Request interceptor attaches token | 1. Be logged in 2. Make any API call | Request has `Authorization: Bearer <token>` header, token read from Redux (synchronous, not SecureStore) |
| 9.2 | Request interceptor sets dynamic baseURL | 1. Be logged in to tenant with domain "acme.example.com" 2. Make API call | Request baseURL set to `https://api.acme.example.com` based on tenant domain from Redux (with MMKV fallback) |
| 9.3 | No token on unauthenticated requests | 1. Be logged out 2. Trigger login API call | Login request does NOT have Authorization header (no token in Redux) |
| 9.4 | Discovery client uses fixed URL | 1. Call tenant discovery endpoint | Uses separate `discoveryClient` with fixed base URL (not tenant-scoped), no Authorization header |
| 9.5 | Response interceptor passes non-401 errors | 1. Trigger API call that returns 400, 403, 404, 500 | Error passed through to caller without triggering token refresh |

---

## 10. Navigation & Auth State

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 10.1 | Auth Stack shown when not authenticated | 1. Open app with no session | `isAuthenticated: false` in Redux, Auth Stack rendered (Tenant Discovery or Sign In based on stored tenant domain) |
| 10.2 | Main Stack shown when authenticated | 1. Login or restore session | `isAuthenticated: true` in Redux, Main Stack rendered with Home screen |
| 10.3 | Instant navigation switch on login | 1. Login successfully | Navigation immediately switches from Auth Stack to Main Stack (driven by Redux state change) |
| 10.4 | Instant navigation switch on logout | 1. Logout | Navigation immediately switches from Main Stack to Auth Stack |
| 10.5 | No flash of wrong stack on cold start | 1. Kill app with valid session 2. Reopen | RootNavigator waits for `sessionRestoreSettled` before rendering, no flash of Sign In before Home |
| 10.6 | Auth Stack initial route with tenant | 1. Have stored tenant domain 2. Open Auth Stack | Initial route is Sign In (skips Tenant Discovery) |
| 10.7 | Auth Stack initial route without tenant | 1. Fresh install, no tenant domain 2. Open Auth Stack | Initial route is Tenant Discovery |

---

## 11. Edge Cases & Stress Tests

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 11.1 | Rapid login/logout cycle | 1. Login 2. Immediately logout 3. Immediately login again | All state transitions complete cleanly, no stale data from previous session, no race conditions |
| 11.2 | App backgrounded during login | 1. Enter credentials and tap Sign In 2. Immediately background the app 3. Return to app | Login completes (or fails gracefully), state is consistent |
| 11.3 | App killed during token refresh | 1. Have expired token 2. Trigger API call (refresh starts) 3. Kill app 4. Reopen | Session restore runs fresh, gets new token via refresh endpoint, no corrupted state |
| 11.4 | Multiple tabs/instances (if web) | 1. Open app in two browser tabs 2. Logout in one | Other tab should detect auth state change (if applicable) |
| 11.5 | Token expires exactly during navigation | 1. Token expires 2. Navigate to a screen that fetches data | 401 caught, refresh triggered, screen loads after successful refresh — no error flash |
| 11.6 | Login with special characters in password | 1. Use password with `!@#$%^&*()` etc. 2. Tap Sign In | Password sent correctly, no encoding issues, login succeeds if credentials are valid |
| 11.7 | Very long email address | 1. Enter email with 254 characters (max valid length) | Form accepts it, validation passes if format is valid |
| 11.8 | Login after prolonged app background | 1. Login 2. Background app for hours (token expires) 3. Return to app | First API call triggers 401 → auto-refresh → user sees no interruption |

---

## 12. Security

| # | Test Case | Steps | Expected Result |
|---|-----------|-------|-----------------|
| 12.1 | Token stored encrypted | 1. Login 2. Inspect device storage | JWT stored in SecureStore (encrypted by OS keychain/keystore), NOT in MMKV or AsyncStorage |
| 12.2 | Token not in MMKV | 1. Login 2. Inspect MMKV storage | Only `is_logged_in`, `tenant_domain`, `tenant_name` in MMKV — no tokens or sensitive user data |
| 12.3 | Token cleared on failed refresh | 1. Force token refresh to fail | Token removed from SecureStore and Redux, not left in an accessible but invalid state |
| 12.4 | No token in logs | 1. Enable verbose logging 2. Perform login and API calls | JWT tokens are not logged in console/debug output |
| 12.5 | Logout invalidates server-side token | 1. Logout successfully 2. Try using the old token via direct API call | Server rejects the old token (401) — token is server-invalidated, not just locally cleared |
| 12.6 | Password not persisted | 1. Login 2. Inspect all storage layers | Password is never stored in SecureStore, MMKV, or Redux — only sent during login API call |

---

## Test Environment Notes

- **Platforms to test on**: iOS (physical + simulator), Android (physical + emulator)
- **Network conditions**: WiFi, cellular, airplane mode, slow network (throttled)
- **App states**: Fresh install, returning user, backgrounded, killed and reopened
- **Tenant configurations**: Single tenant, multiple tenants, no tenants found

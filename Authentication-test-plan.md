# Authentication Testing Plan

Manual test scenarios for the full authentication system, including biometric login.
All scenarios are written for execution on an **Android device with a fingerprint sensor** (e.g. Xiaomi Redmi Note 8).

---

## Known Stubs / Limitations

Read these before you start. They affect how you interpret results in certain scenarios.

- **Forgot Password is stubbed.** `handleSubmit` in `useForgotPassword.ts` never calls the backend. It shows a success modal immediately for any valid email format. Scenarios in Section 6 are form-validation checks only, not end-to-end tests.
- **No UI to disable biometrics.** `disableBiometricLogin()` exists in code but nothing calls it. Once biometrics are enabled the only reset is clearing app data from device settings.
- **Token refresh scenarios require backend cooperation.** Token lifetime is 6 hours. Scenarios 44–46 are only practical if the backend team can issue short-lived tokens or you use a network proxy (e.g. Charles, Proxyman) to manipulate responses.

---

## Section 1 — Login (Email / Password)

### 1. Happy-path login

- **Steps:** Enter a valid email and password. Tap "Log In".
- **Expected:** Button enters loading state. App navigates to Home. No error banner.

### 2. Both fields empty

- **Steps:** Tap "Log In" without typing anything.
- **Expected:** Two inline errors appear simultaneously — "Email is required" and "Password is required".

### 3. Email empty, password filled

- **Steps:** Leave email blank. Type a password (6+ chars). Tap "Log In".
- **Expected:** Only "Email is required" appears. Password field has no error.

### 4. Password empty, email filled

- **Steps:** Type a valid email. Leave password blank. Tap "Log In".
- **Expected:** Only "Password is required" appears.

### 5. Invalid email format

- **Steps:** Type `notanemail` in the email field. Type a valid password. Tap "Log In".
- **Expected:** "Please enter a valid email address". No API call is made.

### 6. Password shorter than 6 characters

- **Steps:** Type a valid email. Type `abc` as password. Tap "Log In".
- **Expected:** "Password must be at least 6 characters". No API call is made.

### 7. Wrong credentials (valid format, incorrect values)

- **Steps:** Enter a correctly-formatted email and a 6+ char password that are wrong. Tap "Log In".
- **Expected:** Red error banner at the top of the form displays the backend error message (e.g. "Invalid credentials"). App stays on SignIn.

### 8. Error banner clears when you start typing

- **Steps:** Reproduce scenario 7 to get the red error banner. Then type a single character in either field.
- **Expected:** The red error banner disappears immediately.

### 9. Inline validation errors clear when you start typing

- **Steps:** Reproduce scenario 2 (both inline errors). Type one character in the email field.
- **Expected:** The email inline error disappears. The password error stays until you touch the password field.

### 10. Login via keyboard "Done" key

- **Steps:** Type email. Tap the password field, type a password. Tap the "done" key on the keyboard.
- **Expected:** Login is triggered — identical behaviour to tapping "Log In".

### 11. Keyboard "Next" key on email field moves focus

- **Steps:** Tap email field, type an email. Tap the "next" key on the keyboard.
- **Expected:** Focus moves to the password field. No login fires.

### 12. Button stays disabled during login request

- **Steps:** Tap "Log In" with valid credentials. Watch the button immediately after.
- **Expected:** Button enters loading state and is untappable until the request completes or fails.

### 13. Special characters in password

- **Steps:** Log in with a password containing symbols such as `!@#$%&*`.
- **Expected:** Login succeeds if credentials are correct. No crash or encoding error.

### 14. Leading / trailing spaces in email

- **Steps:** Type ` user@example.com ` (with spaces). Enter a valid password. Tap "Log In".
- **Expected:** Validation passes (it trims before checking the regex). Observe the actual value sent to the backend — the request body sends the raw input, so verify the backend accepts or rejects it accordingly.

---

## Section 2 — Biometric Setup (Post-Login Prompt)

### 15. Setup prompt appears after first successful login

- **Precondition:** Biometrics have never been enabled in this app. Device fingerprint is enrolled.
- **Steps:** Log in with email / password.
- **Expected:** An alert pops up: *"Would you like to enable Fingerprint for faster login next time?"* with "No" and "Yes". The app has NOT navigated to Home yet — SignIn is still visible behind the alert.

### 16. Accept setup — fingerprint succeeds

- **Steps:** From scenario 15, tap "Yes". Place your finger on the sensor when the device prompts you.
- **Expected:** Fingerprint authenticates. Biometric login is enabled. App navigates to Home.

### 17. Accept setup — fingerprint fails or is cancelled

- **Steps:** From scenario 15, tap "Yes". Cancel the fingerprint prompt or use the wrong finger until the device rejects.
- **Expected:** A second alert: *"Could not enable biometric login"*. Dismiss it. App still navigates to Home. Biometric was NOT enabled — the prompt will appear again on the next login.

### 18. Decline setup

- **Steps:** From scenario 15, tap "No".
- **Expected:** App navigates to Home immediately. Biometric is not enabled. The prompt will appear again on the next login.

### 19. Prompt does NOT appear when biometric is already enabled

- **Precondition:** Scenario 16 was completed at least once previously and the biometric preference persists.
- **Steps:** Soft-logout (automatic because biometric is enabled). Log in again with email / password.
- **Expected:** No setup prompt. App navigates to Home directly after login.

---

## Section 3 — Biometric Login

### 20. Biometric button is visible after soft logout

- **Precondition:** Biometric was enabled (scenario 16). User soft-logged out.
- **Steps:** Observe the SignIn screen.
- **Expected:** Below "Log In" a divider labelled "Or login with" appears, followed by a button with a fingerprint icon: *"Log in with Fingerprint"*.

### 21. Successful biometric login

- **Steps:** Tap "Log in with Fingerprint". Place your finger on the sensor.
- **Expected:** Fingerprint authenticates. Button enters loading state. App navigates to Home. (Under the hood: authenticate → refresh token → restore session from cached data.)

### 22. Biometric login — user cancels

- **Steps:** Tap the biometric button. Cancel the fingerprint prompt.
- **Expected:** App stays on SignIn. No error alert is shown. Button returns to its normal state. (The code silently ignores `'Authentication cancelled'`.)

### 23. Biometric login — wrong finger / rejected

- **Steps:** Tap the biometric button. Use the wrong finger or let the device reject.
- **Expected:** Alert: *"Authentication Failed"* with the device error. App stays on SignIn.

### 24. Biometric button disabled while email login is loading

- **Steps:** Tap "Log In" with valid credentials. While the loading state is active, observe the biometric button.
- **Expected:** The biometric button is disabled.

### 25. Email login button disabled while biometric login is loading

- **Steps:** Tap the biometric button. While the fingerprint prompt or token refresh is in progress, observe the "Log In" button.
- **Expected:** "Log In" is disabled.

### 26. Biometric login after token is invalidated on backend

- **Precondition:** Biometric enabled, soft-logged out. Token revoked or expired server-side (requires backend cooperation or a proxy).
- **Steps:** Tap the biometric button. Authenticate with fingerprint.
- **Expected:** Fingerprint passes locally. Token refresh call fails. Alert: *"Session has expired. Please sign in with your email and password."* App stays on SignIn. The biometric button disappears (token cleared from storage, `hasStoredToken` becomes false).

### 27. Consecutive biometric login cycles

- **Steps:** Biometric login → open drawer → logout (soft) → biometric login → logout → biometric login. Repeat three times.
- **Expected:** Every cycle works cleanly. No stale state or crashes.

---

## Section 4 — Session Restore (Cold Start)

### 28. Cold start with a valid active session

- **Precondition:** User is logged in. App is fully killed from the recents list.
- **Steps:** Reopen the app.
- **Expected:** A loading spinner appears briefly. App goes directly to Home. The SignIn screen never flashes. (The `isRestoring` flag in `RootNavigator` specifically prevents this flash.)

### 29. Cold start after hard logout (biometric disabled)

- **Precondition:** User logged out with biometrics disabled. App fully killed.
- **Steps:** Reopen the app.
- **Expected:** SignIn screen appears immediately. No stuck spinner. No biometric button.

### 30. Cold start after soft logout (biometric enabled)

- **Precondition:** User soft-logged out (biometric enabled). App fully killed.
- **Steps:** Reopen the app.
- **Expected:** SignIn screen appears (NOT Home). The `is_logged_in` gate blocks session restore. The biometric button IS visible because the token and preference are still in storage.

### 31. Cold start — fresh install

- **Precondition:** App freshly installed, never logged in.
- **Steps:** Open the app.
- **Expected:** SignIn screen. No stuck spinner. No biometric button. No errors.

### 32. Cold start with token but missing user data (partial state)

- **Precondition:** Difficult to reproduce manually. If possible, use a dev tool to delete only the MMKV `user` key while leaving the token in SecureStore and `is_logged_in` as `true`.
- **Steps:** Reopen the app.
- **Expected:** Session restore detects `user` or `tenant` is null while the token exists. It clears all auth data (SecureStore + MMKV). App shows SignIn. No crash.

### 33. Background → foreground (not a cold start)

- **Precondition:** User is logged in.
- **Steps:** Press the home button. Tap the app icon to bring it back.
- **Expected:** App returns to wherever it was. No re-login. Session restore does not re-run (`refetchOnWindowFocus: false`).

---

## Section 5 — Logout

### 34. Logout with biometrics disabled (hard logout)

- **Precondition:** Logged in, biometric NOT enabled.
- **Steps:** Open drawer. Tap "Log out".
- **Expected:** Drawer closes. Logout API is called. All local data cleared (token, email, user, tenant, permissions). App shows SignIn. No biometric button.

### 35. Logout with biometrics enabled (soft logout)

- **Precondition:** Logged in with biometrics enabled.
- **Steps:** Open drawer. Tap "Log out".
- **Expected:** Drawer closes. Logout API is NOT called (token must stay valid for the next biometric login). Token and user data remain in storage. Only `is_logged_in` is set to `false`. App shows SignIn. Biometric button IS visible.

### 36. Token is still usable after soft logout

- **Steps:** After scenario 35, tap the biometric button and authenticate.
- **Expected:** Token refresh succeeds. App navigates to Home.

### 37. Logout when backend is unreachable (biometric disabled)

- **Precondition:** Logged in, biometric NOT enabled. Network disabled.
- **Steps:** Open drawer. Tap "Log out".
- **Expected:** Logout API call fails, but local cleanup continues. All local data cleared. App shows SignIn. No crash. (Code logs a warning and continues.)

### 38. Rapid double-tap on logout button

- **Steps:** Open the drawer. Tap "Log out" very quickly twice.
- **Expected:** No crash. Mutation fires once; the second tap has no effect.

---

## Section 6 — Forgot Password

> Reminder: the forgot password endpoint is currently stubbed. No real API call is made.

### 39. Empty email

- **Steps:** Tap "Forgot Password?" on SignIn. Clear the email field if pre-filled. Tap "Send Reset Link".
- **Expected:** "Email is required" error appears.

### 40. Invalid email format

- **Steps:** Type `badformat`. Tap "Send Reset Link".
- **Expected:** "Please enter a valid email address" error appears.

### 41. Valid email (stub behaviour)

- **Steps:** Type a valid email. Tap "Send Reset Link".
- **Expected:** Success modal: *"Reset link sent"*. Tap "Back to Login" in the modal. App returns to SignIn.

### 42. Email pre-filled from SignIn

- **Steps:** On SignIn, type `user@example.com`. Tap "Forgot Password?".
- **Expected:** The email field on ForgotPassword is pre-populated with `user@example.com`.

### 43. "Back to Login" text button

- **Steps:** On ForgotPassword (before submitting), tap the "Back to Login" text button at the bottom.
- **Expected:** App returns to SignIn.

---

## Section 7 — Token Refresh (Automatic Interceptor)

> These scenarios require either backend support for short-lived tokens or a network proxy to manipulate responses.

### 44. Transparent refresh on 401

- **Precondition:** Backend can issue a token that expires quickly, or use a proxy to return 401 on the next call.
- **Steps:** Log in. Wait for or trigger token expiry. Perform any action that makes an API call.
- **Expected:** The interceptor silently refreshes the token, retries the original request, and continues. No visible error or interruption.

### 45. Refresh fails — forced logout

- **Precondition:** Both access and refresh attempts return errors from the backend.
- **Steps:** Make an API call after the token is fully invalid.
- **Expected:** Interceptor attempts refresh, refresh fails, all local auth data is cleared, app redirects to SignIn. No cryptic error.

### 46. No infinite loop when refresh endpoint itself returns 401

- **Precondition:** The `/v1/auth/refresh-token` endpoint returns 401.
- **Steps:** Trigger a refresh (via an expired token on any other endpoint).
- **Expected:** The interceptor has an `isRefreshTokenRequest()` guard. The refresh 401 is rejected immediately — no retry loop. App clears auth and goes to SignIn.

---

## Section 8 — Network / Connectivity

### 47. Login attempt with no network

- **Steps:** Disable WiFi and mobile data. Enter valid credentials. Tap "Log In".
- **Expected:** An error message is displayed (Axios network error). App stays on SignIn. No crash.

### 48. Biometric login with no network

- **Precondition:** Biometric enabled, soft-logged out. Network disabled.
- **Steps:** Tap the biometric button. Authenticate with fingerprint.
- **Expected:** Fingerprint passes on device. Token refresh fails (no network). Alert: *"Session has expired. Please sign in with your email and password."* App stays on SignIn.

### 49. Cold start with no network but valid cached session

- **Precondition:** Logged in with a valid session. Close app. Disable network. Reopen.
- **Steps:** Observe the app.
- **Expected:** Session restore reads token and user data from local storage only — no network needed. App goes to Home. Any subsequent API call from Home will fail on its own, but the restore itself succeeds.

### 50. Soft logout with no network

- **Precondition:** Biometric enabled. Network disabled.
- **Steps:** Open drawer. Tap "Log out".
- **Expected:** Soft logout does not call the API at all. `is_logged_in` is set to `false` locally. App goes to SignIn cleanly.

---

## Section 9 — UI / Interaction Details

### 51. Password visibility toggle

- **Steps:** Type a password. Locate the eye icon in the password field. Tap it.
- **Expected:** Password characters become visible. Tap again — they are hidden.

### 52. Keyboard does not permanently cover the form

- **Steps:** Tap the email field. Observe the form layout with the keyboard open.
- **Expected:** The form scrolls or shifts so the password field and "Log In" button remain accessible.

### 53. Drawer logout button position

- **Steps:** Log in. Open the drawer. Scroll to the bottom.
- **Expected:** "Log out" with a red icon and red text is at the very bottom, separated from the menu by a horizontal line.

### 54. Loading spinner on startup is centred and clean

- **Steps:** Cold start with a valid session (scenario 28).
- **Expected:** A centred loading spinner on a white background. The Auth Stack does not flash behind it at any point.

---

## Section 10 — Edge Cases & Regression Scenarios

### 55. Login → logout → login cycle

- **Steps:** Log in. Log out (hard, biometric not enabled). Log in again with the same credentials.
- **Expected:** Second login is identical to the first. No stale data or errors.

### 56. Repeated biometric login / logout cycles

- **Steps:** Log in → enable biometrics → soft-logout → biometric login → soft-logout → biometric login. Repeat three full cycles.
- **Expected:** Every cycle works. No accumulation of stale state or crashes.

### 57. Login → navigate to Forgot Password → back → login

- **Steps:** On SignIn, type email and password. Tap "Forgot Password?". On that screen, tap "Back to Login". Observe the SignIn fields. Tap "Log In".
- **Expected:** Either the fields retain their values and login works, or they are cleared and require re-entry. No crash in either case.

### 58. Device biometric enrollment removed mid-session

- **Precondition:** Biometric was enabled in the app. Then go to Android Settings and remove your fingerprint enrollment from the device.
- **Steps:** Soft-logout. Reopen the app. Observe the SignIn screen.
- **Expected:** The `useBiometrics` hook re-checks `isEnrolled` on mount. If the device reports not-enrolled, the biometric button should not render. Verify this on the actual device — this is a potential edge case.

### 59. App data clear / reinstall

- **Steps:** Android Settings → Apps → app → Storage → Clear All Data. Reopen the app.
- **Expected:** Behaves as a fresh install — SignIn screen, no biometric button, no stuck spinner.

### 60. Switching apps during biometric fingerprint prompt

- **Steps:** Tap the biometric button. While the device fingerprint prompt is visible, switch to another app and switch back.
- **Expected:** Device-dependent (Android may dismiss the prompt automatically). If dismissed, biometric login gracefully fails or cancels. No crash.

### 61. Verify request payload (with network proxy)

- **Steps:** Set up a proxy (e.g. Proxyman, Charles) on your device. Log in.
- **Expected:** The login request body contains `device_name` in the format `"Redmi Note 8 - Android X.X"` and `remember: true`.

### 62. Verify refresh token uses SecureStore token (with network proxy)

- **Steps:** Observe the `Authorization` header on any `/v1/auth/refresh-token` request via your proxy.
- **Expected:** The token matches what is in SecureStore. The refresh API intentionally uses raw `axios` (not `apiClient`) to avoid the request interceptor attaching a potentially-stale Redux token.

# Authentication Module Guide

> **What this document is for**: A file-by-file map of every module that participates in authentication. Each entry states exactly what the file owns, what it exports, and what it depends on. The final section shows how all the pieces connect.
>
> **Companion docs**: [AUTH_FLOWS.md](./AUTH_FLOWS.md) answers "what happens step-by-step when X occurs." [AUTHENTICATION.md](./AUTHENTICATION.md) has API contracts and security notes. This guide answers "what does this file do and who talks to it?"

---

## Directory Layout (auth-related files only)

```
src/
├── types/
│   ├── auth.ts                          # Shared type definitions
│   └── navigation.ts                    # Navigation param-list types
├── utils/
│   ├── biometrics.ts                    # expo-local-authentication wrapper
│   ├── validation.ts                    # Form validation (email, password)
│   └── device.ts                        # Device-name string for login payload
├── services/
│   ├── api/
│   │   ├── config.ts                    # Base URL resolution
│   │   ├── tokenRefreshManager.ts       # Refresh-state + request queue
│   │   └── client.ts                    # Axios instance + interceptors
│   ├── storage/
│   │   ├── secureStore.ts               # Encrypted persistence (tokens)
│   │   └── mmkvStore.ts                 # Fast cache (user, tenant, flags)
│   └── authCleanup.ts                   # Single "wipe auth from storage" call
├── store/
│   ├── index.ts                         # Redux store configuration
│   ├── hooks.ts                         # Typed useAppDispatch / useAppSelector
│   └── auth/
│       └── authSlice.ts                 # Auth reducer + all auth actions
├── features/
│   └── auth/
│       ├── api/
│       │   ├── signInApi.ts             # POST /v1/auth/login
│       │   ├── refreshTokenApi.ts       # POST /v1/auth/refresh-token
│       │   ├── logoutApi.ts             # POST /v1/auth/logout
│       │   └── index.ts                 # Barrel
│       ├── mutations/
│       │   ├── useLoginMutation.ts      # Login + dual-storage write + rollback
│       │   ├── useLogoutMutation.ts     # Logout (full or biometric-preserving)
│       │   ├── useTokenRefreshMutation.ts # Manual token refresh
│       │   └── index.ts                 # Barrel
│       ├── queries/
│       │   ├── useRestoreSessionQuery.ts # Cold/warm session restore on startup
│       │   └── index.ts                 # Barrel
│       ├── hooks/
│       │   ├── useSignIn.ts             # All business logic for SignIn screen
│       │   └── useForgotPassword.ts     # All business logic for ForgotPassword
│       └── index.ts                     # Feature barrel
├── hooks/
│   └── useBiometrics.ts                 # Biometric capability + enable/disable
├── navigation/
│   ├── RootNavigator.tsx                # Auth vs Main stack switcher
│   ├── AuthStack.tsx                    # SignIn + ForgotPassword stack
│   └── types.ts                         # Re-exports from types/navigation.ts
├── components/
│   ├── auth/
│   │   └── BiometricButton/
│   │       └── index.tsx                # Biometric login button (UI only)
│   └── layouts/
│       └── AuthLayout.tsx               # Two-tone background + title header
├── screens/
│   ├── auth/
│   │   ├── SignInScreen/
│   │   │   └── index.tsx                # Login form + biometric button (UI only)
│   │   └── ForgotPasswordScreen/
│   │       └── index.tsx                # Reset-password form (UI only)
│   └── main/
│       └── MoreScreen/
│           └── index.tsx                # Debug: live storage inspector
└── navigation/
    └── components/
        └── CustomDrawerContent.tsx      # Drawer menu (contains logout trigger)
```

---

## Module-by-Module Reference

Each section below covers one file (or one tightly related group). Sections are ordered bottom-up: foundational modules first, higher-level orchestration last. This mirrors the dependency direction — modules later in the document depend on modules earlier.

---

### 1. `src/types/auth.ts` — Shared Type Definitions

**Responsibility**: Single source of truth for every TypeScript interface used across the auth system. No runtime code lives here.

**Exports used by other modules**:

| Type | Used by | What it represents |
|---|---|---|
| `LoginCredentials` | `signInApi.ts`, `useLoginMutation` | What the user sends to log in (`email`, `password`, `remember`, `device_name`) |
| `LoginApiResponse` | `signInApi.ts` | Raw JSON shape from `POST /login` (snake_case keys) |
| `LoginResponse` | `useLoginMutation`, `useSignIn`, `authSlice` | Transformed shape after mutation maps snake_case → camelCase |
| `User` | `authSlice`, `mmkvStore`, `useRestoreSessionQuery` | `{ id, name, email }` |
| `Tenant` | `authSlice`, `mmkvStore`, `useRestoreSessionQuery` | `{ id, name }` |
| `TokenRefreshApiResponse` | `refreshTokenApi.ts`, `useTokenRefreshMutation` | Raw JSON from `POST /refresh-token` |
| `LogoutApiResponse` | `logoutApi.ts` | Raw JSON from `POST /logout` |
| `AuthState` | `authSlice` | Shape of the Redux `auth` slice |
| `ForgotPasswordRequest` / `ForgotPasswordResponse` | (future use) | Password-reset contract |

---

### 2. `src/utils/biometrics.ts` — Platform Biometric Wrapper

**Responsibility**: Thin wrapper around `expo-local-authentication`. Handles the three questions the rest of the app needs answered: *Can this device do biometrics? What kind? Can it authenticate right now?* No state, no storage, no Redux.

**Exports**:
- `BiometricType` — `'face' | 'fingerprint' | 'iris' | null`
- `BiometricSupport` — `{ isAvailable, biometricType, isEnrolled }`
- `checkBiometricSupport()` — queries the device for hardware + enrollment status
- `getBiometricDisplayName(type)` — returns the platform-appropriate label (`Face ID` on iOS, `Face Recognition` on Android, etc.)
- `authenticateWithBiometrics(promptMessage?)` — triggers the OS authentication dialog. Returns `{ success, error? }`. Sets `disableDeviceFallback: true` so PIN/passcode is never offered as a fallback.

**Used by**: `useBiometrics.ts` (the only consumer).

---

### 3. `src/utils/validation.ts` — Form Validation

**Responsibility**: Pure functions that validate form fields before any network call is made. Returns structured error objects the UI renders directly.

**Exports**:
- `validateEmail(email)` — required + regex check
- `validatePassword(password)` — required + minimum 6 characters
- `validateLoginForm(email, password)` — runs both and returns `{ isValid, errors: { email?, password? } }`

**Used by**: `useSignIn.ts` (login form), `useForgotPassword.ts` (email field).

---

### 4. `src/utils/device.ts` — Device Identification

**Responsibility**: Generates the `device_name` string that is sent to the backend on every login. Format: `"iPhone 14 Pro - iOS 17.1"`. The backend uses this so users can see which devices have active sessions.

**Exports**: `getDeviceName()` — returns the formatted string.

**Used by**: `signInApi.ts` (as the default value of `device_name` in the login payload).

---

### 5. `src/services/storage/secureStore.ts` — Encrypted Persistent Storage

**Responsibility**: Owns all reads and writes to `expo-secure-store` (iOS Keychain / Android KeyStore). Every sensitive piece of data that must survive app kills and must be encrypted lives here. This file is the only place in the codebase that calls `SecureStore.getItemAsync` / `setItemAsync` / `deleteItemAsync`.

**What it stores**:

| Key | Value | Written by | Read by |
|---|---|---|---|
| `auth_access_token` | JWT string | `useLoginMutation`, `client.ts` (refresh), `useTokenRefreshMutation` | `refreshTokenApi.ts`, `useBiometrics`, `useRestoreSessionQuery`, `MoreScreen` |
| `biometric_enabled` | `"true"` or `"false"` | `useBiometrics` | `useBiometrics`, `useRestoreSessionQuery`, `MoreScreen` |
| `user_email` | email string | `useLoginMutation` | `MoreScreen` |

**Key design decisions**:
- `safeGetItem` returns `null` on any error instead of throwing — reads are always safe.
- `safeSetItem` **does** throw — callers (login, refresh) need to know if a token write failed so they can abort.
- `safeDeleteItem` never throws — a deletion failure during cleanup is non-critical.
- `clearAllAuthData()` deliberately does **not** delete `biometric_enabled`. That is a user preference, not session data. Use `clearAllAuthDataIncludingPreferences()` only for full resets.

---

### 6. `src/services/storage/mmkvStore.ts` — Fast Synchronous Cache

**Responsibility**: Owns all reads and writes to MMKV. Non-sensitive data that needs to be loaded instantly on startup (no `await`) lives here. This is the only file that touches the `MMKV` instance directly.

**What it stores**:

| Key | Value | Written by | Read by |
|---|---|---|---|
| `user` | JSON `User` object | `useLoginMutation` | `useRestoreSessionQuery`, `useSignIn` (biometric path), `MoreScreen` |
| `tenant` | JSON `Tenant` object | `useLoginMutation` | `useRestoreSessionQuery`, `useSignIn` (biometric path), `MoreScreen` |
| `permissions` | JSON `string[]` | `useLoginMutation` | `useRestoreSessionQuery`, `useSignIn` (biometric path), `MoreScreen` |
| `is_logged_in` | boolean | `useLoginMutation`, `useLogoutMutation`, `useSignIn` (biometric path) | `useRestoreSessionQuery` |

**`is_logged_in` is the session-restore gate.** `useRestoreSessionQuery` reads this synchronously first. If `false`, it short-circuits immediately — zero async work, SignIn appears in under a millisecond. The token may still exist in SecureStore (biometric-preserving logout keeps it), but session restore will not run.

**`safeJsonParse`**: If MMKV contains corrupted JSON for a key, this helper catches the error, deletes the corrupted entry, and returns the default value. The app never crashes from a bad cache entry.

---

### 7. `src/services/api/config.ts` — Base URL

**Responsibility**: Single place to resolve the API base URL. Reads `EXPO_PUBLIC_API_URL` from the environment; falls back to the staging URL.

**Exports**: `getBaseUrl()`.

**Used by**: `client.ts` (for the axios instance `baseURL`), `refreshTokenApi.ts` (for the raw-axios refresh call).

---

### 8. `src/services/api/tokenRefreshManager.ts` — Refresh Lock + Queue

**Responsibility**: Prevents multiple simultaneous token refresh requests. When the first 401 triggers a refresh, all subsequent 401s that arrive while the refresh is in flight are added to a queue. Once the refresh resolves, every queued request is either retried (success) or rejected (failure) in one batch.

**Exports**:
- `TokenRefreshManager` class — `setRefreshing`, `getIsRefreshing`, `addToQueue`, `processQueue`, `reset`
- `tokenRefreshManager` — the singleton instance used by `client.ts`
- `resetTokenRefreshState()` — calls `reset()`. Used during logout to ensure clean state before the next login.

**Used by**: `client.ts` (response interceptor), `useLogoutMutation` (cleanup), `authCleanup.ts` (cleanup).

---

### 9. `src/services/api/client.ts` — Axios Instance + Interceptors

**Responsibility**: The single HTTP client the entire app uses for authenticated requests. It does two things beyond a plain axios instance:

**Request interceptor** — runs before every outgoing request:
- Reads `accessToken` from Redux (`store.getState().auth.accessToken`).
- If present, attaches `Authorization: Bearer <token>`.
- This is synchronous and reads from in-memory Redux, not from SecureStore. That is intentional: SecureStore is async and would add 100-200ms to every request.

**Response interceptor** — runs after every response:
- Passes 2xx responses through untouched.
- On a 401, it checks three guards:
  1. Is this the refresh endpoint itself? → reject immediately (prevents infinite loop).
  2. Is this the logout endpoint? → reject immediately (the token is being intentionally invalidated; intercepting here would clear tokens before `useLogoutMutation`'s biometric-aware cleanup runs).
  3. Has this request already been retried (`_retry` flag)? → reject (prevents a second refresh attempt on the same request).
- If all guards pass and no refresh is in flight, it becomes the "leader":
  - Sets `_retry = true` on the original request.
  - Calls `refreshToken()` (from `refreshTokenApi.ts` — raw axios, not `apiClient`, to bypass its own interceptors).
  - On success: updates SecureStore and Redux with the new token, processes the queue, retries the original request.
  - On failure: processes the queue with the error, calls `clearAuthState()`, dispatches `logoutSuccess()`.
- If a refresh is already in flight, this request is added to the queue and waits.

**Exports**: `apiClient` (the axios instance). This is what `signInApi.ts` and `logoutApi.ts` use.

**Why `refreshTokenApi.ts` uses raw `axios` instead of `apiClient`**: Two reasons. First, on a cold start after a biometric-preserving logout, Redux `accessToken` is `null` — the request interceptor would attach nothing. The refresh needs the real token, which is in SecureStore. Second, if `apiClient` were used for the refresh call, a 401 on that call would trigger the response interceptor again, causing infinite recursion.

---

### 10. `src/services/authCleanup.ts` — Centralized Storage Wipe

**Responsibility**: The single function that tears down auth state across storage. Every place in the codebase that needs to "clear auth from disk" calls this instead of reaching into SecureStore and MMKV individually.

**What it does** (in order):
1. `clearAllAuthData()` from SecureStore (token + email; preserves biometric preference).
2. `clearAuthData()` from MMKV (user, tenant, permissions; sets `is_logged_in = false`).
3. `resetTokenRefreshState()` from tokenRefreshManager.

**What it does NOT do**: It does not dispatch to Redux and it does not touch React Query. Callers handle that themselves because different call sites need different Redux actions and different Query-cache strategies.

**Used by**: `client.ts` (401 refresh failure), `useTokenRefreshMutation` (manual refresh failure), `useLogoutMutation` (full/non-biometric logout).

---

### 11. `src/store/index.ts` — Redux Store

**Responsibility**: Creates and exports the Redux store. Currently contains one reducer: `auth`. Also exports the `RootState` and `AppDispatch` types that the typed hooks depend on.

**Exports**: `store`, `RootState`, `AppDispatch`.

**Used by**: `store/hooks.ts` (for types), `client.ts` (imports `store` directly to call `store.getState()` and `store.dispatch()` outside of React — the interceptors run outside component lifecycle).

---

### 12. `src/store/hooks.ts` — Typed Redux Hooks

**Responsibility**: Re-exports `useDispatch` and `useSelector` with the app's types baked in. Every component or hook in the app uses these instead of the raw Redux hooks, so TypeScript knows the shape of state without a cast.

**Exports**: `useAppDispatch`, `useAppSelector`.

**Used by**: Every feature hook, mutation, query, and screen that reads from or writes to Redux.

---

### 13. `src/store/auth/authSlice.ts` — Auth State + Actions

**Responsibility**: The Redux slice that is the in-memory source of truth for all auth state while the app is running. Navigation reads `isAuthenticated` from here. The API client reads `accessToken` from here. The UI reads `error` and loading states from here.

**State shape**:
```
user: User | null
tenant: Tenant | null
permissions: string[]
accessToken: string | null
isAuthenticated: boolean
isLoading: boolean
error: string | null
biometricEnabled: boolean
```

**Actions and who dispatches them**:

| Action | Dispatched by | Effect |
|---|---|---|
| `setLoading(bool)` | `useLoginMutation`, `useRestoreSessionQuery` | Sets the loading spinner |
| `loginSuccess(LoginResponse)` | `useSignIn` (after optional biometric prompt) | Populates all fields, sets `isAuthenticated = true` |
| `loginFailure(message)` | `useLoginMutation` onError | Sets `error`, clears loading |
| `restoreSessionSuccess(...)` | `useRestoreSessionQuery`, `useSignIn` (biometric login path) | Restores full state from stored data |
| `restoreSessionFailure()` | `useRestoreSessionQuery` | Clears loading, sets `isAuthenticated = false` |
| `logoutSuccess()` | `useLogoutMutation`, `client.ts` (refresh failure), `useTokenRefreshMutation` (refresh failure) | Clears user/tenant/permissions/token/error. **Does not touch `biometricEnabled`** — that is a user preference that outlives sessions. |
| `updateAccessToken(token)` | `client.ts` (after successful refresh), `useTokenRefreshMutation` | Replaces the token in Redux so subsequent requests use the new one |
| `setBiometricEnabled(bool)` | `useBiometrics` | Syncs the biometric preference into Redux |
| `clearError()` | `useSignIn` (when user starts typing) | Removes the error banner |
| `setUser`, `setTenant`, `setPermissions`, `setAuthenticated` | (manual updates if needed) | Direct field setters |

**Critical design note on `logoutSuccess`**: It intentionally preserves `biometricEnabled`. If it cleared it, the biometric button would disappear on logout even though the preference is still in SecureStore. The value is restored from SecureStore by `useBiometrics` on the next mount.

---

### 14. `src/features/auth/api/signInApi.ts` — Login API Call

**Responsibility**: Owns exactly one thing: sending `POST /v1/auth/login` and returning the raw response. It builds the request body (adding defaults for `remember` and `device_name`) and calls `apiClient.post`. No storage logic, no Redux, no error handling beyond what axios throws.

**Why it uses `apiClient`** (not raw axios): At login time there is no token yet, so the request interceptor is a no-op. Using `apiClient` is fine and keeps the code consistent.

---

### 15. `src/features/auth/api/refreshTokenApi.ts` — Refresh API Call

**Responsibility**: Sends `POST /v1/auth/refresh-token` with the current token and returns the raw response.

**Why it uses raw `axios` instead of `apiClient`**: See [Section 9](#9-srcservicesapiclientts--axios-instance--interceptors) — the request interceptor would attach a stale or null token from Redux, and the response interceptor would create an infinite loop.

**Where it reads the token from**: SecureStore (`getAccessToken()`), not Redux. On cold starts after biometric-preserving logout, Redux `accessToken` is null but the real token is still in SecureStore.

---

### 16. `src/features/auth/api/logoutApi.ts` — Logout API Call

**Responsibility**: Sends `POST /v1/auth/logout`. The token is attached automatically by `apiClient`'s request interceptor — this file sends no body at all.

**Note**: This function is only called in the full-logout path. The biometric-preserving logout path in `useLogoutMutation` skips this entirely (calling the API would invalidate the token the biometric flow needs later).

---

### 17. `src/features/auth/mutations/useLoginMutation.ts` — Login Orchestrator

**Responsibility**: The mutation that turns a successful API response into a fully persisted session. It is the only place in the codebase that writes to **both** storage layers in the correct order with a rollback strategy.

**Step-by-step**:
1. Dispatches `setLoading(true)`.
2. Calls `login()` from `signInApi.ts`.
3. Maps the snake_case API response to the camelCase `LoginResponse`.
4. Writes the token and email to SecureStore (the critical write — if this fails, nothing else has been written, so no cleanup is needed).
5. Writes user, tenant, permissions, and `is_logged_in = true` to MMKV inside a `try` block.
6. If the MMKV block throws: rolls back the token from SecureStore, clears any partial MMKV data, and re-throws. This prevents a half-written session that would confuse the next startup.
7. Returns the `LoginResponse` to the caller.

**What it does NOT do**: It does not dispatch `loginSuccess`. That is intentionally left to `useSignIn`, which needs to show the biometric setup prompt first (see [Section 22](#22-srcfeaturesauthhooksusesignints--sign-in-business-logic)).

---

### 18. `src/features/auth/mutations/useLogoutMutation.ts` — Logout Orchestrator

**Responsibility**: Handles both logout paths — full logout and biometric-preserving ("soft") logout — and guarantees the user is always logged out locally, even if the API or storage throws.

**Two paths, decided by `biometricEnabled` from Redux**:

**Full logout** (`biometricEnabled = false`):
1. Calls `logout()` API. If it fails, logs a warning and continues — network outages must never trap a user in the app.
2. Calls `clearAuthState()` (wipes SecureStore + MMKV + resets refresh manager).
3. `onSuccess`: dispatches `logoutSuccess()`, clears the entire React Query cache.

**Biometric-preserving logout** (`biometricEnabled = true`):
1. Does **not** call the logout API. The token must remain valid on the server so the biometric flow can refresh it later.
2. Does **not** clear SecureStore or MMKV user data. The biometric flow reads user/tenant/permissions from MMKV after refresh.
3. Only writes `is_logged_in = false` to MMKV. This flips the gate that prevents automatic session restore on the next cold start.
4. `onSuccess`: dispatches `logoutSuccess()`, **invalidates** (not clears) only the `['auth', 'session-restore']` query.

**Resilience**: `onError` calls `finalizeLogout(false)` — dispatches `logoutSuccess()` and clears the full cache. The user is always logged out, even if `mutationFn` threw partway through.

---

### 19. `src/features/auth/mutations/useTokenRefreshMutation.ts` — Manual Token Refresh

**Responsibility**: Exposes a React-hook-friendly way to refresh the token on demand. This is for **manual** refresh scenarios: proactive refresh before a long operation, or the biometric login flow. The automatic refresh on 401 is handled by `client.ts`'s interceptor — do not use both on the same request.

**What it does**:
1. Reads the current token from SecureStore (not Redux — for the same cold-start reason as `refreshTokenApi`).
2. Calls `refreshToken()` API.
3. Writes the new token to SecureStore.
4. `onSuccess`: dispatches `updateAccessToken` to Redux.
5. `onError`: calls `clearAuthState()` and dispatches `logoutSuccess()` — a refresh failure means the session is dead.

**Used by**: `useSignIn.ts` — the biometric login path calls `mutateAsync()` to get a fresh token after the OS authenticates the user.

---

### 20. `src/features/auth/queries/useRestoreSessionQuery.ts` — Session Restore on Startup

**Responsibility**: The query that runs once on app startup and decides whether a previous session can be restored. It is mounted by `RootNavigator` and its result directly controls whether the user sees the Auth Stack or the Main Stack.

**Flow**:
1. Reads `is_logged_in` from MMKV synchronously. If `false`, throws immediately — this is the fast path that shows SignIn with zero async work.
2. Reads the token from SecureStore (async).
3. Reads user, tenant, permissions from MMKV (sync, runs in parallel with step 2).
4. Reads `biometric_enabled` from SecureStore (async, parallel with step 2).
5. **Integrity check**: If the token exists but user or tenant is missing, the session is corrupted (e.g. app crashed mid-login write). Clears everything and throws.
6. Returns the full session payload. A `useEffect` dispatches `restoreSessionSuccess` exactly once (guarded by `useRef` flags to prevent duplicate dispatches if React re-runs the effect).

**Query configuration**: `retry: false`, `staleTime: Infinity`, `refetchOnMount/WindowFocus/Reconnect: false`. This query should run exactly once per app lifecycle. It is re-fired only when explicitly invalidated (e.g. after biometric-preserving logout).

---

### 21. `src/hooks/useBiometrics.ts` — Biometric Capability Hook

**Responsibility**: Answers four questions that the SignIn screen needs:
1. Does this device support biometrics and are they enrolled? (hardware check)
2. Has the user enabled biometric login in this app? (SecureStore preference)
3. Is there a stored token that biometric login can work with? (SecureStore token check)
4. Are all of the above true at the same time? (`isBiometricLoginAvailable` — the single flag the UI checks to decide whether to show the biometric button)

**Initialization**: On mount, runs three reads in parallel (`checkBiometricSupport`, `getBiometricEnabled`, `getAccessToken`), sets local state, and syncs `biometricEnabled` into Redux via `setBiometricEnabled`.

**`enableBiometricLogin()`**: Before saving the preference, it triggers an actual biometric authentication. This is a smoke test — it proves the user can actually use biometrics before the button is enabled. If the OS auth fails, the preference is never saved and the button never appears.

**`isMountedRef`**: All state updates are guarded by this ref to prevent React warnings if the component unmounts while async reads are still in flight.

**Used by**: `useSignIn.ts` (consumes `isBiometricLoginAvailable`, `authenticate`, `enableBiometricLogin`, display names).

---

### 22. `src/features/auth/hooks/useSignIn.ts` — Sign-In Business Logic

**Responsibility**: Contains all the logic for the SignIn screen. The screen component (`SignInScreen/index.tsx`) is pure UI — it calls this hook and renders whatever it returns. This hook is the place where the login flow, the biometric login flow, and the biometric setup prompt are all coordinated.

**What it owns**:
- Form state (`email`, `password`, `formErrors`).
- `handleLogin()` — validates → calls `loginMutation.mutate()` → on success, calls `promptBiometricSetup`.
- `promptBiometricSetup(loginData)` — if the device supports biometrics and the user hasn't enabled them, shows an Alert asking. If the user says Yes, calls `enableBiometricLogin()` (the smoke-test path). `loginSuccess` is dispatched **inside** the Alert callbacks, not before — this keeps SignInScreen mounted while the prompt is visible, which keeps `useBiometrics`' `isMountedRef` valid.
- `handleBiometricLogin()` — the full biometric login flow:
  1. Triggers OS biometric auth via `authenticateBiometric()`.
  2. On success, calls `tokenRefreshMutation.mutateAsync()` to get a fresh token (the stored token may have expired).
  3. Flips `is_logged_in = true` in MMKV.
  4. Reads user/tenant/permissions from MMKV.
  5. Dispatches `restoreSessionSuccess` with the fresh token and cached data.
  6. If refresh fails, shows an alert ("Session has expired") — `useTokenRefreshMutation`'s `onError` has already cleared storage and dispatched logout.
- `handleForgotPassword()` — navigates to ForgotPassword, passing the current email value as a param so the form can be pre-filled.

---

### 23. `src/features/auth/hooks/useForgotPassword.ts` — Forgot-Password Business Logic

**Responsibility**: Manages the ForgotPassword screen's form state, validation, and modal display. Note: as of the current code, the actual API call to send the reset email is not yet wired (the `handleSubmit` sets success state directly). This hook is the placeholder for that integration.

**Does not touch**: Redux, storage, or any auth mutations. It is self-contained form logic.

---

### 24. `src/navigation/RootNavigator.tsx` — Auth/Main Stack Switcher

**Responsibility**: The top-level navigator. It does three things:
1. Mounts `useRestoreSessionQuery` so session restore runs on startup.
2. Reads `isAuthenticated` from Redux.
3. Renders either `AuthStack` or `DrawerNavigator` based on that flag.

**Flash prevention**: There is a one-render gap between the query finishing and the `useEffect` that dispatches `restoreSessionSuccess`. During that gap, `isAuthenticated` is still `false` even though the session is about to be restored. `RootNavigator` detects this with `isRestoring = query.isLoading || (query.isSuccess && !isAuthenticated)` and keeps the loading spinner visible until Redux actually updates. This eliminates any flash of the SignIn screen.

---

### 25. `src/navigation/AuthStack.tsx` — Auth Screen Stack

**Responsibility**: A stack navigator containing SignIn and ForgotPassword. No auth logic lives here — it is purely a routing declaration.

---

### 26. `src/screens/auth/SignInScreen/index.tsx` — Sign-In UI

**Responsibility**: Renders the login form. Calls `useSignIn()` and maps its return values directly onto UI components. Contains zero business logic. Every handler (`handleLogin`, `handleBiometricLogin`, etc.) is a function reference from the hook.

**Conditional rendering**: The biometric section (Divider + BiometricButton) renders only when `isBiometricLoginAvailable` is true.

---

### 27. `src/components/auth/BiometricButton/index.tsx` — Biometric Button

**Responsibility**: A presentational component. Receives `biometricType`, `displayName`, `onPress`, `loading`, `disabled` as props. Picks the correct icon (`scan-outline` for face, `finger-print-outline` for fingerprint) and renders an `AppButton` with variant `secondary`. No logic.

---

### 28. `src/components/layouts/AuthLayout.tsx` — Auth Screen Layout

**Responsibility**: Provides the visual chrome shared by all auth screens: the two-tone background (teal top, light bottom) and the centered title + subtitle header. Screens pass their content as `children`.

---

### 29. `src/screens/auth/ForgotPasswordScreen/index.tsx` — Forgot-Password UI

**Responsibility**: Renders the reset-password form. Calls `useForgotPassword()` and maps return values to UI. Includes the `ResultModal` for success/error feedback.

---

### 30. `src/navigation/components/CustomDrawerContent.tsx` — Drawer (Logout Entry Point)

**Responsibility**: Renders the side-drawer menu. The only auth-relevant thing it does is mount `useLogoutMutation` and call `logoutMutation.mutate()` when the user taps "Log out". It also closes the drawer immediately after triggering the mutation.

---

### 31. `src/screens/main/MoreScreen/index.tsx` — Storage Debug Inspector

**Responsibility**: A development/debug screen that displays the live contents of all three state layers (SecureStore, MMKV, Redux) side by side. Useful for verifying that all layers stay in sync after login, refresh, or logout. Has a "Refresh" button to re-read async SecureStore values.

---

## How the Modules Talk to Each Other

### Dependency Graph

```
                        ┌─────────────────────────────────────┐
                        │              UI Layer                 │
                        │  SignInScreen   ForgotPasswordScreen  │
                        │  BiometricButton   AuthLayout         │
                        │  CustomDrawerContent                  │
                        └───────────┬─────────────┬────────────┘
                                    │             │
                          useSignIn()      useForgotPassword()
                          useLogoutMutation()
                                    │             │
                        ┌───────────▼─────────────▼────────────┐
                        │         Feature Hooks Layer           │
                        │  useSignIn    useForgotPassword       │
                        │  (orchestrates login + biometric)     │
                        └───────────┬─────────────┬────────────┘
                                    │             │
              ┌─────────────────────┼─────────────┼──────────────┐
              │                     │             │              │
     useLoginMutation     useLogoutMutation  useTokenRefresh  useBiometrics
     useRestoreSessionQuery                  Mutation
              │                     │             │              │
        ┌─────▼─────┐    ┌─────────▼───┐   ┌────▼────┐   ┌────▼────┐
        │  Mutations │    │  Mutations  │   │Mutation │   │  Hook   │
        └─────┬─────┘    └─────────┬───┘   └────┬────┘   └────┬────┘
              │                     │             │              │
        ┌─────▼─────────────────────▼─────────────▼──────────────▼────┐
        │                     Feature API Layer                        │
        │          signInApi   logoutApi   refreshTokenApi             │
        └─────┬──────────────────────┬──────────────┬─────────────────┘
              │                      │              │
              ▼                      ▼              ▼
        ┌───────────┐        ┌────────────┐  ┌────────────┐
        │ apiClient │        │ apiClient  │  │ raw axios  │
        │ (client)  │        │ (client)   │  │            │
        └───────────┘        └────────────┘  └────────────┘
              │                                     │
        ┌─────▼─────────────────────────────────────▼────┐
        │              Backend API                        │
        │  POST /login   POST /logout   POST /refresh     │
        └─────────────────────────────────────────────────┘

   All mutations + queries + client.ts also connect to:

        ┌──────────────────────────────────────────────┐
        │              Storage Layer                    │
        │  secureStore.ts       mmkvStore.ts            │
        │  (tokens, biometric)  (user, tenant, flag)    │
        └──────────────────────────────────────────────┘

        ┌──────────────────────────────────────────────┐
        │              Redux (authSlice)                │
        │  Single in-memory source of truth             │
        │  Read by: client.ts, RootNavigator, UI        │
        │  Written by: mutations, queries, client.ts    │
        └──────────────────────────────────────────────┘
```

### Key Data-Flow Paths

**Login** (left to right, top to bottom):
```
User types credentials
  → useSignIn.handleLogin()
    → validateLoginForm()                  [validation.ts]
    → loginMutation.mutate()               [useLoginMutation]
      → login()                            [signInApi → apiClient → Backend]
      → setAccessToken() + setStoredEmail()  [secureStore]
      → setUser() + setTenant() + setPermissions() + setLoggedIn(true)  [mmkvStore]
      → returns LoginResponse
    → promptBiometricSetup()               [useSignIn]
      → (optional) enableBiometricLogin()  [useBiometrics → biometrics.ts → OS]
    → dispatch(loginSuccess())             [authSlice]
      → isAuthenticated = true
        → RootNavigator renders Main Stack
```

**Session Restore** (app startup):
```
RootNavigator mounts
  → useRestoreSessionQuery fires
    → getLoggedIn()                        [mmkvStore] — if false, short-circuit
    → getAccessToken()                     [secureStore]
    → getUser() / getTenant() / getPermissions()  [mmkvStore]
    → getBiometricEnabled()                [secureStore]
    → integrity check (user + tenant must exist)
    → returns session payload
  → useEffect dispatches restoreSessionSuccess()  [authSlice]
    → isAuthenticated = true
      → RootNavigator renders Main Stack
```

**Biometric Login** (after OS authenticates):
```
User taps biometric button
  → useSignIn.handleBiometricLogin()
    → authenticateBiometric()              [useBiometrics → biometrics.ts → OS]
    → tokenRefreshMutation.mutateAsync()   [useTokenRefreshMutation]
      → refreshToken()                     [refreshTokenApi → raw axios → Backend]
      → setAccessToken(newToken)           [secureStore]
      → dispatch(updateAccessToken())      [authSlice]
    → setLoggedIn(true)                    [mmkvStore]
    → getUser() / getTenant() / getPermissions()  [mmkvStore]
    → dispatch(restoreSessionSuccess())    [authSlice]
      → isAuthenticated = true
        → RootNavigator renders Main Stack
```

**Automatic Token Refresh** (transparent, mid-session):
```
Any component makes an API call via apiClient
  → request interceptor attaches token from Redux
  → Backend returns 401
  → response interceptor fires
    → tokenRefreshManager: am I already refreshing? If yes → queue this request
    → refreshToken()                       [refreshTokenApi → raw axios → Backend]
    → setAccessToken(newToken)             [secureStore]
    → dispatch(updateAccessToken())        [authSlice]
    → processQueue(newToken)               [tokenRefreshManager — retries all queued requests]
    → retry original request with new token
  → Component receives its data. Nothing looked broken.
```

**Logout** (two variants):
```
User taps logout
  → useLogoutMutation.mutate()

  IF biometricEnabled = false (full logout):
    → logout()                             [logoutApi → apiClient → Backend]
    → clearAuthState()                     [authCleanup → secureStore + mmkvStore + tokenRefreshManager]
    → dispatch(logoutSuccess())            [authSlice]
    → queryClient.clear()                  [React Query — wipe all cache]
      → isAuthenticated = false
        → RootNavigator renders Auth Stack

  IF biometricEnabled = true (soft logout):
    → setLoggedIn(false)                   [mmkvStore — the only write]
    → resetTokenRefreshState()             [tokenRefreshManager]
    → dispatch(logoutSuccess())            [authSlice]
    → queryClient.invalidateQueries(['auth', 'session-restore'])  [React Query]
      → isAuthenticated = false
        → RootNavigator renders Auth Stack
      (token + user data remain in storage for next biometric login)
```

### Which Modules Own Which Concerns

| Concern | Owner |
|---|---|
| HTTP transport + auto-refresh interceptor | `client.ts` |
| Token encryption + persistence | `secureStore.ts` |
| User/tenant caching + login gate | `mmkvStore.ts` |
| In-memory auth state + navigation trigger | `authSlice.ts` |
| "Wipe everything from disk" | `authCleanup.ts` |
| Preventing concurrent refresh attempts | `tokenRefreshManager.ts` |
| Login storage write + rollback | `useLoginMutation.ts` |
| Full vs biometric-preserving logout decision | `useLogoutMutation.ts` |
| Cold/warm session restore + integrity check | `useRestoreSessionQuery.ts` |
| "Can the user do biometrics? Should the button show?" | `useBiometrics.ts` |
| Login flow coordination + biometric prompt | `useSignIn.ts` |
| Manual token refresh (biometric login path) | `useTokenRefreshMutation.ts` |
| Auth vs Main stack routing | `RootNavigator.tsx` |
| Login form UI | `SignInScreen/index.tsx` |
| Logout trigger | `CustomDrawerContent.tsx` |

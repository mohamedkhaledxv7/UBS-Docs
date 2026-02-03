# Authentication System Documentation

Complete documentation for the authentication system in the UBS React Native application.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [API Endpoints](#api-endpoints)
- [Storage Strategy](#storage-strategy)
- [Authentication Flows](#authentication-flows)
- [Token Management](#token-management)
- [Error Handling](#error-handling)
- [Security Considerations](#security-considerations)
- [Usage Examples](#usage-examples)

## Overview

The authentication system implements JWT-based authentication with:
- Email/password login
- Automatic token refresh
- Biometric authentication (Face ID, Touch ID, Android biometrics)
- Session persistence across app restarts
- Secure token storage
- Fast user data caching

**Key Technologies**:
- **Redux Toolkit**: Client state management
- **React Query (TanStack Query)**: Server state and mutations
- **Expo SecureStore**: Encrypted token storage
- **MMKV**: Fast, synchronous cache for non-sensitive data
- **Axios**: HTTP client with automatic token refresh

## Architecture

### System Architecture Overview

Complete authentication system architecture showing all layers and their interactions.

```mermaid
graph TB
    subgraph "User Interface Layer"
        SignIn[SignInScreen]
        Home[HomeScreen]
        Settings[SettingsScreen]
    end

    subgraph "State Management Layer"
        Redux[Redux Store<br/>authSlice]
        ReactQuery[React Query<br/>Mutations & Queries]
    end

    subgraph "Feature Layer: src/features/auth/"
        subgraph "API"
            SignInAPI[signInApi.ts]
            RefreshAPI[refreshTokenApi.ts]
            LogoutAPI[logoutApi.ts]
        end
        subgraph "Mutations"
            LoginMut[useLoginMutation]
            RefreshMut[useTokenRefreshMutation]
            LogoutMut[useLogoutMutation]
        end
        subgraph "Queries"
            RestoreQuery[useRestoreSessionQuery]
        end
    end

    subgraph "Infrastructure Layer"
        APIClient[API Client<br/>client.ts<br/>Interceptors]
    end

    subgraph "Storage Layer"
        SecureStore[(SecureStore<br/>Encrypted<br/>Tokens)]
        MMKV[(MMKV<br/>Fast Cache<br/>User/Tenant/Permissions)]
    end

    subgraph "Backend"
        API[Backend API<br/>api.stgtenant.hub-travel.getpayin.com]
    end

    SignIn --> LoginMut
    Home --> LogoutMut
    Settings --> RestoreQuery

    LoginMut --> SignInAPI
    RefreshMut --> RefreshAPI
    LogoutMut --> LogoutAPI
    RestoreQuery --> SecureStore
    RestoreQuery --> MMKV

    SignInAPI --> APIClient
    RefreshAPI --> APIClient
    LogoutAPI --> APIClient

    APIClient --> API

    LoginMut --> SecureStore
    LoginMut --> MMKV
    LoginMut --> Redux

    RefreshMut --> SecureStore
    LogoutMut --> SecureStore
    LogoutMut --> MMKV

    Redux --> SignIn
    Redux --> Home
    Redux --> Settings

    ReactQuery -.manages.-> LoginMut
    ReactQuery -.manages.-> RefreshMut
    ReactQuery -.manages.-> LogoutMut
    ReactQuery -.manages.-> RestoreQuery

    style Redux fill:#95e1d3,color:#000
    style SecureStore fill:#ff6b6b,color:#fff
    style MMKV fill:#4ecdc4,color:#fff
    style API fill:#a8e6cf,color:#000
    style APIClient fill:#ffd3b6,color:#000
```

### Feature-Based Organization

Authentication logic is organized in `src/features/auth/`:

```
src/features/auth/
├── api/                    # API layer
│   ├── signInApi.ts       # POST /v1/auth/login
│   ├── refreshTokenApi.ts # POST /v1/auth/refresh-token
│   ├── logoutApi.ts       # POST /v1/auth/logout
│   └── index.ts
├── mutations/              # React Query mutations
│   ├── useLoginMutation.ts        # Login with rollback on MMKV failure
│   ├── useTokenRefreshMutation.ts # Manual refresh (API client handles automatic)
│   ├── useLogoutMutation.ts       # Logout with error handler for resilience
│   └── index.ts
├── queries/                # React Query queries
│   ├── useRestoreSessionQuery.ts  # Session restore with dispatch guards
│   └── index.ts
└── index.ts                # Feature barrel export
```

### State Management Layers

The authentication system uses a three-layer state management approach:

```mermaid
graph TB
    subgraph "Layer 1: Storage (Persistence)"
        SS[SecureStore<br/>Encrypted<br/>✓ Tokens<br/>✓ Biometric flag]
        MM[MMKV<br/>Fast Cache<br/>✓ User<br/>✓ Tenant<br/>✓ Permissions]
    end

    subgraph "Layer 2: Redux (Client State)"
        Redux[Redux Store - authSlice<br/>Single Source of Truth<br/>user • tenant • permissions<br/>accessToken • isAuthenticated]
    end

    subgraph "Layer 3: React Query (Server State)"
        RQ[React Query<br/>useLoginMutation<br/>useLogoutMutation<br/>useTokenRefreshMutation<br/>useRestoreSessionQuery]
    end

    subgraph "UI Layer"
        Components[React Components<br/>useAppSelector<br/>useAppDispatch]
    end

    SS -->|Load on restore| Redux
    MM -->|Load on restore| Redux
    RQ -->|On success| SS
    RQ -->|On success| MM
    RQ -->|dispatch actions| Redux
    Redux -->|provides state| Components
    Components -->|trigger| RQ

    style SS fill:#ff6b6b,color:#fff
    style MM fill:#4ecdc4,color:#fff
    style Redux fill:#95e1d3,color:#000
    style RQ fill:#ffd3b6,color:#000
```

**Layer Responsibilities**:

1. **Storage Layer** (Persistence)
   - **SecureStore**: Encrypted storage for tokens (async)
   - **MMKV**: Fast cache for user data (synchronous)
   - Data survives app restarts

2. **Redux Layer** (Client State)
   - Single source of truth for app state
   - Drives navigation (Auth Stack vs Main Stack)
   - Synchronous access from any component
   - State: `{ user, tenant, permissions, accessToken, isAuthenticated, isLoading, error }`

3. **React Query Layer** (Server State)
   - Manages API mutations and queries
   - Handles loading/error states
   - Automatic retries and caching
   - Updates storage and Redux on success

## API Endpoints

**Base URL**: `https://api.stgtenant.hub-travel.getpayin.com`

### POST /v1/auth/login

Authenticate user with email and password.

**Request**:
```json
{
  "email": "user@example.com",
  "password": "password123",
  "remember": true,
  "device_name": "iPhone 14 Pro - iOS 17.1"
}
```

**Response** (200 OK):
```json
{
  "data": {
    "access_token": "37|d7Wc5aylMjc0gxyOUXfwfmUxVsM7RgUp4FeychExae762399",
    "token_type": "Bearer",
    "expires_in": 21600,
    "user": {
      "id": 1,
      "name": "John Doe",
      "email": "user@example.com"
    },
    "tenant": {
      "id": "a4ba7a64-d5d8-4a01-b303-02d76d77d0a9",
      "name": "Acme Corp"
    },
    "permissions": [
      "View:Dashboard",
      "ViewAny:Location",
      "Create:Location"
    ]
  }
}
```

**Token Lifetime**: 6 hours (21600 seconds)

**Device Name**: Auto-generated from device info (e.g., "iPhone 14 Pro - iOS 17.1"). Used for session tracking.

### POST /v1/auth/refresh-token

Refresh an expired access token.

**Request**:
- **Headers**: `Authorization: Bearer <current_access_token>`
- **Body**: None

**Response** (200 OK):
```json
{
  "data": {
    "access_token": "39|jT2IS9KyTNSPlJqQqBvUEx1MFrXywMY3Rm0jlgUX7410db85",
    "token_type": "Bearer",
    "expires_in": 21600
  }
}
```

**Note**: Backend uses a single token pattern. The same token is used for both authentication and refresh. When refreshed, a new token is generated.

### POST /v1/auth/logout

Logout user and invalidate token on backend.

**Request**:
- **Headers**: `Authorization: Bearer <access_token>`
- **Body**: None

**Response** (200 OK):
```json
{
  "success": true,
  "status_code": 200,
  "message": "Logged Out"
}
```

**Note**: Even if logout API fails, the app will clear local storage to ensure user can always log out.

## Storage Strategy

### Dual Storage Architecture

The app uses two storage systems for optimal performance and security:

#### 1. Expo SecureStore (Encrypted)

**Location**: [src/services/storage/secureStore.ts](../src/services/storage/secureStore.ts)

**Stored Data**:
- `auth_access_token` - JWT access token (encrypted)
- `auth_refresh_token` - Same as access token (encrypted)
- `biometric_enabled` - User preference boolean (persists across logout/login cycles)
- `user_email` - User's email (for convenience)

**Key Functions**:
- `setTokens(accessToken, refreshToken)` - Store tokens (validates non-empty)
- `getAccessToken()` - Retrieve access token (returns null on error)
- `getRefreshToken()` - Retrieve refresh token (returns null on error)
- `clearTokens()` - Remove tokens
- `clearAllAuthData()` - Remove tokens and email (preserves biometric preference)
- `clearAllAuthDataIncludingPreferences()` - Remove ALL data including biometric preference

**Error Handling**:
- Read operations return null on error (silent fallback)
- Write operations throw for caller handling
- Delete operations log errors but don't throw (non-critical)
- All operations wrapped in try-catch with logging

**Characteristics**:
- ✅ Encrypted and secure
- ✅ Platform-native security (Keychain on iOS, KeyStore on Android)
- ✅ Robust error handling
- ❌ Asynchronous (requires `await`)
- ❌ Slower than MMKV

#### 2. MMKV Storage (Fast Cache)

**Location**: [src/services/storage/mmkvStore.ts](../src/services/storage/mmkvStore.ts)

**Stored Data**:
- `user` - User profile `{ id, name, email }`
- `tenant` - Tenant info `{ id, name }`
- `permissions` - Array of permission strings

**Key Functions**:
- `setUser(user)` - Store user (throws on error for rollback handling)
- `getUser()` - Retrieve user (synchronous, returns null on parse error)
- `setTenant(tenant)` - Store tenant (throws on error for rollback handling)
- `getTenant()` - Retrieve tenant (synchronous, returns null on parse error)
- `setPermissions(permissions)` - Store permissions (throws on error for rollback handling)
- `getPermissions()` - Retrieve permissions (synchronous, returns [] on parse error)
- `clearAuthData()` - Clear all auth data

**Error Handling**:
- Uses `safeJsonParse()` helper for read operations
- JSON parse errors are caught, logged, and corrupted data is auto-cleared
- Write operations throw to allow caller rollback handling
- Prevents app crashes from corrupted storage data

**Characteristics**:
- ✅ Synchronous (no `await` needed)
- ✅ Very fast (10x faster than AsyncStorage)
- ✅ Perfect for non-sensitive cached data
- ✅ Resilient to data corruption
- ❌ Not encrypted
- ❌ Only for non-sensitive data

### Why Dual Storage?

| Requirement | Storage | Reason |
|------------|---------|--------|
| **Security** | SecureStore | Tokens are sensitive and require encryption |
| **Speed** | MMKV | Non-sensitive data accessed synchronously for better UX |
| **Persistence** | Both | Data survives app restarts |
| **Source of Truth** | Redux | Single source for app state, populated from both storages |

### Performance Optimization

**Token Access Strategy**:
- **Write operations** (login, refresh): Update all three layers (SecureStore → MMKV → Redux)
- **Read operations** (API requests): Read from Redux only (synchronous, in-memory)
- **Session restore**: Read from SecureStore + MMKV → populate Redux

**Benefits**:
- API requests are **~100-200ms faster** (no async SecureStore reads)
- Session restore includes **data integrity validation**
- All storage layers stay **perfectly synchronized**
- Security maintained (SecureStore still used for persistence)

### Data Flow

#### Storage Architecture Diagram

```mermaid
graph TB
    subgraph "Login Data Flow"
        API[API Response] --> SecureStore[SecureStore<br/>Encrypted]
        API --> MMKV[MMKV<br/>Fast Cache]
        API --> Redux[Redux Store<br/>Source of Truth]

        SecureStore -.Tokens.-> Redux
        MMKV -.User/Tenant/Permissions.-> Redux
    end

    subgraph "Session Restore"
        App[App Startup] --> Query[useRestoreSessionQuery]
        Query --> SS[SecureStore.getAccessToken]
        Query --> MM[MMKV.getUser/getTenant/getPermissions]
        SS --> ReduxRestore[Redux.restoreSessionSuccess]
        MM --> ReduxRestore
    end

    subgraph "Logout"
        LogoutBtn[Logout Button] --> Mutation[useLogoutMutation]
        Mutation --> LogoutAPI[POST /v1/auth/logout]
        Mutation --> ClearSS[SecureStore.clearAllAuthData]
        Mutation --> ClearMM[MMKV.clearAuthData]
        Mutation --> ClearQuery[QueryClient.clear]
        Mutation --> ReduxLogout[Redux.logoutSuccess]
    end

    style SecureStore fill:#ff6b6b,color:#fff
    style MMKV fill:#4ecdc4,color:#fff
    style Redux fill:#95e1d3,color:#000
```

#### Login Flow Diagram

```mermaid
flowchart LR
    A[API Response] -->|access_token| B[SecureStore]
    A -->|user| C[MMKV]
    A -->|tenant| C
    A -->|permissions| C
    B -->|tokens| D[Redux]
    C -->|cached data| D
    D -->|isAuthenticated: true| E[Navigation]
    E --> F[Main Stack]

    style B fill:#ff6b6b,color:#fff
    style C fill:#4ecdc4,color:#fff
    style D fill:#95e1d3,color:#000
```

#### Session Restore Flow Diagram

```mermaid
flowchart TB
    A[App Startup] --> B[useRestoreSessionQuery]
    B -->|async| C[SecureStore.getAccessToken]
    B -->|sync| D[MMKV.getUser]
    B -->|sync| E[MMKV.getTenant]
    B -->|sync| F[MMKV.getPermissions]
    C --> G{Token exists?}
    G -->|Yes| H[Combine Data]
    D --> H
    E --> H
    F --> H
    H --> V{Data Validation}
    V -->|user && tenant exist| I[Redux.restoreSessionSuccess]
    I --> J[Navigate to Main Stack]
    J --> K[First API call validates token]
    V -->|Missing user or tenant| M[Clear All Auth Data]
    M --> L[Show SignIn Screen]
    G -->|No| L

    style C fill:#ff6b6b,color:#fff
    style D fill:#4ecdc4,color:#fff
    style E fill:#4ecdc4,color:#fff
    style F fill:#4ecdc4,color:#fff
    style V fill:#ffd3b6,color:#000
```

#### Logout Flow Diagram

```mermaid
flowchart TB
    A[User Taps Logout] --> B[useLogoutMutation]
    B --> C[POST /v1/auth/logout]
    C -->|Even if fails| D[Clear Storage]
    D --> E[SecureStore.clearAllAuthData]
    D --> F[MMKV.clearAuthData]
    D --> G[QueryClient.clear]
    E --> H[Redux.logoutSuccess]
    F --> H
    G --> H
    H --> I[Navigate to SignIn]

    style E fill:#ff6b6b,color:#fff
    style F fill:#4ecdc4,color:#fff
```

## Authentication Flows

### 1. Login Flow

Complete sequence diagram showing the login process from user input to navigation.

```mermaid
sequenceDiagram
    actor User
    participant Screen as SignInScreen
    participant Mutation as useLoginMutation
    participant API as Backend API
    participant SecureStore
    participant MMKV
    participant Redux
    participant Nav as Navigation

    User->>Screen: Enter email & password
    User->>Screen: Tap Sign In
    Screen->>Mutation: mutate({ email, password })
    Mutation->>Redux: dispatch(setLoading(true))
    Mutation->>API: POST /v1/auth/login

    alt Success
        API-->>Mutation: 200 OK + { access_token, user, tenant, permissions }
        Mutation->>SecureStore: setTokens(accessToken, accessToken)
        Mutation->>MMKV: setUser(user)
        Mutation->>MMKV: setTenant(tenant)
        Mutation->>MMKV: setPermissions(permissions)
        Mutation->>Redux: dispatch(loginSuccess({ ... }))
        Redux-->>Nav: isAuthenticated = true
        Nav->>Screen: Navigate to Main Stack
        Screen->>User: Show biometric prompt (if supported)
    else Error
        API-->>Mutation: 401 Unauthorized
        Mutation->>Redux: dispatch(loginFailure(message))
        Redux-->>Screen: Update error state
        Screen->>User: Display error message
    end
```

### 2. Session Restore Flow (App Startup)

Shows how the app restores user session on startup using cached data with data integrity validation and dispatch guards.

```mermaid
sequenceDiagram
    participant App as App Startup
    participant RootNav as RootNavigator
    participant Query as useRestoreSessionQuery
    participant SecureStore
    participant MMKV
    participant Redux
    participant Nav as Navigation
    participant API as First API Call

    App->>RootNav: Mount
    RootNav->>Query: Trigger query
    Note over Query: Initialize dispatch guard refs
    Query->>Redux: dispatch(setLoading(true))
    Query->>SecureStore: getAccessToken()

    alt Token exists
        SecureStore-->>Query: return token
        Query->>MMKV: getUser() [sync]
        Query->>MMKV: getTenant() [sync]
        Query->>MMKV: getPermissions() [sync]
        MMKV-->>Query: return cached data
        Query->>SecureStore: getBiometricEnabled()
        SecureStore-->>Query: return biometric preference

        alt Data integrity check passed
            Note over Query: user && tenant exist
            Note over Query: Check hasDispatchedSuccess ref
            Query->>Redux: dispatch(restoreSessionSuccess({ ..., biometricEnabled }))
            Note over Query: Set hasDispatchedSuccess = true
            Redux-->>Nav: isAuthenticated = true
            Nav->>RootNav: Show Main Stack (no flash)

            Note over API: Token validated on first API call
            RootNav->>API: First authenticated request
            alt Token valid
                API-->>RootNav: 200 OK
            else Token expired
                API-->>RootNav: 401 (triggers auto-refresh)
            end
        else Data corrupted
            Note over Query: token exists but user/tenant missing
            Query->>SecureStore: clearAllAuthData()
            Query->>MMKV: clearAuthData()
            Query->>Redux: dispatch(restoreSessionFailure())
            Redux-->>Nav: isAuthenticated = false
            Nav->>RootNav: Show SignIn Screen
        end
    else No token
        SecureStore-->>Query: null
        Query->>Redux: dispatch(restoreSessionFailure())
        Redux-->>Nav: isAuthenticated = false
        Nav->>RootNav: Show SignIn Screen
    end
```

**Dispatch Guards**: The query uses `useRef` to track `hasDispatchedSuccess` and `hasDispatchedFailure`, preventing duplicate Redux dispatches when data reference changes but content is the same. Refs are reset when query starts loading.

### 3. Token Refresh Flow (Automatic)

Demonstrates automatic token refresh when API returns 401 error. This flow ensures complete state synchronization across all storage layers (SecureStore, MMKV, Redux).

```mermaid
sequenceDiagram
    actor User
    participant Component
    participant API as Backend API
    participant Interceptor as Response Interceptor
    participant SecureStore
    participant MMKV
    participant Redux
    participant Queue as Request Queue

    User->>Component: Trigger API call
    Component->>API: Request with token from Redux<br/>(synchronous, fast)
    API-->>Interceptor: 401 Unauthorized

    alt First 401 (not refreshing)
        Interceptor->>Queue: Queue failed request
        Interceptor->>Interceptor: Set isRefreshing = true
        Interceptor->>SecureStore: getRefreshToken()
        SecureStore-->>Interceptor: return token
        Interceptor->>API: POST /v1/auth/refresh-token<br/>Authorization: Bearer token

        alt Refresh success
            API-->>Interceptor: 200 OK + { access_token }
            Interceptor->>SecureStore: setTokens(newToken, newToken)
            Interceptor->>Redux: dispatch(updateAccessToken(newToken))
            Note over Interceptor,Redux: Redux updated for synchronous access
            Interceptor->>Queue: Process queue with new token
            Queue->>API: Retry original request
            API-->>Component: 200 OK (success)
            Component->>User: Show data (no interruption)
        else Refresh failed
            API-->>Interceptor: 401 or 403
            Interceptor->>SecureStore: clearTokens()
            Interceptor->>MMKV: clearAuthData()
            Interceptor->>Redux: dispatch(logoutSuccess())
            Note over Interceptor,Redux: All storage layers cleared
            Redux->>Component: Navigate to SignIn
        end
    else Already refreshing
        Interceptor->>Queue: Add to queue
        Note over Queue: Wait for refresh to complete
        Queue-->>Component: Retry when ready
    end
```

### 4. Logout Flow

Shows the logout process including API call and storage cleanup with error resilience.

```mermaid
sequenceDiagram
    actor User
    participant Screen as Settings/Profile
    participant Mutation as useLogoutMutation
    participant API as Backend API
    participant SecureStore
    participant MMKV
    participant RefreshMgr as TokenRefreshManager
    participant QueryClient
    participant Redux
    participant Nav as Navigation

    User->>Screen: Tap Logout
    Screen->>Screen: Show confirmation alert
    User->>Screen: Confirm
    Screen->>Mutation: mutate()

    Mutation->>API: POST /v1/auth/logout<br/>Authorization: Bearer token

    alt API Success
        API-->>Mutation: 200 OK { success: true }
    else API Fails
        API-->>Mutation: Error (network, 500, etc)
        Note over Mutation: Continue anyway (fail gracefully)
    end

    Mutation->>SecureStore: clearAllAuthData()
    Note over SecureStore: Remove tokens, email<br/>(preserves biometric preference)

    alt SecureStore Success
        SecureStore-->>Mutation: Success
    else SecureStore Fails
        SecureStore-->>Mutation: Error
        Note over Mutation: Log error, continue anyway
    end

    Mutation->>MMKV: clearAuthData()
    Note over MMKV: Remove user, tenant, permissions
    Mutation->>RefreshMgr: resetTokenRefreshState()
    Note over RefreshMgr: Reset refresh state
    Mutation->>QueryClient: clear()
    Note over QueryClient: Clear all cached queries
    Mutation->>Redux: dispatch(logoutSuccess())
    Note over Redux: Called in onSuccess AND onError<br/>(ensures logout even if storage fails)
    Redux-->>Nav: isAuthenticated = false
    Nav->>Screen: Navigate to SignIn Screen
    Screen->>User: Show login screen
```

**Resilience**: The mutation has both `onSuccess` and `onError` handlers that call `finalizeLogout()`, ensuring Redux state is always cleared even if storage operations fail.

## Token Management

### Single Token Pattern

The backend uses a **single token pattern**:
- Only `access_token` is returned (no separate refresh token)
- The same token is used for:
  - Authentication (accessing protected endpoints)
  - Refresh (generating a new token when expired)
- When token expires, call `/v1/auth/refresh-token` with the current token
- Backend validates the token and returns a new one

### Token Lifecycle

```
Token Created (Login)
  ↓
Token Active (6 hours)
  ↓
Token Expires
  ↓
First API Call Returns 401
  ↓
Interceptor Refreshes Token
  ↓
New Token Active (6 hours)
  ↓
... repeat ...
```

**Token Expiration**: 6 hours (21600 seconds)

### Automatic Token Refresh

The API client ([src/services/api/client.ts](../src/services/api/client.ts)) includes optimized request and response interceptors:

**Request Interceptor** (Performance Optimization):
- Reads token from **Redux** (synchronous, in-memory) instead of SecureStore (async, encrypted)
- Provides **~100-200ms performance improvement per API call**
- Maintains security by syncing Redux with SecureStore on every auth operation

**Response Interceptor** (Token Refresh):
1. **Detects 401 errors** from API responses
2. **Skips refresh for refresh endpoint** to prevent infinite loops
3. **Prevents duplicate refreshes** using `TokenRefreshManager` class (encapsulates state)
4. **Queues failed requests** during refresh to prevent race conditions
5. **Uses separate axios instance** for refresh call (bypasses request interceptor to avoid attaching expired token)
6. **Retrieves refresh token from SecureStore** (not Redux, to get the actual stored token)
7. **On success**:
   - Updates SecureStore with new token (persistence)
   - Updates Redux with new token via `updateAccessToken()` (synchronous access)
   - Retries all queued requests with new token
8. **On failure**:
   - Clears SecureStore (tokens only, preserves biometric preference)
   - Clears MMKV (user, tenant, permissions)
   - Dispatches `logoutSuccess()` to Redux
   - Triggers automatic navigation to SignIn screen

**State Synchronization**: All three storage layers (SecureStore, MMKV, Redux) are kept in perfect sync to prevent state desynchronization issues.

**Testing Support**: Export `resetTokenRefreshState()` to reset the refresh manager state between tests or after logout.

### Request Queuing

During token refresh, multiple simultaneous API calls might fail with 401. To prevent multiple refresh attempts:

1. First 401 triggers refresh
2. Subsequent 401s during refresh are **queued**
3. After refresh succeeds, all queued requests **retry** with new token
4. If refresh fails, all queued requests **reject**

This prevents:
- Multiple simultaneous refresh calls
- Race conditions
- Duplicate requests

## Error Handling

### Layered Error Handling

The app uses a **layered approach** to error handling:

#### 1. API Client Layer

**Location**: [src/services/api/client.ts](../src/services/api/client.ts)

**Responsibilities**:
- Catch 401 errors → trigger token refresh
- Handle refresh failures → logout user
- Queue requests during refresh

**Error Format**: Axios errors with `error.response?.data?.message`

#### 2. React Query Layer

**Location**: [src/features/auth/mutations/](../src/features/auth/mutations/)

**Responsibilities**:
- Extract error messages from API responses
- Provide user-friendly fallback messages
- Update Redux with error state

**Example**:
```typescript
onError: (error: any) => {
  const message =
    error.response?.data?.message ||
    error.response?.data?.error ||
    'Login failed. Please try again.';
  dispatch(loginFailure(message));
}
```

#### 3. Redux Layer

**Location**: [src/store/auth/authSlice.ts](../src/store/auth/authSlice.ts)

**Responsibilities**:
- Store error message as `string | null`
- Provide error to components via `useAppSelector`

**Actions**:
- `loginFailure(message)` - Set error, clear loading
- `clearError()` - Clear error message

#### 4. Component Layer

**Responsibilities**:
- Display error messages to user
- Provide retry mechanisms
- Handle UI state (disable buttons, show spinners)

### Common Error Scenarios

| Scenario | HTTP Status | Handling |
|----------|-------------|----------|
| **Invalid credentials** | 401 | Show error message, keep user on SignIn |
| **Network error** | - | Show generic error, allow retry |
| **Token expired** | 401 | Auto-refresh via interceptor, transparent to user |
| **Refresh fails** | 401/403 | Force logout, redirect to SignIn |
| **Logout fails** | Any | Continue with local cleanup, always logout locally |
| **Session restore fails** | - | Clear storage, show SignIn |

## Security Considerations

### Token Storage

✅ **DO**:
- Store tokens in SecureStore (encrypted)
- Use platform-native security (Keychain, KeyStore)
- Clear tokens on logout

❌ **DON'T**:
- Store tokens in AsyncStorage (not encrypted)
- Store tokens in Redux only (lost on app close)
- Log tokens to console in production

### MMKV Storage

✅ **DO**:
- Store non-sensitive data only (user profile, tenant, permissions)
- Clear MMKV on logout
- Validate data from MMKV before use

❌ **DON'T**:
- Store tokens in MMKV (not encrypted)
- Store passwords or PII in MMKV
- Trust MMKV data without validation

### API Communication

✅ **DO**:
- Use HTTPS in production (`https://api.stgtenant.hub-travel.getpayin.com`)
- Send tokens in `Authorization: Bearer` header
- Validate token expiration on backend

❌ **DON'T**:
- Use HTTP in production
- Send tokens in URL query parameters
- Trust client-side token validation only

### Biometric Authentication

✅ **DO**:
- Check device support before enabling
- **Require successful biometric auth before enabling** (proves user can use biometrics)
- Store biometric preference in SecureStore (persists across sessions)
- Fall back to email/password if biometric fails
- Read `hasStoredToken` from Redux (updates immediately on logout)
- Use `isMountedRef` to prevent memory leaks on unmount

❌ **DON'T**:
- Force biometric authentication
- Store biometric data (handled by OS)
- Skip token validation after biometric success
- Clear `biometricEnabled` on logout (it's a user preference)
- Read `hasStoredToken` from SecureStore (becomes stale after logout)

## Usage Examples

### Using Login Mutation

```typescript
import { useLoginMutation } from '../features/auth';

function SignInScreen() {
  const loginMutation = useLoginMutation();
  const { error, isLoading } = useAppSelector(state => state.auth);

  const handleLogin = async () => {
    loginMutation.mutate(
      { email, password },
      {
        onSuccess: () => {
          // Navigation handled automatically by Redux state change
          console.log('Login successful');
        },
        onError: (err) => {
          // Error already dispatched to Redux, just log
          console.error('Login failed:', err);
        },
      }
    );
  };

  return (
    <View>
      <TextInput value={email} onChangeText={setEmail} />
      <TextInput value={password} onChangeText={setPassword} secureTextEntry />

      {error && <Text style={styles.error}>{error}</Text>}

      <Button
        onPress={handleLogin}
        disabled={isLoading}
        title={isLoading ? 'Signing in...' : 'Sign In'}
      />
    </View>
  );
}
```

### Using Logout Mutation

```typescript
import { useLogoutMutation } from '../features/auth';

function SettingsScreen() {
  const logoutMutation = useLogoutMutation();

  const handleLogout = () => {
    Alert.alert(
      'Logout',
      'Are you sure you want to logout?',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Logout',
          onPress: () => logoutMutation.mutate(),
          style: 'destructive',
        },
      ]
    );
  };

  return (
    <View>
      <Button
        onPress={handleLogout}
        title="Logout"
        disabled={logoutMutation.isLoading}
      />
    </View>
  );
}
```

### Accessing Auth State

```typescript
import { useAppSelector } from '../store/hooks';

function ProfileScreen() {
  const { user, tenant, permissions, isAuthenticated } = useAppSelector(
    state => state.auth
  );

  if (!isAuthenticated || !user) {
    return <Text>Not authenticated</Text>;
  }

  const hasPermission = (permission: string) =>
    permissions.includes(permission);

  return (
    <View>
      <Text>Welcome, {user.name}</Text>
      <Text>Email: {user.email}</Text>
      <Text>Tenant: {tenant?.name}</Text>

      {hasPermission('Create:Hotels') && (
        <Button title="Create Hotel" onPress={handleCreate} />
      )}
    </View>
  );
}
```

### Session Restore

```typescript
// In RootNavigator.tsx
import { useRestoreSessionQuery } from '../features/auth';

function RootNavigator() {
  const { isLoading } = useRestoreSessionQuery();
  const { isAuthenticated } = useAppSelector(state => state.auth);

  if (isLoading) {
    return <SplashScreen />;
  }

  return (
    <NavigationContainer>
      <Stack.Navigator>
        {isAuthenticated ? (
          <Stack.Screen name="Main" component={MainStack} />
        ) : (
          <Stack.Screen name="Auth" component={AuthStack} />
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

---

**For more information**:
- See [CLAUDE.md](../CLAUDE.md) for project overview
- See [MIGRATION_MOCK_TO_REAL_API.md](./MIGRATION_MOCK_TO_REAL_API.md) for migration guide
- Source code: [src/features/auth/](../src/features/auth/)

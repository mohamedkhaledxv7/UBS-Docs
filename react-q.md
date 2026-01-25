# React Query Auth Migration Plan (with Redux)

## Executive Summary

Migrate authentication API calls from Redux AsyncThunk to React Query mutations while **keeping Redux authSlice for state management**. This approach uses React Query for what it does best (server state/API calls) and Redux for what it does best (client state management).

**Architecture: React Query + Redux (Simplified Hybrid)**

---

## 1. Architecture Decision

### Chosen Approach: React Query for API + Redux for State

**React Query handles (Server State):**
- Login mutation - API call to backend
- Logout mutation - Optional API call (if backend logout endpoint exists)
- Token refresh mutation - NEW FEATURE
- Session restore query - Validate token on startup

**Redux authSlice handles (Client State):**
- All UI state: `user`, `accessToken`, `isAuthenticated`, `isLoading`, `error`, `biometricEnabled`
- State updates via reducers (no AsyncThunks)
- Selectors for components
- Single source of truth for auth state

**Integration Pattern:**
```
Component → React Query Mutation → API Call → Redux Dispatch → State Update → Component Re-render
```

### Why This Approach?

✅ **Minimal Migration**: Keep existing Redux structure, only change API layer
✅ **Separation of Concerns**: React Query = API, Redux = State
✅ **Incremental Adoption**: Can migrate one mutation at a time
✅ **Keep Existing Patterns**: Components already use Redux selectors
✅ **No Context Needed**: Redux already provides global state
✅ **Type Safety**: Existing TypeScript types remain valid
✅ **Backward Compatible**: Easy rollback if needed

### Why Not Just Keep AsyncThunk?

- AsyncThunk couples API calls with state management
- React Query provides better retry logic, caching, and devtools
- Automatic token refresh easier with React Query
- Want to use React Query patterns across the app consistently

---

## 2. File Structure Changes

```
src/
├── hooks/
│   ├── auth/
│   │   ├── useLoginMutation.ts         # NEW: Login API mutation
│   │   ├── useLogoutMutation.ts        # NEW: Logout API mutation
│   │   ├── useRestoreSessionQuery.ts   # NEW: Session restore query
│   │   └── useTokenRefreshMutation.ts  # NEW: Token refresh mutation
│   └── useBiometrics.ts                # MODIFY: Minor updates
├── services/
│   └── api/
│       ├── client.ts                   # MODIFY: Add token refresh interceptor
│       └── auth.ts                     # UNCHANGED
├── screens/
│   ├── auth/SignInScreen/
│   │   └── useSignIn.ts                # MODIFY: Use new mutation hooks
│   └── main/HomeScreen/
│       └── index.tsx                   # MODIFY: Use new mutation hooks
├── navigation/
│   └── RootNavigator.tsx               # MODIFY: Use new query hook
└── store/
    └── slices/
        └── authSlice.ts                # MODIFY: Remove AsyncThunks, add reducers
```

**Key Point**: No new contexts, no major structural changes!

---

## 3. Implementation Steps

### Step 1: Modify authSlice - Remove AsyncThunks, Add Reducers

**File**: `src/store/slices/authSlice.ts` (MODIFY)

**Current Structure**:
- Has AsyncThunks: `login`, `logout`, `restoreSession`, `enableBiometric`
- Has extraReducers for handling AsyncThunk states

**New Structure**:
- Remove all AsyncThunks
- Convert to regular reducers
- Keep all state shape exactly the same

**Changes**:

```typescript
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { AuthState, User, AuthResponse } from '../../types/auth';

const initialState: AuthState = {
  user: null,
  accessToken: null,
  isAuthenticated: false,
  isLoading: false,
  error: null,
  biometricEnabled: false,
};

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    // Loading state
    setLoading: (state, action: PayloadAction<boolean>) => {
      state.isLoading = action.payload;
    },

    // Login success
    loginSuccess: (state, action: PayloadAction<AuthResponse>) => {
      state.isLoading = false;
      state.isAuthenticated = true;
      state.accessToken = action.payload.accessToken;
      state.user = action.payload.user;
      state.error = null;
    },

    // Login failure
    loginFailure: (state, action: PayloadAction<string>) => {
      state.isLoading = false;
      state.error = action.payload;
    },

    // Restore session success
    restoreSessionSuccess: (state, action: PayloadAction<{ accessToken: string; user?: User }>) => {
      state.isLoading = false;
      state.isAuthenticated = true;
      state.accessToken = action.payload.accessToken;
      if (action.payload.user) {
        state.user = action.payload.user;
      }
    },

    // Restore session failure
    restoreSessionFailure: (state) => {
      state.isLoading = false;
      state.isAuthenticated = false;
    },

    // Logout
    logoutSuccess: (state) => {
      state.user = null;
      state.accessToken = null;
      state.isAuthenticated = false;
      state.biometricEnabled = false;
      state.error = null;
    },

    // Update access token (for refresh)
    updateAccessToken: (state, action: PayloadAction<string>) => {
      state.accessToken = action.payload;
    },

    // Biometric
    setBiometricEnabled: (state, action: PayloadAction<boolean>) => {
      state.biometricEnabled = action.payload;
    },

    // Error handling
    clearError: (state) => {
      state.error = null;
    },

    // Manual setters (keep existing)
    setUser: (state, action: PayloadAction<User>) => {
      state.user = action.payload;
    },

    setAuthenticated: (state, action: PayloadAction<boolean>) => {
      state.isAuthenticated = action.payload;
    },
  },
});

export const {
  setLoading,
  loginSuccess,
  loginFailure,
  restoreSessionSuccess,
  restoreSessionFailure,
  logoutSuccess,
  updateAccessToken,
  setBiometricEnabled,
  clearError,
  setUser,
  setAuthenticated,
} = authSlice.actions;

export default authSlice.reducer;
```

**Key Changes**:
1. ❌ Remove: All `createAsyncThunk` calls
2. ❌ Remove: `extraReducers` section
3. ✅ Add: New reducers for each state change
4. ✅ Keep: All existing state shape
5. ✅ Keep: `clearError`, `setUser`, `setAuthenticated` reducers

---

### Step 2: Create Login Mutation Hook

**File**: `src/hooks/auth/useLoginMutation.ts` (NEW)

```typescript
import { useMutation } from '@tanstack/react-query';
import { authApi } from '@/services/api/auth';
import { setTokens, setStoredEmail } from '@/services/storage/secureStore';
import { useAppDispatch } from '@/store/hooks';
import { setLoading, loginSuccess, loginFailure } from '@/store/slices/authSlice';
import type { LoginCredentials } from '@/types/auth';

export const useLoginMutation = () => {
  const dispatch = useAppDispatch();

  return useMutation({
    mutationFn: async (credentials: LoginCredentials) => {
      // Set loading state
      dispatch(setLoading(true));

      const response = await authApi.login(credentials);

      // Save to secure storage
      await setTokens(response.accessToken, response.refreshToken);
      await setStoredEmail(credentials.email);

      return response;
    },
    onSuccess: (data) => {
      // Update Redux state
      dispatch(loginSuccess(data));
    },
    onError: (error: any) => {
      const message = error.response?.data?.message || 'Login failed. Please try again.';
      dispatch(loginFailure(message));
    },
  });
};
```

**How it works**:
1. Component calls `mutation.mutate(credentials)`
2. React Query executes the API call
3. On success: dispatch `loginSuccess` to Redux
4. On error: dispatch `loginFailure` to Redux
5. Component reads state from Redux selectors (unchanged)

---

### Step 3: Create Logout Mutation Hook

**File**: `src/hooks/auth/useLogoutMutation.ts` (NEW)

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { clearTokens } from '@/services/storage/secureStore';
import { useAppDispatch } from '@/store/hooks';
import { logoutSuccess } from '@/store/slices/authSlice';

export const useLogoutMutation = () => {
  const dispatch = useAppDispatch();
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async () => {
      // Clear secure storage
      await clearTokens();

      // Optional: Call backend logout endpoint if exists
      // await authApi.logout();
    },
    onSuccess: () => {
      // Clear Redux state
      dispatch(logoutSuccess());

      // Clear React Query cache (security best practice)
      queryClient.clear();
    },
  });
};
```

**Features**:
- Clears tokens from secure storage
- Dispatches Redux action to reset state
- Clears all React Query cache

---

### Step 4: Create Session Restore Query Hook

**File**: `src/hooks/auth/useRestoreSessionQuery.ts` (NEW)

```typescript
import { useQuery } from '@tanstack/react-query';
import { getAccessToken, clearTokens } from '@/services/storage/secureStore';
import { authApi } from '@/services/api/auth';
import { useAppDispatch } from '@/store/hooks';
import { setLoading, restoreSessionSuccess, restoreSessionFailure } from '@/store/slices/authSlice';

export const useRestoreSessionQuery = () => {
  const dispatch = useAppDispatch();

  return useQuery({
    queryKey: ['auth', 'session', 'restore'],
    queryFn: async () => {
      dispatch(setLoading(true));

      const token = await getAccessToken();

      if (!token) {
        throw new Error('No stored token');
      }

      // Validate token with backend
      const isValid = await authApi.validateToken();

      if (!isValid) {
        await clearTokens();
        throw new Error('Token expired');
      }

      return { accessToken: token };
    },
    retry: false,
    staleTime: Infinity,
    refetchOnMount: false,
    refetchOnWindowFocus: false,
    onSuccess: (data) => {
      dispatch(restoreSessionSuccess(data));
    },
    onError: () => {
      dispatch(restoreSessionFailure());
    },
  });
};
```

**Configuration**:
- `staleTime: Infinity` - Don't refetch automatically
- `retry: false` - Don't retry if token is invalid
- `refetchOnMount: false` - Only run once on app startup
- Dispatches to Redux on success/failure

---

### Step 5: Create Token Refresh Mutation Hook

**File**: `src/hooks/auth/useTokenRefreshMutation.ts` (NEW)

**NEW FEATURE** - Automatic token refresh

```typescript
import { useMutation } from '@tanstack/react-query';
import { authApi } from '@/services/api/auth';
import { getRefreshToken, setTokens, clearTokens } from '@/services/storage/secureStore';
import { useAppDispatch } from '@/store/hooks';
import { updateAccessToken, logoutSuccess } from '@/store/slices/authSlice';

export const useTokenRefreshMutation = () => {
  const dispatch = useAppDispatch();

  return useMutation({
    mutationFn: async () => {
      const refreshToken = await getRefreshToken();

      if (!refreshToken) {
        throw new Error('No refresh token');
      }

      const response = await authApi.refreshToken(refreshToken);

      // Update stored tokens
      await setTokens(response.accessToken, response.refreshToken);

      return response;
    },
    onSuccess: (data) => {
      // Update access token in Redux
      dispatch(updateAccessToken(data.accessToken));
    },
    onError: async () => {
      // Refresh failed, force logout
      await clearTokens();
      dispatch(logoutSuccess());
    },
  });
};
```

---

### Step 6: Update API Client with Token Refresh Interceptor

**File**: `src/services/api/client.ts` (MODIFY)

Add automatic token refresh on 401 errors:

```typescript
import axios from 'axios';
import { getAccessToken, getRefreshToken, setTokens, clearTokens } from '../storage/secureStore';
import { authApi } from './auth';

const API_URL = process.env.EXPO_PUBLIC_API_URL || 'http://localhost:3000/api';

export const apiClient = axios.create({
  baseURL: API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor - attach token
apiClient.interceptors.request.use(
  async (config) => {
    const token = await getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor - handle token refresh
let isRefreshing = false;
let failedQueue: Array<{
  resolve: (value?: unknown) => void;
  reject: (reason?: any) => void;
}> = [];

const processQueue = (error: any, token: string | null = null) => {
  failedQueue.forEach((prom) => {
    if (error) {
      prom.reject(error);
    } else {
      prom.resolve(token);
    }
  });
  failedQueue = [];
};

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // If 401 and not already retrying
    if (error.response?.status === 401 && !originalRequest._retry) {
      // If already refreshing, queue this request
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        })
          .then((token) => {
            originalRequest.headers.Authorization = `Bearer ${token}`;
            return apiClient(originalRequest);
          })
          .catch((err) => Promise.reject(err));
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = await getRefreshToken();

        if (!refreshToken) {
          throw new Error('No refresh token');
        }

        const response = await authApi.refreshToken(refreshToken);
        const { accessToken } = response;

        await setTokens(accessToken, response.refreshToken);

        // Update all queued requests
        processQueue(null, accessToken);

        // Retry original request
        originalRequest.headers.Authorization = `Bearer ${accessToken}`;
        return apiClient(originalRequest);
      } catch (err) {
        processQueue(err, null);
        await clearTokens();

        // Note: Can't dispatch Redux action here directly
        // The logout will be handled by the mutation hook or navigation
        return Promise.reject(err);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

**Key Features**:
- Automatic retry on 401
- Queue concurrent requests during refresh
- Fallback to logout if refresh fails

---

### Step 7: Update useSignIn Hook

**File**: `src/screens/auth/SignInScreen/useSignIn.ts` (MODIFY)

**Before** (Redux AsyncThunk):
```typescript
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { login, clearError, enableBiometric, setAuthenticated } from '@/store/slices/authSlice';

export const useSignIn = () => {
  const dispatch = useAppDispatch();
  const { isLoading, error, biometricEnabled } = useAppSelector((state) => state.auth);

  const handleLogin = async (credentials: LoginCredentials) => {
    const result = await dispatch(login(credentials));
    if (login.fulfilled.match(result)) {
      // Success handling
    }
  };

  const handleClearError = () => {
    dispatch(clearError());
  };

  return {
    handleLogin,
    isLoading,
    error,
    handleClearError,
    // ... biometric stuff
  };
};
```

**After** (React Query + Redux):
```typescript
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { useLoginMutation } from '@/hooks/auth/useLoginMutation';
import { clearError, setAuthenticated } from '@/store/slices/authSlice';

export const useSignIn = () => {
  const dispatch = useAppDispatch();

  // Read state from Redux (unchanged)
  const { error, biometricEnabled } = useAppSelector((state) => state.auth);

  // Use React Query mutation for API call
  const loginMutation = useLoginMutation();

  const handleLogin = (credentials: LoginCredentials) => {
    loginMutation.mutate(credentials, {
      onSuccess: () => {
        // Biometric setup prompt if needed
        if (biometricSupported && !biometricEnabled) {
          promptBiometricSetup();
        }
      },
    });
  };

  const handleClearError = () => {
    dispatch(clearError());
  };

  return {
    handleLogin,
    isLoading: loginMutation.isPending, // From React Query
    error, // From Redux
    handleClearError,
    // ... biometric stuff
  };
};
```

**Key Changes**:
- ✅ Use `useLoginMutation()` instead of dispatching AsyncThunk
- ✅ `isLoading` from `mutation.isPending`
- ✅ `error` still from Redux
- ✅ Keep `dispatch(clearError())` unchanged
- ✅ Keep all other Redux selectors unchanged

---

### Step 8: Update RootNavigator

**File**: `src/navigation/RootNavigator.tsx` (MODIFY)

**Before** (Redux AsyncThunk):
```typescript
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { restoreSession } from '@/store/slices/authSlice';

export const RootNavigator = () => {
  const dispatch = useAppDispatch();
  const { isAuthenticated, isLoading } = useAppSelector((state) => state.auth);

  useEffect(() => {
    dispatch(restoreSession());
  }, []);

  if (isLoading) {
    return <LoadingScreen />;
  }

  return (
    <NavigationContainer>
      {isAuthenticated ? <MainStack /> : <AuthStack />}
    </NavigationContainer>
  );
};
```

**After** (React Query + Redux):
```typescript
import { useAppSelector } from '@/store/hooks';
import { useRestoreSessionQuery } from '@/hooks/auth/useRestoreSessionQuery';

export const RootNavigator = () => {
  // Trigger session restore (runs automatically)
  const { isLoading } = useRestoreSessionQuery();

  // Read auth state from Redux (unchanged)
  const { isAuthenticated } = useAppSelector((state) => state.auth);

  if (isLoading) {
    return <LoadingScreen />;
  }

  return (
    <NavigationContainer>
      {isAuthenticated ? <MainStack /> : <AuthStack />}
    </NavigationContainer>
  );
};
```

**Key Changes**:
- ✅ Replace `useEffect + dispatch` with `useRestoreSessionQuery()`
- ✅ `isLoading` from React Query
- ✅ `isAuthenticated` from Redux (unchanged)
- ✅ No manual trigger needed - query runs on mount

---

### Step 9: Update HomeScreen Logout

**File**: `src/screens/main/HomeScreen/index.tsx` (MODIFY)

**Before**:
```typescript
import { useAppDispatch } from '@/store/hooks';
import { logout } from '@/store/slices/authSlice';

const HomeScreen = () => {
  const dispatch = useAppDispatch();

  const handleLogout = () => {
    dispatch(logout());
  };

  return (
    <Button onPress={handleLogout} title="Logout" />
  );
};
```

**After**:
```typescript
import { useLogoutMutation } from '@/hooks/auth/useLogoutMutation';

const HomeScreen = () => {
  const logoutMutation = useLogoutMutation();

  const handleLogout = () => {
    logoutMutation.mutate();
  };

  return (
    <Button onPress={handleLogout} title="Logout" />
  );
};
```

**Key Changes**:
- ✅ Use `useLogoutMutation()` instead of dispatch
- ✅ No need to import Redux hooks
- ✅ Optional: Can show loading with `logoutMutation.isPending`

---

### Step 10: Update Biometric Hook (Minor)

**File**: `src/hooks/useBiometrics.ts` (MODIFY)

The biometric hook uses `enableBiometric` AsyncThunk. Convert to use the new Redux action:

**Before**:
```typescript
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { enableBiometric } from '@/store/slices/authSlice';

const dispatch = useAppDispatch();

const enableBiometricLogin = async () => {
  await setBiometricEnabled(true); // secure storage
  await dispatch(enableBiometric(true)); // redux
};
```

**After**:
```typescript
import { useAppDispatch, useAppSelector } from '@/store/hooks';
import { setBiometricEnabled as setBiometricEnabledAction } from '@/store/slices/authSlice';
import { setBiometricEnabled } from '@/services/storage/secureStore';

const dispatch = useAppDispatch();

const enableBiometricLogin = async () => {
  await setBiometricEnabled(true); // secure storage
  dispatch(setBiometricEnabledAction(true)); // redux action (not AsyncThunk)
};
```

**Key Changes**:
- ✅ Use regular Redux action instead of AsyncThunk
- ✅ No API call needed (it's just local storage)

---

## 4. Migration Checklist

### Phase 1: Update Redux Slice
- [ ] Modify `src/store/slices/authSlice.ts`
  - [ ] Remove all `createAsyncThunk` calls
  - [ ] Remove `extraReducers` section
  - [ ] Add new reducers for each state change
  - [ ] Export all new action creators

### Phase 2: Create React Query Hooks
- [ ] Create `src/hooks/auth/useLoginMutation.ts`
- [ ] Create `src/hooks/auth/useLogoutMutation.ts`
- [ ] Create `src/hooks/auth/useRestoreSessionQuery.ts`
- [ ] Create `src/hooks/auth/useTokenRefreshMutation.ts`

### Phase 3: Update Components
- [ ] Update `src/screens/auth/SignInScreen/useSignIn.ts`
- [ ] Update `src/navigation/RootNavigator.tsx`
- [ ] Update `src/screens/main/HomeScreen/index.tsx`
- [ ] Update `src/hooks/useBiometrics.ts`

### Phase 4: Implement Token Refresh
- [ ] Update `src/services/api/client.ts` with interceptor
- [ ] Test automatic token refresh on 401 errors
- [ ] Test refresh token expiration handling

### Phase 5: Testing
- [ ] Test login flow
- [ ] Test logout flow
- [ ] Test session restore on app startup
- [ ] Test biometric login
- [ ] Test token refresh
- [ ] Test navigation guards
- [ ] Test with mock API
- [ ] Test error handling

### Phase 6: Cleanup
- [ ] Remove unused imports
- [ ] Update TypeScript types if needed
- [ ] Test on iOS and Android

---

## 5. Comparison: Before vs After

### Before (AsyncThunk)
```typescript
// In component
const dispatch = useAppDispatch();
const { isLoading, error } = useAppSelector(state => state.auth);

const handleLogin = async (creds) => {
  const result = await dispatch(login(creds));
  if (login.fulfilled.match(result)) {
    // success
  }
};
```

### After (React Query + Redux)
```typescript
// In component
const loginMutation = useLoginMutation();
const { error } = useAppSelector(state => state.auth);

const handleLogin = (creds) => {
  loginMutation.mutate(creds, {
    onSuccess: () => {
      // success
    }
  });
};

const isLoading = loginMutation.isPending;
```

**Benefits**:
- Cleaner mutation syntax
- Better loading state from React Query
- Error state still in Redux (single source of truth)
- Easy to add onSuccess/onError callbacks

---

## 6. Key Benefits

✅ **Minimal Changes**: Components mostly unchanged, only swap hooks
✅ **Keep Redux**: All existing selectors, state shape preserved
✅ **Better API Layer**: React Query handles retries, caching, devtools
✅ **Automatic Token Refresh**: New feature with clean implementation
✅ **Type Safety**: Existing types remain valid
✅ **Incremental**: Can migrate one mutation at a time
✅ **Rollback Easy**: Can revert to AsyncThunk quickly if needed

---

## 7. Potential Challenges & Solutions

### Challenge 1: Duplicate Loading State?

**Issue**: React Query has `isPending`, Redux has `isLoading`

**Solution**:
- Option A: Use React Query's `isPending` for UI, keep Redux `isLoading` for legacy compatibility
- Option B: Remove Redux `isLoading`, only use `isPending`
- **Recommendation**: Option A initially, migrate to Option B later

### Challenge 2: Error State Location

**Issue**: React Query has error, Redux has error

**Solution**:
- Keep error in Redux for consistency with existing code
- Mutation hooks dispatch error to Redux
- Components read from Redux as before

### Challenge 3: Token Refresh Interceptor Without Hooks

**Issue**: Axios interceptor can't use React hooks

**Solution**:
- Interceptor handles refresh directly with API calls
- Updates secure storage
- Redux state gets updated on next request or via event
- **Alternative**: Use a global event emitter to notify app of token refresh

---

## 8. File Change Summary

| File | Action | Changes |
|------|--------|---------|
| `src/store/slices/authSlice.ts` | MODIFY | Remove AsyncThunks, add reducers |
| `src/hooks/auth/useLoginMutation.ts` | CREATE | Login mutation with Redux dispatch |
| `src/hooks/auth/useLogoutMutation.ts` | CREATE | Logout mutation with Redux dispatch |
| `src/hooks/auth/useRestoreSessionQuery.ts` | CREATE | Session restore with Redux dispatch |
| `src/hooks/auth/useTokenRefreshMutation.ts` | CREATE | Token refresh with Redux dispatch |
| `src/services/api/client.ts` | MODIFY | Add token refresh interceptor |
| `src/screens/auth/SignInScreen/useSignIn.ts` | MODIFY | Use mutation hook instead of AsyncThunk |
| `src/navigation/RootNavigator.tsx` | MODIFY | Use query hook instead of AsyncThunk |
| `src/screens/main/HomeScreen/index.tsx` | MODIFY | Use mutation hook instead of AsyncThunk |
| `src/hooks/useBiometrics.ts` | MODIFY | Use regular action instead of AsyncThunk |

**Total: 10 files**
**New: 4 | Modified: 6**

---

## 9. Testing Strategy

### Unit Tests
- Test mutation hooks with React Query testing library
- Test Redux reducers (same as before)
- Mock API responses

### Integration Tests
- Test login → state update → navigation flow
- Test session restore → state update flow
- Test logout → state clear → navigation flow

### Manual Testing
- Test on real devices
- Test token refresh scenarios
- Test with slow network
- Test error cases

---

## 10. Rollback Plan

If issues arise:

1. **Quick Rollback**: Git revert all changes
2. **Keep New Code**: Add feature flag to toggle between AsyncThunk and React Query
3. **Partial Rollback**: Migrate one mutation at a time, keep others on AsyncThunk

---

## Conclusion

This approach provides the best of both worlds:
- React Query for robust API handling
- Redux for familiar state management
- Minimal migration effort
- Easy rollback path

The key insight: **React Query doesn't replace Redux for auth state - it replaces AsyncThunk for API calls.**

**Next Steps**: Approve this approach and proceed with implementation.

# Identity Resolution API

The Identity Resolution API handles secure token-based authentication and session management across multiple platforms (AniList, MyAnimeList, SIMKL).

## 🔒 Security Overview

**CRITICAL**: This API uses token-based authentication to prevent identity spoofing. Unlike previous versions, it no longer trusts client-provided user IDs.

## Endpoint

```
POST /functions/v1/identity-resolve
```

## Request Body

```typescript
interface IdentityRequest {
  client_type: 'anilist' | 'myanimelist' | 'simkl'
  token: string
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_type` | string | Yes | The platform identifier ('anilist', 'myanimelist', 'simkl') |
| `token` | string | Yes | OAuth token from the external platform |

## Response

### Success Response (200 OK)

```typescript
interface IdentityResponse {
  session_token: string
  expires_at: string
  user: {
    id: number
    username: string
    role: 'user' | 'moderator' | 'admin' | 'super_admin'
    banned: boolean
    shadow_banned: boolean
    muted_until: string | null
  }
  existing_user: boolean
}
```

### Error Responses

#### 400 Bad Request
```json
{
  "error": "Missing required fields: client_type, token"
}
```

#### 401 Unauthorized
```json
{
  "error": "Invalid token or authentication failed"
}
```

#### 403 Forbidden
```json
{
  "error": "User banned or muted",
  "banned": true,
  "muted_until": "2024-01-01T00:00:00Z"
}
```

#### 500 Internal Server Error
```json
{
  "error": "Internal server error"
}
```

## 🔐 Authentication Flow

### 1. Token Verification

The API verifies the provided token with the respective platform:

#### AniList Token Verification
```graphql
query {
  Viewer {
    id
    name
    avatar {
      large
    }
  }
}
```

#### MyAnimeList Token Verification
```http
GET https://api.myanimelist.net/v2/users/@me
Authorization: Bearer <token>
```

#### SIMKL Token Verification
```http
GET https://api.simkl.com/users/settings
simkl-api-key: <token>
```

### 2. Session Creation

After successful token verification, the API creates a secure session:

```typescript
// Generate cryptographically secure session token
function generateSessionToken(): string {
  const array = new Uint8Array(32)
  crypto.getRandomValues(array)
  return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
}

// 30-day expiry
const expiresAt = new Date()
expiresAt.setDate(expiresAt.getDate() + 30)
```

### 3. User Resolution Flow

1. **Token Verification**: Verify token with external platform API
2. **Identity Lookup**: Check if identity already exists for the platform and user ID
3. **User Status Validation**: Check if user is banned or muted
4. **Session Creation**: Create new session or update existing one
5. **Response**: Return session token and user information

## 🚀 Usage Examples

### Basic Authentication

```javascript
const authenticateUser = async (clientType, token) => {
  const response = await fetch('/functions/v1/identity-resolve', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${SUPABASE_ANON_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      client_type: clientType,
      token
    })
  })

  const result = await response.json()
  
  if (result.error) {
    throw new Error(result.error)
  }

  // Store session token for subsequent API calls
  localStorage.setItem('commentum_session_token', result.session_token)
  
  return result.user
}

// Example usage with AniList
try {
  const user = await authenticateUser('anilist', 'your_anilist_oauth_token')
  console.log('Authenticated as:', user.username)
  console.log('Session expires:', result.expires_at)
} catch (error) {
  console.error('Authentication failed:', error.message)
}
```

### React Hook Implementation

```typescript
import { useState, useCallback } from 'react'

interface AuthState {
  user: UserIdentity | null
  sessionToken: string | null
  loading: boolean
  error: string | null
  isAuthenticated: boolean
}

export const useAuth = () => {
  const [state, setState] = useState<AuthState>({
    user: null,
    sessionToken: localStorage.getItem('commentum_session_token'),
    loading: false,
    error: null,
    isAuthenticated: false
  })

  const login = useCallback(async (clientType: string, token: string) => {
    setState(prev => ({ ...prev, loading: true, error: null }))

    try {
      const response = await fetch('/functions/v1/identity-resolve', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          client_type: clientType,
          token
        })
      })

      const result = await response.json()
      
      if (result.error) {
        throw new Error(result.error)
      }

      // Store session
      localStorage.setItem('commentum_session_token', result.session_token)
      
      setState({
        user: result.user,
        sessionToken: result.session_token,
        loading: false,
        error: null,
        isAuthenticated: true
      })
    } catch (error) {
      setState(prev => ({
        ...prev,
        loading: false,
        error: error instanceof Error ? error.message : 'Authentication failed'
      }))
    }
  }, [])

  const logout = useCallback(() => {
    localStorage.removeItem('commentum_session_token')
    setState({
      user: null,
      sessionToken: null,
      loading: false,
      error: null,
      isAuthenticated: false
    })
  }, [])

  return {
    ...state,
    login,
    logout
  }
}
```

### Vue.js Implementation

```typescript
import { ref, computed } from 'vue'

export const useAuth = () => {
  const sessionToken = ref(localStorage.getItem('commentum_session_token'))
  const user = ref(null)
  const loading = ref(false)
  const error = ref(null)

  const isAuthenticated = computed(() => !!sessionToken.value && !!user.value)

  const login = async (clientType: string, token: string) => {
    loading.value = true
    error.value = null

    try {
      const response = await fetch('/functions/v1/identity-resolve', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${import.meta.env.VITE_SUPABASE_ANON_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          client_type: clientType,
          token
        })
      })

      const result = await response.json()
      
      if (result.error) {
        throw new Error(result.error)
      }

      sessionToken.value = result.session_token
      user.value = result.user
      localStorage.setItem('commentum_session_token', result.session_token)
    } catch (err) {
      error.value = err instanceof Error ? err.message : 'Authentication failed'
    } finally {
      loading.value = false
    }
  }

  const logout = () => {
    sessionToken.value = null
    user.value = null
    localStorage.removeItem('commentum_session_token')
  }

  return {
    sessionToken,
    user,
    loading,
    error,
    isAuthenticated,
    login,
    logout
  }
}
```

### Vanilla JavaScript Implementation

```javascript
class CommentumAuth {
  constructor() {
    this.sessionToken = localStorage.getItem('commentum_session_token')
    this.user = null
  }

  async authenticate(clientType, token) {
    try {
      const response = await fetch('/functions/v1/identity-resolve', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${SUPABASE_ANON_KEY}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          client_type: clientType,
          token
        })
      })

      const result = await response.json()
      
      if (result.error) {
        throw new Error(result.error)
      }

      this.sessionToken = result.session_token
      this.user = result.user
      localStorage.setItem('commentum_session_token', result.session_token)
      
      return result.user
    } catch (error) {
      console.error('Authentication failed:', error.message)
      throw error
    }
  }

  logout() {
    this.sessionToken = null
    this.user = null
    localStorage.removeItem('commentum_session_token')
  }

  isAuthenticated() {
    return !!this.sessionToken && !!this.user
  }

  makeAuthenticatedRequest(endpoint, options = {}) {
    if (!this.sessionToken) {
      throw new Error('Not authenticated')
    }

    return fetch(endpoint, {
      ...options,
      headers: {
        'Authorization': `Bearer ${this.sessionToken}`,
        'Content-Type': 'application/json',
        ...options.headers
      }
    })
  }
}

// Usage
const auth = new CommentumAuth()

// Authenticate
try {
  await auth.authenticate('anilist', 'your_oauth_token')
  console.log('Authenticated successfully!')
  
  // Make authenticated request
  const response = await auth.makeAuthenticatedRequest('/functions/v1/comments', {
    method: 'POST',
    body: JSON.stringify({
      action: 'create',
      media_info: {
        external_id: '12345',
        media_type: 'anime',
        title: 'Attack on Titan'
      },
      content: 'Great anime!'
    })
  })
  
  console.log('Comment created:', response)
} catch (error) {
  console.error('Error:', error.message)
}
```

## 🔒 Security Features

### Token Verification
- ✅ All tokens are verified with respective provider APIs
- ✅ Invalid or expired tokens are rejected
- ✅ Real user information is fetched from providers

### Session Management
- ✅ Cryptographically secure session tokens
- ✅ 30-day automatic expiry
- ✅ Server-side session validation
- ✅ Automatic cleanup of expired sessions

### Identity Protection
- ✅ Prevents user ID spoofing attacks
- ✅ Zero-trust architecture
- ✅ Real authentication on every request

## 📊 Session Management

### Session Storage
```sql
CREATE TABLE user_sessions (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  session_token TEXT NOT NULL UNIQUE,
  client_type TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  last_used_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Session Lifecycle
1. **Creation**: Generated after successful token verification
2. **Validation**: Verified on every API request
3. **Expiry**: Automatically expires after 30 days
4. **Cleanup**: Expired sessions are automatically removed

## 🚨 Breaking Changes from Previous Version

### Before (Vulnerable)
```javascript
// ❌ Client could send any user_id
const response = await fetch('/functions/v1/identity-resolve', {
  body: JSON.stringify({
    client_type: 'anilist',
    client_user_id: '12345',  // Could be faked!
    username: 'fake_user'
  })
})
```

### After (Secure)
```javascript
// ✅ Token must be verified with provider
const response = await fetch('/functions/v1/identity-resolve', {
  body: JSON.stringify({
    client_type: 'anilist',
    token: 'real_oauth_token'  // Must be valid!
  })
})
```

## 🔧 Integration Notes

### Frontend Changes Required
1. **Remove user_id parameters** from all API calls
2. **Store session tokens** instead of user IDs
3. **Use Authorization headers** for all subsequent requests
4. **Handle session expiry** gracefully

### Migration Steps
1. Update authentication flow to use tokens
2. Replace user_id-based API calls with session-based calls
3. Update error handling for session expiry
4. Test with real provider tokens

### Error Handling
```javascript
const makeRequest = async (endpoint, options = {}) => {
  try {
    const response = await fetch(endpoint, {
      ...options,
      headers: {
        'Authorization': `Bearer ${sessionToken}`,
        ...options.headers
      }
    })

    if (response.status === 401) {
      // Session expired
      localStorage.removeItem('commentum_session_token')
      throw new Error('Session expired. Please log in again.')
    }

    return response.json()
  } catch (error) {
    console.error('Request failed:', error.message)
    throw error
  }
}
```

This secure authentication system ensures that user identities cannot be spoofed while providing a seamless experience for legitimate users.
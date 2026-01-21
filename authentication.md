# Authentication & Authorization Guide

This guide covers the secure session-based authentication and authorization system in Commentum, including token verification, session management, role-based access control, and security best practices.

## Table of Contents

- [Overview](#overview)
- [Security Architecture](#security-architecture)
- [Token-Based Authentication](#token-based-authentication)
- [Session Management](#session-management)
- [Identity Verification](#identity-verification)
- [Role-Based Access Control](#role-based-access-control)
- [Permission System](#permission-system)
- [Security Implementation](#security-implementation)
- [Integration Examples](#integration-examples)

## Overview

Commentum uses a **secure session-based authentication system** that prevents identity spoofing by verifying provider tokens and managing server-side sessions.

### Key Concepts

- **Token Verification**: All provider tokens are verified with respective APIs
- **Session Management**: Secure server-side sessions with automatic expiry
- **Zero-Trust Architecture**: No client-provided user data is trusted
- **Multi-Platform Support**: AniList, MyAnimeList, SIMKL integration
- **Role Hierarchy**: User → Moderator → Admin → Super Admin
- **Row Level Security**: Database-level access control

## Security Architecture

### 🚨 Critical Security Fix

The system prevents **identity spoofing attacks** that were possible in previous versions:

```typescript
// ❌ OLD VULNERABLE: Client could send any user_id
const { user_id } = await resolveIdentity(...)
fetch('/functions/v1/comments', {
  body: JSON.stringify({ user_id, action: 'create' }) // DANGEROUS!
})

// ✅ NEW SECURE: Server validates session token
const { session_token } = await authenticateUser(...)
fetch('/functions/v1/comments', {
  headers: { 'Authorization': `Bearer ${session_token}` },
  body: JSON.stringify({ action: 'create' }) // SAFE!
})
```

### Security Benefits

- ✅ **Identity Spoofing Prevention**: Impossible to fake user IDs
- ✅ **Real Token Verification**: All tokens verified with provider APIs
- ✅ **Session Security**: Cryptographically secure session tokens
- ✅ **Automatic Expiry**: 30-day session expiration with cleanup
- ✅ **Zero-Trust**: No client-provided data trusted

## Token-Based Authentication

### Provider Token Verification

Each platform's token is verified with its respective API:

#### AniList Token Verification
```typescript
async function verifyAniListToken(token: string): Promise<UserInfo | null> {
  const response = await fetch('https://graphql.anilist.co', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      query: `
        query {
          Viewer {
            id
            name
            avatar {
              large
            }
          }
        }
      `
    })
  })

  if (!response.ok) return null
  
  const data = await response.json()
  if (data.errors) return null

  const viewer = data.data.Viewer
  return {
    id: viewer.id.toString(),
    username: viewer.name,
    avatar_url: viewer.avatar?.large
  }
}
```

#### MyAnimeList Token Verification
```typescript
async function verifyMyAnimeListToken(token: string): Promise<UserInfo | null> {
  const response = await fetch('https://api.myanimelist.net/v2/users/@me', {
    method: 'GET',
    headers: {
      'Authorization': `Bearer ${token}`,
    }
  })

  if (!response.ok) return null
  
  const data = await response.json()
  return {
    id: data.id.toString(),
    username: data.name,
    avatar_url: data.picture
  }
}
```

#### SIMKL Token Verification
```typescript
async function verifySimklToken(token: string): Promise<UserInfo | null> {
  const response = await fetch('https://api.simkl.com/users/settings', {
    method: 'GET',
    headers: {
      'simkl-api-key': token,
    }
  })

  if (!response.ok) return null
  
  const userData = await response.json()
  
  return {
    id: userData.user.id.toString(),
    username: userData.user.name,
    avatar_url: userData.user.avatar
  }
}
```

## Session Management

### Session Creation

```typescript
// Generate cryptographically secure session token
function generateSessionToken(): string {
  const array = new Uint8Array(32)
  crypto.getRandomValues(array)
  return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
}

// Create session after successful token verification
async function createSession(userId: number, clientType: string): Promise<SessionInfo> {
  const sessionToken = generateSessionToken()
  const expiresAt = new Date()
  expiresAt.setDate(expiresAt.getDate() + 30) // 30 days

  // Store session in database
  await supabase
    .from('user_sessions')
    .insert({
      user_id: userId,
      session_token: sessionToken,
      client_type: clientType,
      expires_at: expiresAt.toISOString()
    })

  return {
    session_token: sessionToken,
    expires_at: expiresAt.toISOString()
  }
}
```

### Session Verification

```typescript
// Verify session token on every API request
async function verifySessionToken(sessionToken: string): Promise<SessionContext | null> {
  // Clean up expired sessions first
  await supabase
    .from('user_sessions')
    .delete()
    .lt('expires_at', new Date().toISOString())

  // Find valid session
  const { data: session } = await supabase
    .from('user_sessions')
    .select(`
      id,
      user_id,
      users (
        id,
        username,
        role,
        banned,
        shadow_banned,
        muted_until
      )
    `)
    .eq('session_token', sessionToken)
    .gt('expires_at', new Date().toISOString())
    .single()

  if (!session) return null

  // Check if user is banned or muted
  const user = session.users
  if (user.banned || (user.muted_until && user.muted_until > new Date().toISOString())) {
    return null
  }

  // Update last used time
  await supabase
    .from('user_sessions')
    .update({ last_used_at: new Date().toISOString() })
    .eq('id', session.id)

  return {
    user: {
      id: user.id,
      username: user.username,
      role: user.role,
      banned: user.banned,
      shadow_banned: user.shadow_banned,
      muted_until: user.muted_until
    },
    session_id: session.id
  }
}
```

## Identity Verification

### Authentication Flow

```typescript
// 1. Provider Authentication
const authenticateUser = async (clientType: string, token: string) => {
  // Verify token with provider
  const userInfo = await verifyToken(clientType, token)
  if (!userInfo) {
    throw new Error('Invalid token or authentication failed')
  }

  // 2. Identity Resolution
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

  // 3. Store Session Token
  localStorage.setItem('commentum_session_token', result.session_token)
  
  return result.user
}
```

### Session-Based API Calls

```typescript
// All subsequent API calls use session tokens
const makeAuthenticatedRequest = async (endpoint: string, options: RequestInit = {}) => {
  const sessionToken = localStorage.getItem('commentum_session_token')
  
  if (!sessionToken) {
    throw new Error('Not authenticated')
  }

  const response = await fetch(endpoint, {
    ...options,
    headers: {
      'Authorization': `Bearer ${sessionToken}`,
      'Content-Type': 'application/json',
      ...options.headers
    }
  })

  // Handle session expiry
  if (response.status === 401) {
    localStorage.removeItem('commentum_session_token')
    throw new Error('Session expired')
  }

  return response.json()
}
```

## Role-Based Access Control

### Role Hierarchy

```typescript
enum UserRole {
  USER = 'user',
  MODERATOR = 'moderator',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin'
}

interface RolePermissions {
  level: number
  permissions: Permission[]
}

const ROLE_HIERARCHY: Record<UserRole, RolePermissions> = {
  [UserRole.USER]: {
    level: 0,
    permissions: [
      'comment:create',
      'comment:edit:own',
      'comment:delete:own',
      'vote:create',
      'report:create'
    ]
  },
  [UserRole.MODERATOR]: {
    level: 1,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.USER].permissions,
      'comment:delete:any',
      'comment:pin',
      'comment:lock',
      'comment:tag',
      'report:resolve',
      'user:warn',
      'user:mute'
    ]
  },
  [UserRole.ADMIN]: {
    level: 2,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.MODERATOR].permissions,
      'user:ban',
      'user:shadow_ban',
      'user:promote:moderator',
      'config:view:sensitive'
    ]
  },
  [UserRole.SUPER_ADMIN]: {
    level: 3,
    permissions: [
      ...ROLE_HIERARCHY[UserRole.ADMIN].permissions,
      'user:promote:admin',
      'config:modify:any',
      'system:admin'
    ]
  }
}
```

### Permission Checking

```typescript
// React Hook for permissions
export const usePermissions = (user?: UserIdentity | null) => {
  const permissionChecker = useMemo(() => {
    if (!user) return null
    return new PermissionChecker(user.role)
  }, [user])

  const hasPermission = useCallback((permission: Permission) => {
    return permissionChecker?.hasPermission(permission) || false
  }, [permissionChecker])

  const canModerate = useCallback(() => {
    return hasPermission('comment:delete:any')
  }, [hasPermission])

  const canAdmin = useCallback(() => {
    return hasPermission('user:ban')
  }, [hasPermission])

  return {
    permissionChecker,
    hasPermission,
    canModerate,
    canAdmin
  }
}
```

## Permission System

### Database-Level Permissions

Row Level Security (RLS) policies enforce permissions at the database level:

```sql
-- Example: Comments table RLS policy
CREATE POLICY "Users can edit own comments" ON comments
    FOR UPDATE USING (
        user_id = current_setting('app.current_user_id', true)::INTEGER AND
        NOT locked AND
        NOT deleted AND
        user_id NOT IN (
            SELECT id FROM users WHERE banned = TRUE OR muted_until > NOW()
        )
    );

-- Moderator policy
CREATE POLICY "Moderators can moderate comments" ON comments
    FOR ALL USING (
        current_setting('app.current_user_id', true)::INTEGER IN (
            SELECT id FROM users WHERE role IN ('moderator', 'admin', 'super_admin')
        )
    );
```

### Session-Based Context Setting

```typescript
// Edge functions automatically set user context from session
export const setUserContext = async (supabase: any, userId: number, userRole: string) => {
  await supabase.rpc('set_user_context', { 
    user_id: userId.toString(),
    user_role: userRole
  })
}
```

## Security Implementation

### Authentication Context

```typescript
// contexts/AuthContext.tsx
interface AuthState {
  user: UserIdentity | null
  sessionToken: string | null
  loading: boolean
  error: string | null
  isAuthenticated: boolean
}

export const AuthProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [state, setState] = useState<AuthState>({
    user: null,
    sessionToken: null,
    loading: false,
    error: null,
    isAuthenticated: false
  })

  const login = async (clientType: string, token: string) => {
    setState(prev => ({ ...prev, loading: true, error: null }))

    try {
      const user = await authenticateUser(clientType, token)
      
      setState({
        user,
        sessionToken: localStorage.getItem('commentum_session_token'),
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
  }

  const logout = () => {
    localStorage.removeItem('commentum_session_token')
    setState({
      user: null,
      sessionToken: null,
      loading: false,
      error: null,
      isAuthenticated: false
    })
  }

  // Initialize session from storage
  useEffect(() => {
    const sessionToken = localStorage.getItem('commentum_session_token')
    if (sessionToken) {
      // Validate session on app start
      validateSession(sessionToken)
        .then(user => {
          if (user) {
            setState({
              user,
              sessionToken,
              loading: false,
              error: null,
              isAuthenticated: true
            })
          } else {
            logout()
          }
        })
        .catch(() => logout())
    }
  }, [])

  return (
    <AuthContext.Provider value={{
      ...state,
      login,
      logout
    }}>
      {children}
    </AuthContext.Provider>
  )
}
```

### API Client with Session Management

```typescript
// services/apiClient.ts
export class CommentumAPIClient {
  private baseURL: string
  private sessionToken: string | null = null

  constructor(baseURL: string) {
    this.baseURL = baseURL
  }

  setSessionToken(token: string) {
    this.sessionToken = token
  }

  async request(endpoint: string, options: RequestInit = {}): Promise<any> {
    const url = `${this.baseURL}${endpoint}`
    
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    }

    // Add session token if available
    if (this.sessionToken) {
      headers['Authorization'] = `Bearer ${this.sessionToken}`
    }

    const response = await fetch(url, {
      ...options,
      headers
    })

    // Handle session expiry
    if (response.status === 401) {
      this.setSessionToken('')
      throw new Error('Session expired')
    }

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    return response.json()
  }

  // API methods
  async createComment(data: CreateCommentRequest) {
    return this.request('/functions/v1/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...data
      })
    })
  }

  async voteOnComment(commentId: number, voteType: 'upvote' | 'downvote') {
    return this.request('/functions/v1/voting', {
      method: 'POST',
      body: JSON.stringify({
        action: voteType,
        comment_id: commentId
      })
    })
  }
}
```

## Integration Examples

### React Integration

```typescript
// hooks/useCommentum.ts
export const useCommentum = () => {
  const { user, sessionToken, login, logout } = useAuth()
  const [apiClient] = useState(() => new CommentumAPIClient(API_BASE_URL))

  // Update API client when session changes
  useEffect(() => {
    if (sessionToken) {
      apiClient.setSessionToken(sessionToken)
    }
  }, [sessionToken, apiClient])

  const createComment = useCallback(async (
    mediaId?: number,
    mediaInfo?: MediaInfo,
    content: string,
    parentId?: number
  ) => {
    if (!user) throw new Error('Not authenticated')

    return apiClient.createComment({
      media_id: mediaId,
      media_info: mediaInfo,
      parent_id: parentId,
      content
    })
  }, [user, apiClient])

  const voteOnComment = useCallback(async (
    commentId: number,
    voteType: 'upvote' | 'downvote'
  ) => {
    if (!user) throw new Error('Not authenticated')

    return apiClient.voteOnComment(commentId, voteType)
  }, [user, apiClient])

  return {
    user,
    login,
    logout,
    createComment,
    voteOnComment
  }
}
```

### Vanilla JavaScript Integration

```javascript
// services/commentum.js
class CommentumService {
  constructor(baseURL) {
    this.baseURL = baseURL
    this.sessionToken = localStorage.getItem('commentum_session_token')
  }

  async authenticate(clientType, token) {
    const response = await fetch(`${this.baseURL}/functions/v1/identity-resolve`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${ANON_KEY}`,
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
    localStorage.setItem('commentum_session_token', result.session_token)
    
    return result.user
  }

  async makeRequest(endpoint, options = {}) {
    if (!this.sessionToken) {
      throw new Error('Not authenticated')
    }

    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers: {
        'Authorization': `Bearer ${this.sessionToken}`,
        'Content-Type': 'application/json',
        ...options.headers
      }
    })

    if (response.status === 401) {
      this.sessionToken = null
      localStorage.removeItem('commentum_session_token')
      throw new Error('Session expired')
    }

    return response.json()
  }

  async createComment(data) {
    return this.makeRequest('/functions/v1/comments', {
      method: 'POST',
      body: JSON.stringify({
        action: 'create',
        ...data
      })
    })
  }

  logout() {
    this.sessionToken = null
    localStorage.removeItem('commentum_session_token')
  }
}

// Usage
const commentum = new CommentumService('https://your-project.supabase.co')

// Authenticate with AniList
try {
  const user = await commentum.authenticate('anilist', 'your_anilist_token')
  console.log('Authenticated as:', user.username)
  
  // Create comment
  const comment = await commentum.createComment({
    media_info: {
      external_id: '12345',
      media_type: 'anime',
      title: 'Attack on Titan'
    },
    content: 'Great anime!'
  })
  
  console.log('Comment created:', comment)
} catch (error) {
  console.error('Error:', error.message)
}
```

### Vue.js Integration

```typescript
// composables/useCommentum.ts
export const useCommentum = () => {
  const { session } = useAuth()
  const apiClient = new CommentumAPIClient(API_BASE_URL)

  watch(session, (newSession) => {
    if (newSession?.token) {
      apiClient.setSessionToken(newSession.token)
    }
  }, { immediate: true })

  const createComment = async (data: CreateCommentRequest) => {
    if (!session.value) {
      throw new Error('Not authenticated')
    }

    return apiClient.createComment(data)
  }

  const voteOnComment = async (commentId: number, voteType: string) => {
    if (!session.value) {
      throw new Error('Not authenticated')
    }

    return apiClient.voteOnComment(commentId, voteType)
  }

  return {
    createComment,
    voteOnComment
  }
}
```

## Security Best Practices

### Frontend Security

```typescript
// 1. Never store tokens in URL parameters
// ❌ BAD
window.location.hash = `#token=${sessionToken}`

// ✅ GOOD
localStorage.setItem('commentum_session_token', sessionToken)

// 2. Use HTTPS for all API calls
// 3. Validate session on app start
// 4. Clear session on logout
// 5. Handle session expiry gracefully
```

### Backend Security

```typescript
// 1. Always verify session tokens
const authResult = await authenticateRequest(req)
if (authResult.response) {
  return authResult.response
}

// 2. Use RLS policies
await setUserContext(supabase, user.id, user.role)

// 3. Log all actions
await supabase
  .from('moderation_actions')
  .insert({
    moderator_id: user.id,
    action_type: 'create_comment',
    action_details: { media_id: finalMediaId }
  })
```

### Session Management

```typescript
// 1. Generate cryptographically secure tokens
function generateSessionToken(): string {
  const array = new Uint8Array(32)
  crypto.getRandomValues(array)
  return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('')
}

// 2. Set reasonable expiry (30 days)
const expiresAt = new Date()
expiresAt.setDate(expiresAt.getDate() + 30)

// 3. Clean up expired sessions
await supabase
  .from('user_sessions')
  .delete()
  .lt('expires_at', new Date().toISOString())
```

This secure authentication system ensures that user identities cannot be spoofed while providing a seamless experience for legitimate users.